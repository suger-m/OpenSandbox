# Codex 的 Linux Sandbox：先懂 Linux，再看实现

这篇文档先讲 Linux sandbox 的基本原理，再讲 Codex 怎么把这些机制组合起来。

如果你对进程、系统调用、权限边界、namespace、seccomp 这些概念还不熟，建议先读 [Codex Sandbox 技术基础入门](./codex-sandbox-foundations.md)。

如果你只先记一句话，可以记这个：

> Codex 在 Linux 上不是靠单一“黑科技”隔离命令，而是把几层机制叠起来：先用 `bubblewrap` 建一个新的运行视图，再用 `no_new_privs` 和 seccomp 收紧危险系统调用；旧的 Landlock 路径只作为 legacy/fallback 方案保留。

本文重点不是“函数从哪里跳到哪里”，而是先回答这些问题：

- namespace 到底是什么？
- 为什么要区分 user namespace、PID namespace、network namespace？
- “挂载视图”是什么，为什么它比“改权限”更像隔离？
- `/proc` 为什么重要，为什么有时还得回退到“不挂 `/proc`”？
- `bubblewrap`、seccomp、`no_new_privs`、Landlock 各自负责什么，又各自负责不了什么？
- Codex 为什么要用 helper re-entry 这种两段式模式？

## 先建立整体模型

从第一性原理看，一个 Linux 命令要“被关进沙箱”，通常至少要处理三类问题：

1. 它能看见什么。
2. 它能调用什么。
3. 它能不能借壳提权。

这三类问题通常分别对应：

- “能看见什么”靠 namespace 和挂载视图。
- “能调用什么”靠 seccomp。
- “能不能借壳提权”靠 `no_new_privs`。

再加上一层文件系统规则，才会形成我们日常说的“sandbox”。

把它想成一个具体例子会更直观。假设你想运行：

```bash
python build.py
```

但你只希望它：

- 能读 `/usr`、`/bin` 这些系统目录；
- 能写工作区 `/repo` 和 `/tmp/build`；
- 不能改 `/repo/.git`；
- 不能直接联网；
- 不能通过奇怪 syscall 绕过限制；
- 不能靠 setuid 程序突然拿到更高权限。

这时，单独使用任意一种机制都不够：

- 只有 mount view，不够，因为进程仍然可能调用危险 syscall。
- 只有 seccomp，不够，因为它照样能看见不该看见的目录。
- 只有 `no_new_privs`，更不够，因为它本身几乎不提供资源隔离。
- 只有 Landlock，也不够，因为它不是拿来搭完整挂载视图和命名空间的。

所以 Codex 选择的是“组合拳”。

## 什么是 namespace

### 它是什么

namespace 可以理解为“给进程一份新的系统视图”。同一台机器上的两个进程，可以看到不同的进程列表、不同的网络栈、不同的挂载结果，哪怕底层仍然跑在同一个内核上。

Linux 常见的 namespace 有很多，这篇只关心 Codex 这里最关键的三个：

- user namespace
- PID namespace
- network namespace

### 它解决什么

它解决的是“看见的世界不一样”。

这很重要，因为很多隔离问题的第一步不是“禁止动作”，而是“先别让它看见宿主环境的真实样子”。

### 它解决不了什么

namespace 不是万能权限系统。它不会自动：

- 阻止文件写入；
- 阻止危险 syscall；
- 阻止提权；
- 自动给你一个正确的最小文件系统。

它只是把“视图”切开。

### Codex 怎么组合

Codex 里真正落地 namespace 的主要工具不是直接手写 `unshare(2)`，而是交给 `bubblewrap` 来做。也就是说，namespace 在 Codex 里通常是“搭环境”这一步的一部分，而不是单独暴露的一层。

第二层源码对应：

- [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs)

## user namespace

### 它是什么

user namespace 会给进程一套新的 UID/GID 身份映射。最常见的效果是：进程在 namespace 里面看起来像“root”，但这个“root”不是宿主机真正的 root。

### 它解决什么

它主要解决两个实际问题：

- 让进程在隔离环境里有足够权限去完成 mount、PID namespace 等准备动作；
- 又不要求它拿到宿主机的全局 root 权限。

一句话总结就是：给沙箱内部“够用的管理能力”，但不把宿主机管理员权限直接交出去。

### 它解决不了什么

它不等于真正安全了。进程即使在 user namespace 里是“root”，也仍然需要别的机制来约束：

