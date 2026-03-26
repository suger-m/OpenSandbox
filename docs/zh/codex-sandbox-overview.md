# Codex Sandbox 总览

> 这篇文章先讲“沙箱到底是在解决什么问题”，再讲 Codex 怎样把同一份策略落到 macOS、Linux 和 Windows。
> 如果你对进程、系统调用、权限边界、ACL、token、namespace、seccomp 这些词还不熟，建议先读 [Codex Sandbox 技术基础入门](./codex-sandbox-foundations.md)。
> 如果你还缺少更宽泛的运行时和容器基础，再回头补 [基础预备：从零看懂 Docker、系统知识与 Kubernetes](./runtime-primer-for-reading-source.md)。

## 先建立一张通用心智图

先不要把 sandbox 理解成某个操作系统里的单一开关。更准确的理解是：

- 沙箱先定义边界，再运行进程。
- 边界通常同时覆盖文件系统、网络、进程权限、可见命名空间，以及少数平台上的桌面或会话资源。
- “隔离”从来不是绝对黑白，而是“默认拒绝 + 明确放行哪些路径、哪些端口、哪些身份、哪些继承关系”。

所以跨平台看 sandbox，最重要的不是先背机制名字，而是先问 4 个问题：

1. 这个进程默认能看到什么？
2. 它被允许修改什么？
3. 它能向外连到哪里？
4. 它是不是拿着比宿主更小的一组权限在运行？

只要这 4 个问题不丢，你看 macOS 的 Seatbelt、Linux 的 bubblewrap/seccomp、Windows 的 restricted token/ACL/firewall 时，就不会被平台细节带偏。

## 沙箱通常分成两层

从设计上看，绝大多数沙箱都可以拆成两层：

### 第一层：策略层

策略层回答“我想要什么边界”。它描述的是意图，不负责直接调用内核或系统 API。

比如：

- 工作区可以写，但 `.git` 和 `.codex` 之类的元数据目录不能乱写。
- 绝大多数文件只读。
- 网络默认关闭，或者只能通过代理走本地回环。
- 子进程不应该借机重新拿到更高权限。

### 第二层：强制层

强制层回答“这个平台怎样真的拦住它”。同样的策略，到了不同系统上，落地手段会完全不同：

- macOS 更像把规则编译成一份 Seatbelt policy。
- Linux 更像先搭一个受限运行视图，再补系统调用和网络收口。
- Windows 更像把身份、文件 ACL、网络出口、桌面会话分层收紧。

真正难的地方，不是“有没有 policy”，而是“能不能把同一份 policy 翻译成每个平台都成立的限制”。

## Codex 的做法：共享策略，平台执行

Codex 的整体思路可以压缩成一句话：

> 先用共享 policy 描述权限意图，再把它交给平台专属执行层去兑现。

这件事在阅读源码时可以理解成三段：

1. 共享策略层
   `SandboxPolicy`、`FileSystemSandboxPolicy`、`NetworkSandboxPolicy` 负责表达“允许什么”和“禁止什么”。
2. 统一启动层
   `spawn` 负责把环境变量、stdio、进程组、网络禁用标记这些启动细节摆正。
3. 平台强制层
   macOS、Linux、Windows 各自把共享策略翻译成操作系统真正能执行的限制。

这也是为什么你会在 Codex 里同时看到“总策略”和“拆分后的文件系统/网络策略”。前者更像上层接口，后者更适合某些平台直接消费。

## 先把 Codex 的几个关键概念对齐

### `SandboxPolicy` 不是“沙箱本身”

它更像一份权限说明书。你可以把它理解成“我要 full access、read only、workspace write，还是我已经身处外部沙箱里”。

关键点不是枚举名字，而是它表达了这些常见语义：

- 有没有全盘读权限
- 有没有全盘写权限
- 有没有全网访问权限
- 工作区写入是不是还带着排除目录
- `/tmp` 或 `TMPDIR` 这种惯常写入点要不要跟着开放

换句话说，Codex 不是把“工作区可写”当成一句模糊口号，而是把它拆成了可核查的规则。

### `spawn` 不是沙箱机制，但它很关键

很多新读者会把 `spawn` 误当成“跟 sandbox 无关的普通启动代码”。其实它负责把策略带入一个稳定、可控的子进程环境：

