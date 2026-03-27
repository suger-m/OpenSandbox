# Codex 原生 Windows 沙箱入门

> 说明  
> 这篇文档只讲 Codex 在 **原生 Windows** 上的沙箱路径，不讨论 WSL。内容基于 `47a9e2e084e21542821ab65aae91f2bd6bf17c07` 这个 commit 的源码静态阅读；我会尽量只写“能从代码里直接确认”的行为。凡是我不完全确定的地方，都会明确标注。

## 先用一句话理解

Codex 的 Windows 沙箱不是单一开关，而是一条“先做 setup，再用受限 token + ACL + 防火墙 + 独立桌面运行命令”的链路。  
在当前代码里，它大致分成两种模式：

- `Elevated`：需要提权做一次较完整的初始化，之后再运行受限命令。
- `Unelevated` / `RestrictedToken`：走更轻的旧路径，重点是预检和一些文件系统 ACL 动作。

## 配置是怎么选出来的

Codex 先把配置解析成 `WindowsSandboxLevel`，再决定接下来走哪条路。

- `core/src/windows_sandbox.rs` 里，`[windows].sandbox = "elevated"` 会映射到 `Elevated`。
- `[windows].sandbox = "unelevated"` 会映射到 `RestrictedToken`。
- 如果没有新配置，代码还会看旧的 feature 开关；我能明确确认的旧键里有 `enable_experimental_windows_sandbox = true`，它会被当成 `Unelevated` 的兼容入口。
- `sandbox_private_desktop` 默认是 `true`，所以如果你不显式关闭，Codex 倾向于把最终子进程放到私有桌面上。

两个最小配置例子：

```toml
[windows]
sandbox = "elevated"
sandbox_private_desktop = true
```

```toml
[windows]
sandbox = "unelevated"
sandbox_private_desktop = false
```

如果你还在用旧风格开关，源码里也认可：

```toml
[features]
enable_experimental_windows_sandbox = true
```

我这里不把更多旧 feature 键写死，是因为 `Feature::WindowsSandbox*` 背后的字符串名并不全在我当前阅读的文件里展开。

## `core/windows_sandbox.rs` 做了什么

`core/src/windows_sandbox.rs` 更像“总调度器”：

1. 它把配置和 feature 解析成 `WindowsSandboxLevel`。
2. 它封装 `WindowsSandboxSetupRequest`，把 policy、cwd、环境变量和 `codex_home` 一起传给 setup。
3. `run_windows_sandbox_setup` 会先记时，再进入真正的 setup。
4. 如果是 `Elevated`，它会先检查 `sandbox_setup_is_complete(codex_home)`。
5. 如果 setup 没完成，就调用 `run_elevated_setup(...)`。
6. 如果是 `Unelevated`，它会调用 `run_legacy_setup_preflight(...)`。
7. 成功后，`ConfigEditsBuilder` 会把当前模式写回配置，并清掉旧的 legacy keys。

我从代码里看到，`sandbox_setup_is_complete` 只是一个**粗粒度**判断：它主要看 `setup_marker.json` 和 `sandbox_users.json` 的版本是否匹配。它不是“全量验证一切都没问题”的健康检查。

## setup helper 的启动流程

真正干活的是 `windows-sandbox-rs` 里的 setup orchestrator。

- `setup_orchestrator.rs` 会先组装一个 `ElevationPayload`，把 read roots、write roots、proxy 端口、`allow_local_binding` 等信息编码进去。
- 然后它把 payload 做成 base64。
- 如果当前进程**没有提权**，它会直接 `Command::status()` 启动 helper，并加上 `CREATE_NO_WINDOW`。
- 如果当前进程**已经提权**，它会用 `ShellExecuteExW` + `runas` 重新拉起 helper，并把窗口隐藏。
- helper 失败时，orchestrator 会尝试读取 `setup_error.json`，把结构化错误带回去。

这意味着：同一个 helper 二进制，可以被两种启动方式使用，但它们的权限上下文不同。

## helper 里到底做了什么

`setup_main_win.rs` 是实际执行 setup 的入口。我能确认的主流程是：

- 先 base64 解码 payload，并检查 `SETUP_VERSION`。
- 再确保 `CODEX_HOME/.sandbox` 和日志目录存在。
- 如果是完整 setup，它会：
  - provision offline / online 沙箱用户；
  - 加载或创建 capability SIDs；
  - 配置离线网络的防火墙规则；
  - 必要时再拉起一个只负责读 ACL 的 helper；
  - 锁定 `.sandbox`、`.sandbox-bin`、`.sandbox-secrets` 等目录。
- 最后它会把 setup 相关的 marker / 用户工件落盘，让后续流程知道 setup 是否完成。

这里我需要保守一点：我能确认“做了这些事”，但像某些目录的最终 ACE 顺序、或者 Windows 防火墙 UI 里怎么展示这些规则，我没有在本次静态阅读里做运行验证。

## 受限 token、capability SID 和 ACL

