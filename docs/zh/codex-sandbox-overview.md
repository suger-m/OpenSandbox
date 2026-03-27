# Codex Sandbox 总览

> 说明：本文面向能读一些后端代码、但还没有建立 OS sandbox 心智模型的读者。文中的 Codex 源码链接都固定到 `openai/codex` 的 commit `47a9e2e084e21542821ab65aae91f2bd6bf17c07`，方便你边读边对照。

先给你一张“源码地图”：

| 你要理解的东西 | 最该先看哪份源码 |
| --- | --- |
| Codex 用什么语言描述“允许/禁止什么” | [`codex-rs/protocol/src/protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L79-L81) 与 [`SandboxPolicy`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L840) |
| Codex 怎么把 policy 变成真正的子进程启动参数 | [`codex-rs/core/src/spawn.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L19-L124) |
| macOS 上怎么落地 | [`codex-rs/sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L305-L485) |
| Linux 上怎么落地 | [`codex-rs/linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L101-L706) |
| Windows 上怎么落地 | [`codex-rs/core/src/windows_sandbox.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L25-L46) 与 [`run_windows_sandbox_setup_and_persist`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L324-L444) |

## 这解决什么问题

Codex 不是只负责“执行一个命令”，而是要在不同操作系统上，把同一份 sandbox 意图变成真实的系统限制。对新人来说，最容易误会的一点是：sandbox 不是一个单独的开关，而是一组分层的约束，包含文件系统、网络、进程生命周期、权限边界，以及平台特有的实现细节。

你可以把它想成两层问题。第一层是“允许什么”，这属于共享的 policy 语言。第二层是“怎么在 macOS、Linux、Windows 上真正做到”，这属于平台 helper 的工作。前者比较像协议，后者比较像执行器。

对应源码：先看 [`SandboxPolicy`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L840)，再看 [`spawn_child_async`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L50-L124)。

## 最少背景

你不需要先成为操作系统安全专家，只要先记住 4 个词：

1. `policy`：一份“允许/禁止什么”的描述，不是执行本身。
2. `sandbox`：把进程放进受限环境里，让它看见的文件、网络、权限都变少。
3. `helper`：平台专用的落地代码。Codex 会把共享 policy 交给它，再由它调用系统能力。
4. `legacy` 与 `direct enforcement`：有些平台会先接受旧式的综合 policy，再拆成文件系统和网络两份；有些时候必须直接用拆分后的 policy 才能正确执行。

如果你只想先建立直觉，可以先把 `SandboxPolicy` 理解成“统一的意图层”，把 macOS 的 Seatbelt、Linux 的 bubblewrap / seccomp / Landlock、Windows 的 sandbox setup 理解成“不同平台的执行层”。

对应源码：[`protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L840) 定义 policy 语言，[`linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L275-L340) 展示了“旧式 policy / 拆分 policy”如何一起被接受。

## Codex 的整体策略

Codex 的策略可以概括成一句话：先把 sandbox 意图写成跨平台的 policy，再把这个 policy 交给平台专用的 helper 去执行。这样做的好处是，调用方不需要关心底层到底是 Seatbelt、bubblewrap 还是 Windows sandbox，只要知道“我想要什么样的隔离”。

更具体一点，Codex 先在协议层定义 `SandboxPolicy`、`FileSystemSandboxPolicy` 和 `NetworkSandboxPolicy`；随后在启动子进程时，由统一的 `spawn` 层补齐环境变量、stdio、进程组和生命周期控制；最后交给 macOS、Linux、Windows 的专用代码去真正限制文件、网络和权限。

这也是为什么你在源码里会看到“同一份意图，有时既有 legacy 形式，也有拆分后的 filesystem / network 形式”。它不是重复设计，而是在给不同执行路径提供同一份语义。

对应源码：[`protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L840)、[`spawn.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L50-L124)、[`seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L305-L485)、[`linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L101-L706)、[`windows_sandbox.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L25-L46)。

## 源码调用链

可以先把主链路想成这样：

```text
调用者
  -> 共享 policy（protocol.rs）
  -> 子进程启动器（spawn.rs）
  -> 平台 helper
       -> macOS: Seatbelt
       -> Linux: bubblewrap + seccomp，必要时回退到 Landlock
       -> Windows: sandbox mode 解析 + setup
  -> 真正的用户命令
```

更贴近源码的阅读顺序是：

1. `protocol.rs` 先定义“什么叫 sandbox”。
2. `spawn.rs` 再把这些意图变成一个具体的 `Command`。
3. macOS 路径里，`seatbelt.rs` 把 policy 变成 `sandbox-exec` 的参数。
4. Linux 路径里，`linux_run_main.rs` 先解析 policy，再决定是 bubblewrap 主路径、seccomp 内层路径，还是 legacy Landlock 路径。
5. Windows 路径里，`windows_sandbox.rs` 根据配置和 feature 选择 `Elevated`、`RestrictedToken` 或 `Disabled`，并把 setup 结果持久化。

你如果只想抓住“谁负责什么”，可以这么记：

- `protocol.rs` 负责说清楚规则。
- `spawn.rs` 负责把子进程拉起来。
- `seatbelt.rs`、`linux_run_main.rs`、`windows_sandbox.rs` 负责把规则落到各自平台。

## 关键机制

### `SandboxPolicy` 是共享语言

`SandboxPolicy` 不是执行器，而是一份共享的“权限语言”。它描述的是执行意图，例如完全放开、只读、工作区可写、或者已经在外部 sandbox 中。`WorkspaceWrite` 还带有 `writable_roots`、`exclude_tmpdir_env_var` 和 `exclude_slash_tmp` 这类细节，说明“可写”也可以很精细，不是非黑即白。