- 网络受限时，会显式写入 `CODEX_SANDBOX_NETWORK_DISABLED=1`
- shell tool 场景会重定向 stdio，避免命令在后台偷偷等交互输入
- Unix 上会尽量处理进程组和父进程死亡后的继承行为

它不负责“拦截文件访问”，但它负责确保真正进入平台沙箱之前，进程出生方式已经符合预期。

### 平台层不是“同一招换个名字”

Codex 到平台层以后，不是简单地把一个布尔值传下去，而是做真正的机制翻译：

- macOS：把文件和网络策略翻成 Seatbelt 可执行的文本规则与参数。
- Linux：把文件系统策略翻成 bubblewrap 视图，把网络和危险 syscall 进一步收口到 seccomp。
- Windows：先决定 sandbox level，再通过 restricted token、ACL、firewall、private desktop 等组件把边界补齐。

## 三个平台各自擅长的约束方式

下面这张表适合先抓直觉，不适合当源码导航表：

| 平台 | 直觉上的主要手段 | 你可以怎样理解它 |
| --- | --- | --- |
| macOS | Seatbelt policy | “把允许/禁止规则写成一份系统可执行的安全脚本” |
| Linux | mount namespace + bubblewrap + seccomp | “先搭受限世界，再堵危险出口” |
| Windows | restricted token + ACL + firewall + desktop | “把身份、路径权限、网络和交互环境拆开分别收紧” |

这三种路径没有谁“更像真正的沙箱”，它们只是各自在操作系统文化里最自然的落地方式。

## 读 Codex 时最容易误解的几件事

- “sandbox 就是一个统一开关。”
  不是。真正起作用的是多层限制叠加。

- “read only 就一定完全不能写任何地方。”
  不是。很多系统都会保留受控写入点，比如工作区、临时目录，或者显式白名单目录。

- “网络限制就是有网或没网。”
  不是。真实实现里通常还会区分 loopback、代理、Unix socket、本地绑定等更细颗粒度。

- “平台层只是实现细节，知道 shared policy 就够了。”
  也不是。共享策略告诉你意图，平台机制决定这个意图最终靠什么被执行和被绕不过去。

## 推荐阅读顺序

如果你想把这几篇文档串起来读，推荐按这个顺序：

1. [基础预备：从零看懂 Docker、系统知识与 Kubernetes](./runtime-primer-for-reading-source.md)
2. 本文：先建立跨平台沙箱心智模型
3. [Codex Sandbox 机制横向对照](./codex-sandbox-mechanisms.md)
4. [Codex 在 macOS 上如何使用 Seatbelt](./codex-sandbox-macos.md)
5. [Codex 的 Linux sandbox 管线](./codex-sandbox-linux.md)
6. [Codex 原生 Windows 沙箱入门](./codex-sandbox-windows.md)

这样读的好处是：先抓抽象，再看机制，再下钻到单平台，不容易一开始就陷进源码树。

## 用源码做最后一层佐证

如果你想确认上面的说法，这几处源码最值得当“证据链”而不是“阅读入口”：

- 共享策略定义：[`protocol.rs` 里的 `SandboxPolicy`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L857)
- 拆分策略定义：[`permissions.rs` 里的 `NetworkSandboxPolicy` 等类型](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/permissions.rs#L25-L177)
- 统一启动层：[`core/src/spawn.rs` 里的 `spawn_child_async`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L50-L124)
- macOS 落地：[`sandboxing/src/seatbelt.rs` 里的 `create_seatbelt_command_args_for_policies`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L373-L485)
- Linux 落地：[`linux_run_main.rs` 里的 `run_main` 与 proc fallback 路径](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs#L101-L206)
- Windows 落地：[`windows_sandbox.rs` 里的 level 解析与 setup 持久化](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L25-L46) 和 [`run_windows_sandbox_setup_and_persist`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs#L310-L444)

## 一句话收尾

理解 Codex sandbox 的最好方式，不是先问“源码从哪进”，而是先问：

> 它怎样把一份跨平台权限意图，翻译成每个操作系统都真的拦得住的边界。

只要这句话抓住了，后面的平台文档和源码细节都会更好读。