- 它能看到哪些路径；
- 它能不能联网；
- 它能不能调用危险 syscall。

### Codex 怎么组合

Codex 在 `bubblewrap` 参数里显式打开 `--unshare-user`。这不是为了“让命令变成真正 root”，而是为了让剩余 namespace 和挂载准备在更可控的上下文里完成。源码里还专门提到，显式请求 user namespace，比依赖 bubblewrap 的自动行为更稳，尤其是在调用方本身是 uid 0 时。

第二层源码对应：

- [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs)

## PID namespace

### 它是什么

PID namespace 给进程一份新的进程编号空间。进入新的 PID namespace 后，进程看到的 `ps`、`/proc`、父子进程关系，都是这份隔离出来的视图。

### 它解决什么

它主要解决两件事：

- 不让沙箱内的进程直接看到宿主机完整的进程树；
- 让沙箱内的 `/proc` 和“当前运行的这一小撮进程”对齐。

这对调试、清理子进程、管理进程生命周期都很关键。

### 它解决不了什么

PID namespace 不会阻止：

- 文件读写；
- 网络访问；
- syscall 调用；
- 通过别的共享资源影响宿主机。

### Codex 怎么组合

Codex 在 bubblewrap 参数里也会显式打开 `--unshare-pid`。这意味着最终命令不是在宿主机 PID 视图里裸跑，而是在新的进程空间里运行。后面为什么 `/proc` 很重要，也和这个选择直接相关。

第二层源码对应：

- [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs)

## network namespace

### 它是什么

network namespace 会给进程新的网络栈视图，包括接口、路由、IP 地址和可见连接。

### 它解决什么

它解决的是“这条命令到底运行在谁的网络环境里”。

例如：

- 如果不切 network namespace，命令看到的就是宿主机网络。
- 如果切到隔离 netns，它看到的是新的、空的、受控的网络环境。

### 它解决不了什么

它也不是完整网络防火墙。单靠 network namespace，不能自动表达：

- 哪些 socket family 允许；
- 哪些 syscall 应该直接返回 `EPERM`；
- 代理场景下到底允许什么流量形态。

### Codex 怎么组合

Codex 用 `bubblewrap` 决定是否 `--unshare-net`。它把网络大致分成三种运行形态：

- `FullAccess`：保留宿主网络；
- `Isolated`：进入隔离 netns；
- `ProxyOnly`：也进入隔离 netns，但后续只允许经由受控代理桥接出去。

这里要注意一个关键点：netns 只是“把网络世界换掉”，不是“把网络 syscall 全都禁掉”。后者还需要 seccomp。

第二层源码对应：

- [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs)
- [`linux-sandbox/src/landlock.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs)

## 什么是“挂载视图”（mount view）

### 它是什么

这里的“挂载视图”不是在改宿主机磁盘内容，而是在给进程拼一份“它所看到的文件系统布局”。

最常见的做法是：

- 用 `tmpfs` 从一个近乎空白的根开始；
- 或者把 `/` 只读绑定进去；
- 再把允许读的目录、允许写的目录、重新只读的子目录一层层盖上去。

这和“直接 chmod”完全不是一个思路。`chmod` 是改对象本身的权限；mount view 是给某个进程准备一份特定视图。

### 它解决什么

它特别适合表达这种策略：

- 默认只读；
- 只给少数目录写权限；
- 某个可写根下面再钉死几个只读子目录。

比如：

```text
/repo           可写
/repo/.git      只读
/repo/.codex    只读
/tmp/build      可写
```

这种“先放开父目录，再收紧某些子路径”的效果，用挂载叠层非常自然。

### 它解决不了什么

挂载视图并不负责：

- 阻止进程调用 `connect`、`socket` 之类 syscall；
- 阻止提权；
- 约束所有内核对象访问面。

### Codex 怎么组合

Codex 的 Linux 文件系统沙箱主路径，核心就是用 bubblewrap 构造这份 mount view。实现里能看到很清晰的分层顺序：

1. 先决定从只读 `/` 还是空白 `tmpfs /` 起步。
2. 挂一个最小 `/dev`。
3. 把允许写的根目录重新 bind 回来。
4. 再把这些可写根下面不该写的子路径重新 `ro-bind` 回去。

这就是为什么它比“简单 deny 一个目录”更强：它能表达层叠语义。

第二层源码对应：

- [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs)

## 为什么 `/proc` 很重要

### 它是什么

`/proc` 是内核导出的虚拟文件系统。很多信息不是“普通文件”，而是通过 `/proc` 暴露出来，比如：

- `/proc/self`
- `/proc/self/fd`
- `/proc/<pid>`
- 进程状态、内存映射、命令行参数

### 它解决什么

在一个新的 PID namespace 里，挂一个新的 `/proc` 有两个主要价值：

- 让沙箱内看到的进程视图和新的 PID namespace 对齐；
- 让依赖 `/proc` 的工具继续正常工作。

如果不这样做，很多程序虽然“被隔离了”，但看到的 `/proc` 视图可能还是错位的，行为会很奇怪。

### 它解决不了什么

`/proc` 本身不是安全边界。它更像“让这个隔离环境自洽”的基础设施。

换句话说：

- 它不是 seccomp；
- 它不是文件写保护；
- 它不是提权防护。

### Codex 怎么组合

Codex 默认倾向于在 bubblewrap 里挂新的 `/proc`。但它没有把这件事写死，而是先做一次极短的预检：用同样的 bwrap 参数跑一个几乎什么都不做的 `true`，试着挂 `/proc`。如果 stderr 呈现出典型的 proc mount 失败特征，就自动回退成“不挂 `/proc` 再跑真实命令”。

这背后的考虑很实用：有些容器环境自身就限制了 proc mount。与其因为一项基础设施准备失败让整个命令直接起不来，不如优雅降级。

第二层源码对应：

- [`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs)

