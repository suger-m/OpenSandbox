# Codex 沙箱机制横向对照

这篇文档只回答三个问题：每个机制做什么、它不做什么、它在 Codex 源码里出现在哪里。

如果你是第一次读这类代码，可以先记住一句话：`protocol/src/protocol.rs` 定义的是“沙箱想要什么”，而各平台文件定义的是“操作系统实际怎么拦住它”。

## 先看总图

- macOS 主要靠 Seatbelt。
- Linux 默认靠 bubblewrap + `no_new_privs` + seccomp，Landlock 只保留为旧路径。
- Windows 主要靠 restricted token，再配合 ACL、firewall、desktop 和 setup orchestrator。

## 1. 文件系统隔离

| 平台 | 它做什么 | 它不做什么 | 源码位置 |
|---|---|---|---|
| macOS | `seatbelt.rs` 把 `SandboxPolicy` 翻译成 Seatbelt 的 SBPL 规则，按显式可读根、可写根和平台默认根来拼文件访问边界。 | 它不会把进程变成容器，也不会改变宿主文件的所有权；它只是限制这个进程能读写什么。 | [`protocol/src/protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs) / [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs) |
| Linux | `bwrap.rs` 默认把文件系统变成只读视图，再把 writable roots 叠上去；`linux_run_main.rs` 决定何时走 bubblewrap，何时走旧的 Landlock 兼容路径。 | 它不替代宿主的磁盘权限模型，也不负责包级别或端口级别的网络控制。 | [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs) / [`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs) |
| Windows | `acl.rs` 通过 DACL/ACE 给目录加允许或拒绝规则；`setup_orchestrator.rs` 负责把这些规则刷新到正确的用户和工作区。 | 它不是挂载隔离，也不会把整个文件树“藏起来”；更像是对目录权限做精细重写。 | [`windows-sandbox-rs/src/acl.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/acl.rs) / [`windows-sandbox-rs/src/setup_orchestrator.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs) |

## 2. 进程与权限控制

| 平台 | 它做什么 | 它不做什么 | 源码位置 |
|---|---|---|---|
| macOS | Seatbelt 直接限制进程可访问的资源范围，和文件、网络规则一起生效。 | 它不会像 Linux 那样显式创建用户命名空间，也不会像 Windows 那样给进程换一张新的 restricted token。 | [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs) |
| Linux | `linux_run_main.rs` 先让 bubblewrap 建好隔离环境，再在进程内应用 `PR_SET_NO_NEW_PRIVS` 和 seccomp；这样后面的子进程继承的是更小权限。 | 它不是一个“系统级容器管理器”；它只控制这条命令链，没打算改造整个机器的进程模型。 | [`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs) / [`linux-sandbox/src/landlock.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs) |
| Windows | `token.rs` 用 `CreateRestrictedToken` 创建受限 token，再配合默认 DACL 和 capability SID 压低权限。 | 它不是“把管理员变成普通用户”的通用 Windows 登录切换；它只是让这个 sandbox 进程拿到更小的权限集合。 | [`windows-sandbox-rs/src/token.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/token.rs) |

## 3. 网络限制

| 平台 | 它做什么 | 它不做什么 | 源码位置 |
|---|---|---|---|
| macOS | `seatbelt.rs` 会把网络规则编进 Seatbelt policy；在代理或受管网络场景下，它会把可用网络收窄到 localhost、unix socket 和明确允许的端口。 | 它不是通用防火墙，也不负责 packet 级过滤；它更像“按策略放行 socket 访问”。 | [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs) |
| Linux | `linux_run_main.rs` 会在受限网络场景里安装 seccomp，并在需要时切到 proxy-only 或 isolated 网络命名空间；`bwrap.rs` 负责把这些模式塞进 bubblewrap 参数里。 | 它不是 iptables 之类的流量防火墙，也不试图解析请求内容；它主要拦的是系统调用和命名空间可见性。 | [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs) / [`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs) |
| Windows | `firewall.rs` 用 Windows Firewall 写出非回环流量阻断规则，并给代理端口留白名单；`setup_orchestrator.rs` 会根据离线/在线身份和代理端口刷新这些规则。 | 它不限制进程的文件权限，也不替代 restricted token；它只处理网络出口。 | [`windows-sandbox-rs/src/firewall.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/firewall.rs) / [`windows-sandbox-rs/src/setup_orchestrator.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs) |

## 4. 身份边界

这里的“身份边界”是指：系统有没有把 sandbox 进程当成一个“更小、更专门的身份”来对待。

