# Codex Sandbox 机制横向对照

这篇文档不按“先看哪个源码文件”来组织，而是按“沙箱常见机制到底是什么”来组织。
如果你对进程、权限、ACL、token、namespace 这些基础名词还不稳，建议先读 [Codex Sandbox 技术基础入门](./codex-sandbox-foundations.md)。
如果你还没建立整体心智模型，建议先读 [Codex Sandbox 总览](./codex-sandbox-overview.md)。
如果你还缺少运行时基础，建议再补一遍 [基础预备：从零看懂 Docker、系统知识与 Kubernetes](./runtime-primer-for-reading-source.md)。

读完这篇以后，适合继续下钻到平台篇：

- [Codex 在 macOS 上如何使用 Seatbelt](./codex-sandbox-macos.md)
- [Codex 的 Linux sandbox 管线](./codex-sandbox-linux.md)
- [Codex 原生 Windows 沙箱入门](./codex-sandbox-windows.md)

## 先记一个总原则

“机制”不是“策略”。

- 策略回答：我要什么边界。
- 机制回答：操作系统靠什么把边界变成现实。

同样一句“工作区可写、其他只读、网络尽量关掉”，在三个平台上可能分别变成：

- 一份 policy 文本
- 一组 namespace 和 mount 操作
- 一张受限 token 加上 ACL 和 firewall 规则

所以横向比较时，最稳的办法不是比较文件名，而是比较这些机制各自负责什么。

## 1. 文件系统边界

文件系统边界回答的是：这个进程能看见哪些路径，哪些路径能写，哪些路径虽然位于可写根下但仍要例外保护。

### macOS：策略式路径匹配

macOS 这一路最像“把路径规则编译成安全策略”。Codex 会把可读根、可写根、排除子路径、Unix socket 和平台默认根拼成 Seatbelt 能理解的规则文本，再交给 `sandbox-exec`。

这种机制的特点是：

- 优点是规则表达很直接，路径白名单和例外规则都比较清楚。
- 边界是“按策略允许什么路径访问”，不是“重建一个新的文件系统视图”。
- 它不会像容器那样把宿主目录重新挂载成另一个世界。

### Linux：挂载视图重建

Linux 更像“先造一个新的运行时视图”。Codex 通过 bubblewrap 组合 `ro-bind`、`bind`、`tmpfs`、`/dev`、`/proc` 等挂载动作，让进程从一开始就活在一个被改造过的文件系统里。

这种机制的特点是：

- 优点是边界非常直观：进程看到的世界本身就被重排过了。
- 很适合表达“默认只读，再把少数目录补成可写”。
- 例外保护也自然，比如工作区可写，但 `.git`、`.codex` 这类子目录再盖回只读。

### Windows：ACL 权限重写

Windows 没有沿着“重新挂载文件树”的路走，而是更强调“谁对哪个目录拥有什么访问权限”。Codex 会围绕 DACL/ACE 去检查、补写、收紧目录权限，让受限身份只能在该写的地方写。

这种机制的特点是：

- 优点是更贴合 Windows 原生权限模型。
- 它不是把目录“藏起来”，而是让受限身份即使看得到，也拿不到不该有的写权限。
- 重点不在 mount，而在 access control。

### 一句话对照

| 平台 | 文件系统机制的直觉 |
| --- | --- |
| macOS | 写规则 |
| Linux | 搭视图 |
| Windows | 改权限 |

源码佐证：