## bubblewrap

### 它是什么

`bubblewrap`（`bwrap`）是一个专门用来“拼装隔离运行环境”的小工具。它擅长做的事情是：

- 建 namespace；
- 搭挂载视图；
- 决定是否切网络命名空间；
- 把最终命令放进准备好的环境里执行。

### 它解决什么

它解决的是“把运行环境搭出来”。尤其适合处理：

- 根文件系统只读；
- 某些目录可写；
- 某些子目录重新只读；
- 新的 `/dev`；
- 新的 `/proc`；
- 新的 PID/user/network namespace。

### 它解决不了什么

它不负责精细 syscall 过滤，也不等于完整安全策略。典型地，它不直接替代：

- seccomp；
- `no_new_privs`；
- 进程内的进一步安全收口。

### Codex 怎么组合

Codex 在现代 Linux 路径里把 bubblewrap 放在外层。

但这里有个容易忽略的细节：如果策略已经是“全盘可写 + 全网开放”，Codex 会尽量不套 bwrap，直接执行原命令，避免多一层没有收益的包装。只有在文件系统或网络真的需要隔离时，才会构建 bubblewrap 命令行。

这说明 bubblewrap 在 Codex 里不是“逢命令必上”，而是按策略启用。

第二层源码对应：

- [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs)
- [`linux-sandbox/src/launcher.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/launcher.rs)

## `no_new_privs`

### 它是什么

`no_new_privs` 是 Linux 上的一个进程标志。设置后，进程及其后代不能再通过 setuid、setgid 或类似机制，获得比当前更多的权限。

### 它解决什么

它解决的是“我已经准备进入受限环境了，但后面别再靠某个特权程序突然涨权限”。

它也是 seccomp 常见的前置条件之一。

### 它解决不了什么

它不是文件系统隔离，不是网络隔离，也不是 syscall 过滤。单独打开它，命令依然可以：

- 访问它能看到的文件；
- 调用大多数 syscall；
- 在现有权限范围内做很多事。

所以它更像“防止后续变强”，而不是“直接把你关起来”。

### Codex 怎么组合

Codex 不会一开始就无脑设置 `no_new_privs`。原因很现实：很多 bubblewrap 部署依赖 setuid 来完成 namespace 准备。如果太早打开 `no_new_privs`，反而会妨碍外层 bwrap 完成工作。

因此 Codex 的策略是：

- 需要 seccomp 时，再打开它；
- 走 legacy Landlock 文件系统路径时，也会打开它；
- 外层环境还没搭好之前，尽量不提前开启。

第二层源码对应：

- [`linux-sandbox/src/landlock.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs)

## seccomp

### 它是什么

seccomp 是 Linux 内核提供的系统调用过滤机制。你可以把它理解成“进程调用 syscall 时，内核再看一眼这通调用是否应该被允许”。

### 它解决什么

它特别适合收紧“动作面”。

例如，当你不想让进程直接联网时，除了切 network namespace，还可以直接把这些 syscall 卡掉：