| 平台 | 它做什么 | 它不做什么 | 源码位置 |
|---|---|---|---|
| macOS | Codex 在这条路径里没有单独切换到另一个用户身份；它仍然是在当前 macOS 用户上下文里，用 Seatbelt 限制资源访问。 | 它不创建新的用户登录，也不做类似 Windows logon SID 那种身份重建。 | [`protocol/src/protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs) / [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs) |
| Linux | `--unshare-user` 会让 bubblewrap 进入新的 user namespace，再配合 `--unshare-pid`、`--unshare-net` 把沙箱看见的世界缩小。 | 它不等于“创建了一个真实的 Linux 账号”；更准确地说，是 namespace 层面的身份隔离。 | [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs) / [`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs) |
| Windows | 这是 Windows 里最明确的身份边界：restricted token + capability SID + logon SID，把“谁能做什么”收紧到沙箱需要的最小集合。 | 它不等于整个桌面会话都换了身份；桌面、ACL、firewall 仍然是另外几层。 | [`windows-sandbox-rs/src/token.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/token.rs) / [`windows-sandbox-rs/src/setup_orchestrator.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs) |

## 5. UI / 桌面边界

| 平台 | 它做什么 | 它不做什么 | 源码位置 |
|---|---|---|---|
| macOS | 这条 Codex 沙箱路径里没有单独的桌面隔离层。 | 它不会把 GUI 和终端任务拆成两个桌面世界。 | [`protocol/src/protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs) |
| Linux | 这条路径里也没有独立的 UI 沙箱层，重点还是文件、进程、网络。 | 它不会创建独立桌面，也不会把窗口系统当作一等边界来管理。 | [`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs) |
| Windows | `desktop.rs` 可以创建 private desktop，并且 `process.rs` 会把 `lpDesktop` 显式指过去；`core/src/windows_sandbox.rs` 还把 `sandbox_private_desktop` 作为可配置项。 | 它只隔离桌面会话表层，不会自动替你做文件或网络限制。 | [`windows-sandbox-rs/src/desktop.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/desktop.rs) / [`windows-sandbox-rs/src/process.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/process.rs) / [`core/src/windows_sandbox.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs) |

## 6. 旧路径与回退

| 平台 | 它做什么 | 它不做什么 | 源码位置 |
|---|---|---|---|
| macOS | Seatbelt 仍保留一些兼容性读权限拼接逻辑，让老工作流继续拿到必要的系统读取能力。 | 它不是另一套沙箱引擎，也不是“旧版和新版并行运行”的双实现。 | [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs) |
| Linux | `linux_run_main.rs` 里有 `--use-legacy-landlock` 旧路径；`landlock.rs` 也明确把 Landlock 标成 legacy/backup。bubblewrap 还有 `--proc /proc` 的预检回退，容器不允许时会改走 `--no-proc`。 | 旧 Landlock 不是完整替代品，尤其不支持受限 read-only 这类更细的文件策略。 | [`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs) / [`linux-sandbox/src/landlock.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs) / [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs) |
| Windows | `core/src/windows_sandbox.rs` 还能读老 feature key，并把它们翻译成当前的 `Elevated` / `RestrictedToken` 级别；`setup_orchestrator.rs` 还有 refresh-only 路径，尽量避免不必要的 UAC 弹窗。 | 它不意味着回到“旧版安全模型”；这些更多是迁移和初始化兼容层。 | [`core/src/windows_sandbox.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs) / [`windows-sandbox-rs/src/setup_orchestrator.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs) / [`windows-sandbox-rs/src/lib.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/lib.rs) |

## 读源码顺序

如果你想顺着源码把整条链路看懂，可以按这个顺序读：

1. 先看 [`protocol/src/protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs)，把 `SandboxPolicy`、`ReadOnlyAccess`、`NetworkAccess` 先记住。
2. 再看 macOS 的 [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs)。
3. 再看 Linux 的 [`linux-sandbox/src/bwrap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs)、[`linux-sandbox/src/linux_run_main.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs) 和 [`linux-sandbox/src/landlock.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs)。
4. 最后看 Windows 的 [`core/src/windows_sandbox.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs)，再往下跳到 [`windows-sandbox-rs/src/token.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/token.rs)、[`windows-sandbox-rs/src/acl.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/acl.rs)、[`windows-sandbox-rs/src/firewall.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/firewall.rs)、[`windows-sandbox-rs/src/desktop.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/desktop.rs) 和 [`windows-sandbox-rs/src/setup_orchestrator.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs)。

## 一句提醒

Codex 不是用“一种沙箱”解决所有平台问题，而是先抽象出统一政策，再按平台把它翻译成最合适的 OS 机制。你读这篇文档的目标，不是背机制名字，而是知道每个机制的边界在哪里。