这是 Windows 沙箱最核心的一层。

### 1) 受限 token

`token.rs` 里，Codex 会：

- 打开当前进程 token；
- 从 token 里取 logon SID；
- 调 `CreateRestrictedToken`；
- 使用 `DISABLE_MAX_PRIVILEGE`、`LUA_TOKEN`、`WRITE_RESTRICTED` 这些标志；
- 最后只显式保留 `SeChangeNotifyPrivilege`。

从源码看，**capability SID、logon SID 和 Everyone** 会一起参与 token / DACL 的构造。换句话说，Codex 不是只靠“降权”，而是同时把“谁能访问什么”编码进 token 本身。

### 2) capability SID

`cap.rs` 会持久化一组随机 SID：

- `readonly`
- `workspace`
- `workspace_by_cwd`

我更愿意把它们理解成“沙箱内部的能力标签”。它们不是通用的 Windows 常量，而是 Codex 自己生成并保存的身份标记。

### 3) ACL shaping

`acl.rs` 和 `setup_main_win.rs` 负责把 token 变成真正可用的文件系统权限：

- 代码会检查 DACL 里是否已经存在需要的 allow / deny ACE。
- 如果没有，就补上。
- `setup_main_win.rs` 里还会对当前工作区的 `.codex` 和 `.agents` 做单独保护。

我能确认的结论是：Codex 在这里做的是“按需补 ACL”，而不是每次都重建整个权限树。

## 防火墙规则

`firewall.rs` 负责离线身份的网络限制。源码里能看到它会创建或更新 idempotent 的 Windows Firewall 规则，并把规则限制到特定的 `LocalUserAuthorizedList`，也就是离线沙箱用户的 SID。

从当前实现看，它大致做两件事：

- 阻断非 loopback 的出站流量；
- 在 proxy-only 场景下，对 loopback TCP/UDP 做更细的限制，并尽量只放行代理需要的端口。

如果 `allow_local_binding = true`，代码会倾向于移除旧的 proxy 例外规则，而不是继续保留它们。这个路径的目标很像“fail closed”，不过我没有做系统级实测，所以这里把它表述为源码意图更稳妥。

## 私有桌面

`desktop.rs` 里有一块很容易被忽略，但其实很重要：

- `LaunchDesktop::prepare(true, ...)` 会创建一个随机名字的桌面，比如 `CodexSandboxDesktop-...`。
- 它再给这个桌面写一份 DACL，只允许沙箱的 logon SID 访问。
- `process.rs` 会把这个桌面名写进 `STARTUPINFOW.lpDesktop`。

源码里的注释提到，某些以 restricted token 启动的进程如果不显式设置 `lpDesktop`，可能会出现启动失败。  
所以“私有桌面”不是视觉特效，而是一个更隔离的交互上下文。

## 你可以怎么记住整条链路

最容易记的版本是：

1. `core/windows_sandbox.rs` 决定模式，并决定要不要重新做 setup。
2. `setup_orchestrator.rs` 把需要的 roots 和网络信息打包，拉起 helper。
3. `setup_main_win.rs` 在提权上下文里修用户、修 ACL、修防火墙、写 marker。
4. `token.rs` / `cap.rs` / `acl.rs` / `firewall.rs` / `desktop.rs` 负责把“受限”真正落到 Windows API 上。
5. 最终命令由 `process.rs` 用受限 token、指定 desktop 和标准 IO 运行起来。

## 我会保留的几个不确定点

- 我没有运行这套源码，只做了静态阅读，所以某些边界条件只能说“代码看起来是这样”。
- `run_windows_sandbox_legacy_preflight(...)` 从函数体看更像是 ACL 预检 / 补写路径，我没有在这个函数里看到完整的用户 provisioning 或防火墙初始化。
- `firewall.rs` 的 Windows Firewall 行为我没有在真实机器上复核，最好把它当成“源码层面的设计说明”，而不是现场排障结论。

## 参考源码

以下链接都锁定在同一个 commit：`47a9e2e084e21542821ab65aae91f2bd6bf17c07`。

- [`core/src/windows_sandbox.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs)
- [`core/src/config/types.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/config/types.rs)
- [`core/src/config/mod.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/config/mod.rs)
- [`windows-sandbox-rs/src/lib.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/lib.rs)
- [`windows-sandbox-rs/src/token.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/token.rs)
- [`windows-sandbox-rs/src/acl.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/acl.rs)
- [`windows-sandbox-rs/src/firewall.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/firewall.rs)
- [`windows-sandbox-rs/src/desktop.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/desktop.rs)
- [`windows-sandbox-rs/src/setup_orchestrator.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs)
- [`windows-sandbox-rs/src/setup_main_win.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_main_win.rs)
- [`windows-sandbox-rs/src/elevated_impl.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/elevated_impl.rs)
- [`windows-sandbox-rs/src/identity.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/identity.rs)
- [`windows-sandbox-rs/src/process.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/process.rs)
