# Codex 在 macOS 上如何使用 Seatbelt

> 这篇文章面向刚接触 Codex 源码的新手。你不需要先懂完整的 macOS 安全模型，只要先抓住一件事：Codex 不是“随手开一个沙箱开关”，而是先把“允许什么”写成 policy，再把 policy 翻译成 `/usr/bin/sandbox-exec` 能执行的 Seatbelt 规则。源码链接都固定在 `openai/codex` 的 commit `47a9e2e084e21542821ab65aae91f2bd6bf17c07`。

## 先看整体

如果只记一条主线，就记这个：

```text
SandboxPolicy
  -> 文件/网络的拆分 policy
  -> Seatbelt SBPL 字符串
  -> /usr/bin/sandbox-exec -p <profile> -- <command>
```

对应源码可以先从这三处打开：

- [`codex-rs/protocol/src/protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L783-L848) 负责描述“允许什么”。
- [`codex-rs/core/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/seatbelt.rs#L18-L47) 负责把 macOS 路径接起来。
- [`codex-rs/sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L18-L27) 负责生成真正的 `sandbox-exec` 参数和 profile 文本。

## 最少背景

先把三个名词分清：

- `Seatbelt` 是 macOS 的沙箱机制名。
- `sandbox-exec` 是一个启动器。Codex 不是直接“调用 Seatbelt API”，而是让 `sandbox-exec` 读取一段 profile 文本。
- `SBPL` 可以先理解成“Seatbelt 的规则语言”。它长得像文本规则，而不是 JSON 或 Rust 结构体。

Codex 这里很保守：它固定使用 `/usr/bin/sandbox-exec`，而不是去 `PATH` 里找同名程序。源码里的注释说得很直白，这样可以避免被恶意的 PATH 劫持。

## Codex 的策略

Codex 的做法不是在各个平台都重新发明一套“权限概念”，而是先在协议层统一描述，再在平台层落地。

在协议层，`SandboxPolicy` 说明了几种常见意图：完全放开、只读、已经处在外部沙箱里、或者工作区可写。它本身是“意图层”，不是执行层。

在 macOS 路径里，`core/src/seatbelt.rs` 会把这份意图拆成两部分再交给 Seatbelt：

- 文件系统规则：由 `FileSystemSandboxPolicy::from_legacy_sandbox_policy(...)` 生成。
- 网络规则：由 `NetworkSandboxPolicy::from(sandbox_policy)` 生成。

然后它把这两份规则交给 `create_seatbelt_command_args_for_policies(...)`，最终通过 `spawn_child_async(...)` 启动 `/usr/bin/sandbox-exec`。

一个很像真实源码的伪流程是这样：

```rust
let file_policy = FileSystemSandboxPolicy::from_legacy_sandbox_policy(...);
let net_policy = NetworkSandboxPolicy::from(&sandbox_policy);
let args = create_seatbelt_command_args_for_policies(command, &file_policy, net_policy, ...);
spawn_child_async(SpawnChildRequest { program: "/usr/bin/sandbox-exec", args, ... });
```

对应源码：

- [`core/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/seatbelt.rs#L18-L47)
- [`core/src/spawn.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L50-L124)

## 源码调用链

可以按这条路读：

1. [`protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L848) 定义 `SandboxPolicy`。
2. [`permissions.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/permissions.rs#L135-L140) 定义拆分后的 `FileSystemSandboxPolicy`。
3. [`core/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/seatbelt.rs#L27-L47) 把 legacy policy 转成 macOS 需要的参数。
4. [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L373-L499) 拼出完整 profile 和 `-D` 变量。
5. [`spawn.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L50-L124) 真正启动子进程，并补齐环境变量。

你可以把它记成一句话：`protocol` 说规则，`seatbelt.rs` 写规则，`spawn.rs` 启规则。

## Seatbelt profile 怎么拼出来

`sandboxing/src/seatbelt.rs` 里，profile 不是一整块手写死的字符串，而是几块拼起来的：

1. 基础 policy：`seatbelt_base_policy.sbpl`
2. 文件读规则：根据 readable roots 生成
3. 文件写规则：根据 writable roots 生成
4. 网络规则：根据 `NetworkSandboxPolicy` 和代理/loopback 信息生成
5. 平台默认只读规则：必要时再附加

最后它会把这些段落 `join("\n")` 成一个完整 profile，再把目录参数整理成一串 `-DKEY=VALUE` 传给 `sandbox-exec`。

对应源码：

- [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L468-L499)
- [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L305-L352)

一个小例子：

```text
-p "(allow file-read* (subpath (param \"READABLE_ROOT_0\"))) ..."
-DREADABLE_ROOT_0=/Users/me/project
-- /bin/echo hello
```

这里的关键不是语法细节，而是思路：先把“路径”抽成参数，再在规则里引用参数。这样同一套模板就能适配不同机器和不同工作目录。

## 文件规则怎么表达

文件规则的核心是“root + excluded subpaths”。

在协议层，`SandboxPolicy` / `FileSystemSandboxPolicy` 会先回答两个问题：

- 这个策略是不是“全盘可读 / 全盘可写”？
- 如果不是，哪些根目录可读、哪些根目录可写？

在 `sandboxing/src/seatbelt.rs` 里，`build_seatbelt_access_policy(...)` 会把这些根目录变成 Seatbelt 的 `subpath` 规则；如果某些子目录要保留只读，就给它们加上 `require-not (literal ...)` 和 `require-not (subpath ...)`。

这就是为什么 `.git`、`.codex` 这类目录经常会被单独保护：即使上层工作区是可写的，这些“元数据目录”也会被默认卡住。

一个很粗略的理解可以是：

```text
允许 /Users/me/project
但排除 /Users/me/project/.git
也排除 /Users/me/project/.codex
```

对应源码：

- [`protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L928-L1005)
- [`permissions.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/permissions.rs#L258-L341)
- [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L305-L352)

## 网络规则怎么表达

macOS 这里最容易误解的地方，是“网络”并不是一个简单的开/关。Codex 会先看代理信息，再决定是不是只放行 loopback、特定端口，或者 Unix socket。

### loopback 是什么

loopback 就是本机回环地址，典型值是 `localhost`、`127.0.0.1`、`::1`。它只表示“连回这台机器自己”，不是“放行外网”。

如果代理环境变量里出现了本机地址，比如 `http://127.0.0.1:8080`，Codex 会把端口 `8080` 提取出来，然后只允许访问 `localhost:8080` 这一类目标。

### `allow_local_binding` 是什么

`allow_local_binding = true` 的意思是：进程可以在本机地址上监听和收发，比如给本机代理、调试服务、或本地转发器使用。源码里对应的是：

- `network-bind (local ip "localhost:*")`
- `network-inbound (local ip "localhost:*")`
- `network-outbound (remote ip "localhost:*")`

换句话说，它是“允许本机内循环”，不是“允许连公网”。

### Unix socket 是什么

Unix socket 可以把它先理解成“同一台机器上的本地 IPC”，常见于数据库、代理、守护进程。它不是 TCP/UDP 端口，而是一个文件路径。

Codex 这里有两种模式：

- `dangerously_allow_all_unix_sockets()` 为真时，直接允许全部 Unix socket。
- 否则只允许 `allow_unix_sockets()` 列表里的路径，并且会先把路径规范化、去重，再写入规则。

规则长得大致像这样：

```text
(allow system-socket (socket-domain AF_UNIX))
(allow network-outbound (remote unix-socket (subpath (param "UNIX_SOCKET_PATH_0"))))
```

对应源码：

- [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L103-L133)
- [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L193-L227)
- [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L242-L290)

## 这几个常见误解

- “`sandbox-exec` 只是个命令名，放在 `PATH` 里找就行。” 不对。Codex 固定用 `/usr/bin/sandbox-exec`。
- “只要有代理配置，网络就一定会被完全放开。” 不对。Codex 会尽量只放行能推导出来的 loopback/Unix socket 目标，推不出来就会更保守地失败闭合。
- “`allow_local_binding` 等于允许访问外网。” 不对。它主要是在说本机回环流量。
- “`dangerously_allow_all_unix_sockets` 只是更长一点的 allowlist。” 不对。它是明显更宽的放行。
- “文件系统里一个 root 能写，就代表下面所有目录都安全。” 不对。源码会单独把 `.git`、`.codex` 这类子路径拎出来保护。

## 从源码回读的路线

如果你想顺着源码再读一遍，我建议按这个顺序：

1. 先看 [`protocol.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L848)，先弄清 `SandboxPolicy` 的几种模式。
2. 再看 [`permissions.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/permissions.rs#L258-L341)，看文件系统策略怎么拆、怎么保留默认保护目录。
3. 然后看 [`core/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/seatbelt.rs#L18-L47)，理解 macOS 启动入口。
4. 接着看 [`sandboxing/src/seatbelt.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L373-L499)，重点看 profile 怎么拼、`-D` 参数怎么来。
5. 最后回到 [`spawn.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L50-L124)，看子进程是怎么被真正拉起来的。

如果你读到一半开始迷糊，最值得回头重看的只有一条线：`意图层 -> 参数层 -> 进程启动层`。只要这条线没丢，Seatbelt 里的很多细节都会慢慢变得可读。