- `connect`
- `bind`
- `listen`
- `accept`
- `socket`

这样做的好处是，即使进程看起来还活在一个 Linux 环境里，它做关键动作时还是会被内核拦住。

### 它解决不了什么

seccomp 的粒度是 syscall，不是路径语义。它并不擅长表达：

- “`/repo` 可写，但 `/repo/.git` 不可写”；
- “能读这些目录，但不能读那些目录”。

所以它不是 mount view 或 Landlock 的替代品。

### Codex 怎么组合

Codex 在当前线程里安装 seccomp 过滤器，让最终被 `exec` 出去的命令继承它，而不是把整个 CLI 进程一并锁死。

网络上，Codex 不是简单的“有网/没网”二元开关，而是至少有两类 seccomp 模式：

- `Restricted`：尽量直接拒绝网络相关 syscall；
- `ProxyRouted`：允许受控代理桥接必需的 IP socket，但尽量封住别的 socket family，避免绕开代理。

这说明 seccomp 在 Codex 里承担的是“最后一道动作面收口”。

第二层源码对应：

- [`linux-sandbox/src/landlock.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs)

## Landlock

### 它是什么

Landlock 是 Linux 的一个 LSM（Linux Security Module）能力，允许进程给自己叠加一层更细的文件系统访问规则。

### 它解决什么

它擅长的是“从当前进程视角，再补一层文件访问限制”。最典型的用法是：

- 默认只读；
- 少数目录允许写；
- 规则一旦加上，当前进程和子进程一起受限。

### 它解决不了什么

Landlock 不是拿来搭 namespace 和完整 mount view 的工具。它不负责：

- 构建新的 `/dev`；
- 构建新的 `/proc`；
- 切 PID namespace；
- 切 network namespace；
- 过滤网络 syscall。

而且在表达能力上，它也不总能完整覆盖现代 mount-view 方案里那种复杂的层叠语义。

### Codex 怎么组合

在 Codex 当前的 Linux 实现里，Landlock 已经不是主路径。文件系统隔离主要交给 bubblewrap 做，Landlock 只保留在 legacy/backup 角色里。

源码里还明确写出一个限制：旧的 Landlock 文件系统后端不支持“受限只读访问”这类更细粒度模式。也就是说，只要策略需要更复杂的直接运行时约束，legacy Landlock 路径就不适合。

第二层源码对应：

- [`linux-sandbox/src/landlock.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs)

## `bubblewrap` 和 Landlock 各自负责什么

这两个名字经常一起出现，但它们的职责并不一样。

| 机制 | 更像什么 | 最擅长做什么 | 不擅长做什么 |
| --- | --- | --- | --- |
| `bubblewrap` | “搭一间新房子” | namespace、挂载视图、最小 `/dev`、新的 `/proc`、网络命名空间 | 精细 syscall 过滤、进程内追加规则 |
| Landlock | “给已经进屋的人再加规章制度” | 当前进程及后代的文件系统访问约束 | 构造完整 mount view、替代 namespace、处理网络隔离 |

如果换成一句更口语的话：

- `bubblewrap` 决定“这个进程进入的是怎样的世界”；
- Landlock 决定“即使进了这个世界，它还被哪些内核规则继续约束”。

所以在 Codex 里，它们不是完全等价的替代物。现代路径优先选 bubblewrap，是因为 Codex 更需要先把“世界”搭对；Landlock 留在 legacy/fallback，主要是补兼容和旧路径。

## helper re-entry：为什么要分成两段

### 它是什么

helper re-entry 可以理解成：

1. 先启动一个 sandbox helper；
2. 外层 helper 负责准备环境；
3. 再在准备好的环境里“重进一次自己”；
4. 第二段里再上 seccomp，最后 `exec` 到真正用户命令。

### 它解决什么

它解决的是一个顺序问题：

- bubblewrap 往往需要先把 namespace、挂载视图这些外层环境搭好；
- seccomp 和 `no_new_privs` 又应该尽量在靠近最终命令时才加上；
- 如果加得太早，可能反过来影响外层准备动作。

所以更稳的做法不是“一个进程一口气把所有限制立刻加满”，而是分阶段。

### 它解决不了什么

它本身不是安全机制，而是一种组织安全机制的执行模式。真正起隔离效果的，还是前面那些 Linux 机制本身。

### Codex 怎么组合

Codex 的流程大致是：