对应源码：[`SandboxPolicy`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L840)。

### `spawn_child_async` 负责把进程启动“摆正”

`spawn_child_async` 做的不是 sandbox 本身，而是保证子进程以 Codex 期望的方式启动。它会清空环境变量后再写入需要的变量；如果网络 sandbox 关闭，会设置 `CODEX_SANDBOX_NETWORK_DISABLED=1`；如果是 shell tool 场景，会把 `stdin` 设成空、把 `stdout` / `stderr` 改成管道；还会在 Unix 上尽量处理进程组和父进程死亡信号，让子进程不要在父进程退出后继续乱跑。

你可以把它理解成“子进程出生前的最后整形步骤”。

对应源码：[`spawn.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L50-L124)。

### macOS 用 Seatbelt 把 policy 变成文本规则

macOS 路径里最重要的点是：Codex 不是直接“打开一个系统开关”，而是先把文件读写、网络、Unix socket、平台默认值等拼成一段 Seatbelt policy，然后通过 `/usr/bin/sandbox-exec` 执行。你会在源码里看到 `build_seatbelt_access_policy`、`create_seatbelt_command_args_for_policies` 这类函数，它们本质上是在把“允许哪些 root、排除哪些 subpath、哪些 socket 可以连”翻译成 Seatbelt 能理解的参数。

这说明 macOS 路径的核心不是某个神秘 API，而是“policy 字符串生成器 + sandbox-exec”。

对应源码：[`seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L305-L485)。

### Linux 路径是“两段式”

Linux 这边最容易读晕，因为它不是单一工具，而是两段式流程。外层先决定是否用 bubblewrap 构建文件系统视图；内层再通过 `--apply-seccomp-then-exec` 在已经 sandbox 化的环境里继续收紧。`run_main` 里还会根据 `use_legacy_landlock`、`allow_network_for_proxy` 和 `no_proc` 等参数，决定是否走 legacy Landlock、是否允许 proxy-only 网络、是否在某些环境里放弃挂载 `/proc`。

换句话说，Linux 的重点不是“一个沙箱”，而是“先搭环境，再在里面再收一层”。

对应源码：[`linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L101-L706)。

### Windows 路径先做模式选择，再做 setup

Windows 这边要先把配置和 feature 解析成一个明确的 sandbox level。`WindowsSandboxLevelExt` 会在 `Elevated`、`RestrictedToken` 和 `Disabled` 之间做映射；`run_windows_sandbox_setup_and_persist` 则负责真正执行 setup，并把最终模式写回配置。对新人来说，这里最重要的不是某一个字段，而是“Windows sandbox 的行为是先选模式，再做一次 setup，最后持久化这个选择”。

对应源码：[`windows_sandbox.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L25-L46) 与 [`run_windows_sandbox_setup_and_persist`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L324-L444)。

## 小例子

### 例子 1：一份 policy，多个平台理解

假设你看到这样的意图：

```rust
SandboxPolicy::ReadOnly {
    access: /* 省略 */,
    network_access: false,
}
```

这句代码的重点不是 Rust 语法，而是语义：文件系统尽量只读，网络也不要打开。到了不同平台，这份意图会被翻译成不同的执行方式。macOS 会变成 Seatbelt policy，Linux 会变成 bubblewrap / seccomp / Landlock 的组合，Windows 会先被映射到对应的 sandbox level 再做 setup。

### 例子 2：shell tool 的子进程为什么不一样

`spawn_child_async` 里如果走 `RedirectForShellTool`，它不会把 `stdin` 留给子进程自由读，而是改成空输入，同时把输出接到管道里。这样做的直觉很简单：shell tool 应该尽量可控、可收集输出、不会偷偷等待用户输入。

### 例子 3：Linux 为什么会重试 `--no-proc`

Linux helper 里有一段很实用的保护：它会先用 bubblewrap 做一个小预检，如果发现当前环境不支持挂载 `/proc`，就会退回到不挂载 `/proc` 的路径。你可以把它理解成“先试一次安全的默认值，失败后退回更保守的兼容路径”。

对应源码：[`spawn.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L87-L124)、[`linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L401-L706)。

## 常见误解

- “sandbox 就是一层统一的系统开关。” 不是。Codex 的 sandbox 是共享 policy 加平台 helper 的组合。
- “只读就是绝对不能写。” 也不是。`WorkspaceWrite` 允许有写入根目录，还可以排除某些子路径。
- “Linux 一直靠 Landlock。” 不是。当前主路径更偏向 bubblewrap + seccomp，Landlock 是 legacy fallback。
- “Windows 的 elevated / unelevated 只是 UI 文案。” 不是。它们会影响 setup 逻辑和最终持久化的模式。
- “网络策略就是有网或没网。” 也太粗了。源码里还会考虑 proxy、loopback、本地 socket 和路由桥接。

对应源码：[`protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L840)、[`linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L275-L340)、[`seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L193-L285)、[`windows_sandbox.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L63-L75)。

## 下一步怎么读源码

如果你想继续往下读，我建议按这个顺序走：

1. 先读 [`protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L840)，把 `SandboxPolicy`、`FileSystemSandboxPolicy`、`NetworkSandboxPolicy` 这三个名字先对齐。
2. 再读 [`spawn.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L50-L124)，看 Codex 如何真正启动子进程。
3. 选一个平台深入：macOS 看 [`seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L305-L485)，Linux 看 [`linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L101-L706)，Windows 看 [`windows_sandbox.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L25-L46) 和 [`run_windows_sandbox_setup_and_persist`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L324-L444)。
4. 如果你读到一半开始迷糊，就回到“源码调用链”那一节，重新确认“policy -> spawn -> 平台 helper -> 命令”这条主线。