- [`seatbelt.rs` 的访问策略构造](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L305-L485)
- [`bwrap.rs` 的文件系统参数生成](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs#L41-L385)
- [`acl.rs` 的 DACL 检查与重写路径](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/acl.rs#L54-L621)

## 2. 进程权限边界

这一层回答的是：即使进程启动了，它是不是还可能在运行过程中重新拿到更高权限，或者继承到过宽的宿主能力。

### macOS：主要靠 Seatbelt 约束资源访问

在 Codex 的 macOS 路径里，重点不在“切一个新的用户身份”，而在“让当前进程在 Seatbelt 规则下运行”。它仍然主要依赖资源访问限制，而不是单独重建身份模型。

这意味着：

- 它对“能碰什么资源”限制很强。
- 但它不是 Linux user namespace，也不是 Windows restricted token 那种显式身份收缩路线。

### Linux：namespace + `no_new_privs`

Linux 这里有更明显的“进程边界工程”。bubblewrap 可以把进程放进新的 user/pid/net namespace，而 `PR_SET_NO_NEW_PRIVS` 则负责堵住“之后再借 setuid 或类似路径提权”的可能。

这层机制的教学重点是：

- namespace 解决“你看见的是哪个世界”。
- `no_new_privs` 解决“你之后还能不能把权限再变大”。

它们经常一起出现，但职责并不相同。

### Windows：restricted token

Windows 最典型的做法是给沙箱进程发一张“更小的身份证”。`CreateRestrictedToken` 会把权限集合压缩，再配合 capability SID、logon SID 和默认 DACL，让“这个进程是谁、能做什么”从身份层就先收紧。

这一层的直觉是：

- Linux 更像在改“世界”和“规则”。
- Windows 更像先改“你是谁”。

源码佐证：

- [`spawn.rs` 的启动约束与 stdio/network 标记](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L50-L124)
- [`linux_run_main.rs` 与 `landlock.rs` 中的 inner-stage 和 `no_new_privs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L119-L206)
- [`token.rs` 中的 `CreateRestrictedToken` 路径](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/token.rs#L308-L369)

## 3. 系统调用和网络出口

很多人第一次看 sandbox 时，会把“网络限制”理解成单一的开/关。真实情况通常更细：

- 你是不是还能建 socket
- 你能不能绑定本地端口
- 你是不是只能连 loopback
- 你是不是只能通过代理桥接出网

### macOS：把网络规则写进 Seatbelt

macOS 路径下，网络通常和文件访问一起写进 Seatbelt policy。它擅长表达“允许本地回环”“允许特定 Unix socket”“允许某些代理路径”这类规则。

这意味着它更像：

- 对 socket 访问做策略放行
- 而不是单独构建网络命名空间

### Linux：网络命名空间 + seccomp

Linux 这里是两层组合：

1. bubblewrap 决定网络命名空间是 `FullAccess`、`Isolated` 还是 `ProxyOnly`
2. seccomp 再决定某些网络相关 syscall 能不能继续用

这套组合很值得学，因为它展示了一个通用设计：

- 一层控制“你看见哪个网络”
- 一层控制“你能不能调用危险接口”

两层叠起来，比单独依赖任一层都更稳。

### Windows：firewall 规则补齐出口限制

Windows 这边没有 Linux 那样的 net namespace，于是 Codex 采用的是另一条思路：在受限身份之外，再额外写 Windows Firewall 规则，把非 loopback 出口或不符合代理要求的流量继续收紧。

它的特点是：

- 处理的是网络出口和授权范围
- 不负责文件权限
- 也不替代 restricted token

源码佐证：

- [`seatbelt.rs` 中的 `allow_local_binding`、Unix socket 和网络策略拼接](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L80-L290)
- [`linux_run_main.rs` 的网络模式与 proc fallback 主线](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L141-L206)
- [`landlock.rs` 的网络 seccomp 模式](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs#L91-L184)
- [`firewall.rs` 的 `LocalUserAuthorizedList` 和规则配置路径](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/firewall.rs#L267-L350)

## 4. 身份与会话边界

这层机制回答的是：系统是否把沙箱进程当成一个更小、更专门的主体来对待，以及它有没有单独的会话或桌面边界。

### macOS：以资源限制为主，不强调身份重建

Codex 的 macOS 路径并不突出“新身份”，更突出“当前身份下的资源访问被 Seatbelt 收紧”。所以它的核心不是身份切换。

### Linux：更多是 namespace 身份感，而不是真新用户

Linux 的 user namespace 很容易被误读成“创建了一个新的系统用户”。更准确的说法是：它提供的是 namespace 语义上的隔离感，而不是传统账户体系里的新登录身份。

### Windows：身份边界最显式

Windows 在这方面最“制度化”。受限 token、capability SID、logon SID 让沙箱身份变成系统显式可辨认的一层。再往上，private desktop 还能把 GUI/交互会话和普通桌面继续分开。

这也是为什么 Windows 文档常常会同时提到 token、ACL、desktop。它们不是重复，而是在分别回答：

- 你是谁
- 你能碰哪些对象
- 你在哪个桌面上下文里运行

源码佐证：

- [`windows_sandbox.rs` 中的 private desktop 配置解析](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L78-L86)
- [`desktop.rs` 的 `LaunchDesktop` 路径](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/desktop.rs#L57-L166)
- [`process.rs` 在平台篇中继续承接桌面与受限 token 的执行路径`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/process.rs)

## 5. 兼容层与回退路径

成熟沙箱实现很少是“一条理想路径跑到底”。真实工程里，经常还要处理兼容、迁移和失败回退。

### macOS：平台默认根和 Unix socket 白名单

macOS 这边的兼容性主要体现在策略拼接细节上。比如系统默认只读根、代理所需的本地 socket、是否允许本地绑定，都会影响最终 policy 长什么样。

它更像“规则生成时的兼容补丁”，而不是另一套独立后端。

### Linux：`/proc` fallback 与 legacy Landlock

Linux 是三平台里兼容路径最显眼的一个：

- bubblewrap 尝试挂 `/proc` 失败时，会退回 `--no-proc`
- 仍保留 `--use-legacy-landlock`
- 但 Landlock 明确已经不是当前主路径，而且某些更细的只读策略并不支持

这很适合拿来理解一个工程现实：

> 主路径是主路径，保底路径是保底路径；两者同时存在，不代表它们地位相同。

### Windows：setup 持久化与 refresh-only

Windows 这边的复杂性主要来自“不是每次都重新做一遍完整 setup”。Codex 会解析当前 level、检查 setup 是否完成、必要时重新 setup，并把结果持久化。某些变化只需要 refresh，不一定要每次都走最重的初始化链路。

这说明 Windows 路径里有明显的“长期状态”概念，而不仅仅是一次性启动参数。

源码佐证：

- [`linux_run_main.rs` 的 `run_bwrap_with_proc_fallback`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L401-L706)
- [`landlock.rs` 对 legacy backend 能力边界的说明](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs#L1-L73)
- [`windows_sandbox.rs` 的 setup 解析与持久化主线](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L287-L444)
- [`setup_orchestrator.rs` 中对 firewall drift 和 marker 的处理](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs)

## 6. 把三平台压缩成一张比较表

如果你只想留一张记忆卡，可以看这张：

| 维度 | macOS | Linux | Windows |
| --- | --- | --- | --- |
| 文件系统 | Seatbelt 路径规则 | bubblewrap 挂载视图 | ACL/DACL 权限重写 |
| 进程权限 | 资源访问受 Seatbelt 控制 | namespace + `no_new_privs` | restricted token |
| 网络 | Seatbelt socket/loopback/Unix socket 规则 | net namespace + seccomp | firewall 规则补齐出口限制 |
| 身份模型 | 不强调重建新身份 | namespace 语义上的隔离身份 | SID/token 最显式 |
| 会话/桌面 | 无单独桌面层 | 无单独桌面层 | private desktop 可选 |
| 兼容/回退 | 规则拼接兼容 | `/proc` fallback + legacy Landlock | setup 持久化与 refresh |

## 7. 读平台篇时该带着什么问题去看

到了平台篇，建议每次只带一个问题：

- 看 macOS 篇时，重点问：Seatbelt rule 是怎样从 policy 拼出来的？
- 看 Linux 篇时，重点问：bubblewrap 和 seccomp 分别负责哪一半？
- 看 Windows 篇时，重点问：token、ACL、firewall、desktop 为什么要同时存在？

这样你读单平台源码时，就会是在验证机制，而不是在源码目录里迷路。

## 一句话收尾

沙箱机制的横向比较，真正要比的不是“哪个平台文件更多”，而是：

> 每个平台分别用什么办法，去解决文件、权限、网络、身份和兼容这五类问题。

一旦把这个框架抓稳，源码文件只是证据，不再是入口障碍。