1. `core` 侧先启动 Linux sandbox helper。
2. helper 解析策略。
3. 如果走现代路径，就构造一个“内层命令”，这个命令其实还是当前这个 helper，只是加上 `--apply-seccomp-then-exec` 等参数。
4. 外层用 bubblewrap 把这个“内层 helper”放进新环境。
5. 内层 helper 启动后，再应用 `no_new_privs + seccomp`，最后 `exec` 成真实命令。

这就是典型的 re-entry 模式。

第二层源码对应：

- [`core/src/spawn.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs)
- [`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs)

## fallback 行为：为什么真实系统里一定要留退路

工程上的 sandbox 和论文里的 sandbox 不一样。真实环境里，你常常会碰到：

- 旧版系统 `bwrap` 太老；
- 容器环境不让你挂 `/proc`；
- 某些策略根本不需要外层包装；
- legacy 路径表达不了新策略。

Codex 在这些地方都做了比较克制的 fallback。

### 1. 不需要时，直接不套 bwrap

如果文件系统是全盘可写，网络也是全开放，Codex 会尽量直接执行原命令，避免无意义的包装开销。

但只要网络需要隔离，哪怕文件系统本身不限制，也仍然会使用 bubblewrap，因为 network namespace 还是要靠它来切。

### 2. `/proc` 挂不上，就回退成不挂 `/proc`

这是最典型的“能增强就增强，增强不了也别把命令整死”的例子。

Codex 先预检，再决定是否把真实命令放在带 `/proc` 的环境里运行。

### 3. 系统 `bwrap` 不够新，就走兼容路径或 vendored 版本

Codex 会优先尝试系统里的 `bwrap`，并探测它是否支持 `--argv0`。如果系统版本太老，或者根本没有系统 `bwrap`，就退回到自带版本。

这类 fallback 不是安全模型变化，而是“启动器兼容性”变化。

### 4. legacy Landlock 不是现代路径失败后的自动兜底

这点很重要。Codex 现代路径是“bubblewrap 先搭环境，再 re-entry 应用 seccomp”。源码里明确说明：现代 bwrap 路径失败时，不会偷偷回退到 legacy Landlock。

原因也很简单：两条路径的表达能力和语义并不完全等价。自动切换，反而可能让用户误以为拿到了同等级隔离。

### 5. legacy 路径只在它能表达策略时才成立

如果策略需要 split filesystem/network policy 的直接运行时约束，而 legacy Landlock 后端表达不了，Codex 会直接拒绝，而不是假装“差不多也行”。

这是一种很好的工程取舍：宁可明确不支持，也不静默降级成更弱语义。

第二层源码对应：

- [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs)
- [`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs)
- [`linux-sandbox/src/launcher.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/launcher.rs)
- [`linux-sandbox/src/landlock.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs)

## 把整条链路串成一个例子

假设有这样一条命令：

```text
cwd = /repo
允许写: /repo, /tmp/build
保持只读: /repo/.git, /repo/.codex
网络: 禁用
```

那么从 Linux 原理出发，它大致会经历这条链路：

1. Codex 先起一个 helper 进程。
2. helper 判断这不是“全开放”策略，所以不能直接裸跑。
3. 外层准备 bubblewrap 参数：
   - 新 user namespace
   - 新 PID namespace
   - 新 network namespace
   - 根文件系统只读或从空白 `tmpfs /` 起步
   - 重新放回 `/repo`、`/tmp/build`
   - 再把 `/repo/.git`、`/repo/.codex` 盖回只读
4. helper 先预检 `/proc` 能不能挂。
5. 外层通过 bubblewrap 启动“内层 helper”。
6. 内层 helper 在当前线程里打开 `no_new_privs`，安装 seccomp。
7. 最后 `exec` 成真实用户命令。

这时你就能看出各层分工：

- namespace 和 mount view 负责“你在什么世界里跑”；
- `no_new_privs` 负责“你别再涨权限”；
- seccomp 负责“你能做哪些危险动作”；
- Landlock 不在这条现代主链上，而是旧路径保留项。

## 如果你要对照源码，按这个顺序读

先按概念读，不要一上来陷进参数细节：

1. [`core/src/spawn.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs)
2. [`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs)
3. [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs)
4. [`linux-sandbox/src/launcher.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/launcher.rs)
5. [`linux-sandbox/src/landlock.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs)

按这个顺序，你会先看懂“为什么这么分层”，再看懂“代码为什么这样组织”。
