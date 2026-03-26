# Codex 原生 Windows 沙箱入门

> 说明  
> 这篇文档只讲 Codex 在 **原生 Windows** 上的沙箱路径，不讨论 WSL。内容基于本地 Codex 来源 `47a9e2e084e21542821ab65aae91f2bd6bf17c07` 的**静态阅读**。我会先讲 Windows 机制，再讲 Codex 如何把这些机制拼起来。
> 只要不是我能从源码直接确认的结论，我都会明确标成 **推断**。

如果你对权限边界、token、SID、ACL、capability SID 这些概念还不熟，建议先读 [Codex Sandbox 技术基础入门](./codex-sandbox-foundations.md)。

## 先有一个总印象

很多人第一次看到“Windows 沙箱”时，会下意识以为它是一个单独的 API。其实不是。
在 Windows 上，类似 Codex 这样的沙箱更像是几层机制叠在一起：

1. 用 **token** 决定“这个进程是谁，带着哪些权限和限制”。
2. 用 **SID / capability SID** 给“身份”和“能力标签”编号。
3. 用 **ACL / DACL / ACE** 决定“这些身份到底能碰哪些文件、目录、桌面对象”。
4. 用 **防火墙规则** 决定“这个身份能不能连网络、能连哪些地址和端口”。
5. 用 **desktop / window station / session** 把图形和交互对象隔开。
6. 用一个短暂的 **提权 helper** 做初始化，再回到受限上下文执行真正命令。

所以你可以把 Codex 的 Windows 沙箱理解成：

> 先做一次受控的系统级准备工作，再让真正的命令在受限身份下运行。

---

## 先补几个必备名词

### Token 是什么

Windows 里的 token 可以粗略理解成“进程随身带着的一张身份卡”。

- 它记录这个进程属于哪些用户和组。
- 它记录这个进程有哪些权限。
- 它也可以带上“受限 SID”或被裁剪过的权限集合。

Windows 在做很多访问检查时，都会拿“进程的 token”去和目标对象的 ACL 对比。

### Restricted Token 是什么

Restricted token 就是“被裁过”的 token。

- 一部分高权限会被拿掉。
- 一部分 SID 会变成“只用来额外限制访问”的 restricted SID。
- 它常用于让子进程以更弱的身份运行。

这不是虚拟机，也不是容器，而是 Windows 原生的“降权运行”能力。

### SID 是什么

SID 是 Security Identifier，可以把它理解成 Windows 里的“身份编号”。

- 用户有 SID。
- 组有 SID。
- 某些能力标签也可以用 SID 表达。

ACL 里并不是写“张三可读”，而是写“某个 SID 允许什么”。

### Capability SID 是什么

Capability SID 可以粗略理解成“能力标签 SID”。

它和“用户是谁”不是一回事，更像是在说：

> 这个 token 被打上了某种能力标签，所以只要某个对象的 ACL 愿意认这个标签，它就能访问。

Codex 里有自己的 capability SID，例如：

- `readonly`
- `workspace`
- `workspace_by_cwd`

这些不是 Windows 预定义的通用常量，而是 Codex 持久化出来、自己拿来做访问控制的 SID 字符串。

### ACL / DACL / ACE 分别是什么

这三个名字很容易混。

- ACL：Access Control List，权限列表。
- ACE：Access Control Entry，ACL 里的单条规则。
- DACL：Discretionary ACL，Windows 最常见的“允许/拒绝谁访问”的那张 ACL。

可以把它想成：

- DACL 是整张门禁名单。
- ACE 是名单里的某一行。
- 每一行会写某个 SID 被允许或拒绝哪些操作。

### Session、Window Station、Desktop 是什么

如果你不做 Windows 图形编程，这三个词会很陌生，但这里只要记一个够用版本：

- Session：一次登录会话的边界。
- Window Station：一组桌面对象的容器。
- Desktop：窗口、菜单、剪贴板等 GUI 对象所处的具体桌面。

最常见的交互式桌面路径是 `Winsta0\\Default`。

**推断**：从 Codex 当前源码看，它显式使用 `Winsta0\\...`，所以重点不是“新建 session”，而是“在现有交互式 window station 下切到默认桌面或私有桌面”。

---

## 机制 1：受限 Token

#### 它是什么

受限 token 是把原始 token 经过 `CreateRestrictedToken` 之类的 API 裁剪后得到的新 token。

Codex 的 `token.rs` 里可以直接看到：

- 它调用 `CreateRestrictedToken`
- 使用了 `DISABLE_MAX_PRIVILEGE | LUA_TOKEN | WRITE_RESTRICTED`
- 最后显式保留 `SeChangeNotifyPrivilege`

#### 它解决什么

它解决的是“子进程不要继承调用者的完整能力”。

也就是说，就算发起沙箱的是一个普通用户的交互进程，Codex 仍然会尽量把真正执行命令的 token 再削弱一层。

#### 它不解决什么

它**不直接等于**文件隔离、网络隔离或 UI 隔离。

- 目标对象的 ACL 如果写得很宽，降权后也可能还能访问。
- 它不会自动帮你拦网络。
- 它也不会自动把进程移到另一个 desktop。

所以 restricted token 很重要，但它只是整条链路的一层。

#### Codex 怎么用

Codex 会把 capability SID、logon SID、Everyone 一起放进受限 token 的构造过程里，并给新 token 设一个较宽松的默认 DACL，避免 PowerShell 管道这类 IPC 对象因为 ACL 太严而直接报 `ACCESS_DENIED`。

这说明 Codex 不是只做“减权限”，而是同时在准备“这个 token 后面要如何和 ACL 配合工作”。

---

## 机制 2：SID 和 Capability SID

#### 它是什么

SID 是访问控制里用来指代主体的编号。Capability SID 则是“能力标签”的编号。

Codex 的 `cap.rs` 会在 `codex_home/cap_sid` 里加载或创建一组 SID：

- 一个通用 `workspace`
- 一个通用 `readonly`
- 一组按规范化工作目录区分的 `workspace_by_cwd`

#### 它解决什么

它解决的是“别只靠用户名或用户组做授权”。

如果只有“这个用户能写”这种粗粒度身份，多个工作区之间就不好拆分。
有了 workspace-specific capability SID，就可以表达：

- 某些目录只允许只读能力访问
- 某些目录允许工作区写能力访问
- 某个具体工作区还可以有自己的专属能力 SID

#### 它不解决什么

Capability SID 自己并不会产生权限。

- 没有 ACL 认它，它就只是个标签。
- 没有 token 携带它，也起不了作用。

所以它本身不是“神奇权限”，更像是“后续 ACL 可以引用的名字”。

#### Codex 怎么用

Codex 会持久化这些 capability SID，然后：

- 在创建 restricted token 时把对应 capability SID 放进去
- 在写文件系统 ACL 时，把这些 capability SID 填进 ACE
- 对当前工作区额外生成 `workspace_by_cwd` SID，用来保护 `.codex` 和 `.agents`

这一步的意义是：Codex 不只是在说“谁是这个用户”，还在说“这个进程此刻带着哪一种工作区能力”。

---

## 机制 3：ACL / DACL / ACE 以及为什么要做 ACL shaping

#### 它是什么

Windows 最终仍然是靠 ACL 来判定文件、目录、桌面对象能不能访问。

如果前面的 token 和 capability SID 是“持证上岗”，那 ACL 就是门口的门禁规则。

Codex 的 `acl.rs` 和 `setup_main_win.rs` 里能直接看到几类动作：

- 读取对象当前 DACL
- 检查已有 ACE 是否已经满足访问掩码
- 缺什么 ACE 就补什么
- 某些地方追加 deny ACE 来防止写入或删除

#### 它解决什么

它解决的是“让 Windows 的原生访问检查真正落到文件和目录上”。

这是整个设计里非常关键的一点。因为：

- 只有 restricted token，不足以表达“这个目录可写，那个目录只读”。
- 只有 capability SID，不足以表达“这个标签在这里可用，在那里不可用”。

只有把这些 SID 真正写进对象的 ACL，Windows 才会在每次访问时做出想要的判定。

这就是为什么 ACL shaping 很重要。
它不是装饰，而是把抽象策略翻译成 Windows 真正执行的规则。

#### 它不解决什么

ACL 也不是万能的。

- 它不替代网络隔离。
- 它不处理已经打开的句柄。
- 它不保证系统里别的宽松 ACL 完全不存在。
- 它也不是对内核漏洞或提权漏洞的防护。

所以你应该把 ACL 理解为“对象级访问边界”，而不是“整个系统都被隔离了”。

#### Codex 怎么用

Codex 在这层做的不是“每次重建整棵权限树”，而更像“按需补齐”：

- `ensure_allow_write_aces` 会检查已有 DACL，不够才补写
- setup 时会给 write roots 授权
- read roots 会走单独的 read ACL helper
- 当前工作区的 `.codex` 和 `.agents` 会追加 deny ACE，阻止工作区写能力去篡改这些控制目录
- `filter_sensitive_write_roots` 明确避免把 `CODEX_HOME`、`.sandbox`、`.sandbox-bin`、`.sandbox-secrets` 这类目录变成 capability 可写

这正是“ACL shaping”的落地方式。

---

## 机制 4：防火墙规则

#### 它是什么

Windows 防火墙不只是“按程序名拦”，也可以按用户或 SID 定位规则。

Codex 的 `firewall.rs` 直接使用 COM 接口去创建或更新规则，并给规则设置稳定的内部名字，这样后续可以幂等地重复执行。

#### 它解决什么

它解决的是“即使文件系统权限已经收紧，离线沙箱用户仍然不能随便对外连网”。

从代码看，Codex 至少做两件事：

- 对离线身份阻断非 loopback 出站流量
- 在代理模式下，对 loopback TCP/UDP 做更细的限制，只尽量放行代理需要的端口

#### 它不解决什么

防火墙规则也不是完整网络虚拟化。

- 它不替代 token 和 ACL。
- 它不自动约束本地文件访问。
- 它主要限制“这个用户身份的网络行为”，不是容器式的完整协议栈隔离。

另外，Windows 防火墙在不同系统版本和企业策略下的表现细节可能不完全一样。

#### Codex 怎么用

Codex 的 helper 会为“离线沙箱用户”安装或更新规则：

- 使用稳定的规则名，方便重复执行时覆盖更新
- 使用 `LocalUserAuthorizedList` 风格的用户约束
- 在 `allow_local_binding = true` 时，主动清掉旧的 proxy-only 例外规则，避免脏状态残留

源码里还明确写了几处“先放更宽的 block，再缩窄到代理例外”的注释，目标是尽量 **fail closed**。
这里的 “fail closed” 是**源码意图**，不是我做过系统级实测后的运维结论。

---

## 机制 5：私有 Desktop、Window Station、Session

### 它是什么

Windows 的 GUI 对象不是直接挂在“进程”上，而是挂在 desktop / window station / session 这套对象模型上。

Codex 的 `desktop.rs` 会：

- 在需要时创建随机名的 desktop，例如 `CodexSandboxDesktop-...`
- 把启动目标桌面设置成 `Winsta0\\<private desktop>`
- 给这个 desktop 写 DACL，只授予当前 logon SID 访问

而 `process.rs` 会把桌面名放进 `STARTUPINFOW.lpDesktop`。

### 它解决什么

它解决的是“不要默认和当前交互桌面共享同一套 GUI 对象”。

这至少有两层现实意义：

1. 某些受限 token 进程如果不显式指定 `lpDesktop`，可能直接启动失败。
2. 即使成功启动，单独 desktop 也能减少和默认桌面的 GUI 对象混用。

### 它不解决什么

私有 desktop 不是“整个系统隔离”。

- 它不隔离文件系统。
- 它不隔离网络。
- 它不等于新建 session。
- 它也不等于新建 window station。

**推断**：当前代码显式使用 `Winsta0\\...`，我没有看到它创建新的 window station 或新的 session，所以更准确的说法应该是“在现有交互式 window station 里切换到私有 desktop”。

### Codex 怎么用

Codex 把私有 desktop 当成一个可配置的补充隔离层。

- `sandbox_private_desktop` 默认值是 `true`
- 如果开启，子进程会运行在私有 desktop
- 如果关闭，就退回 `Winsta0\\Default`

这说明它的定位是“提高兼容性和隔离度”，而不是替代 token 或 ACL。

---

## 机制 6：Helper / Orchestrator 的提权模式

### 它是什么

很多 Windows 安全动作不是普通受限进程随手就能做的，例如：

- 建立或更新本地沙箱用户
- 改某些系统级 ACL
- 改防火墙规则

所以常见做法是：

1. 正常进程负责收集需求
2. 把 setup 参数打包
3. 拉起一个短暂存在的提权 helper
4. helper 做完系统级初始化后退出
5. 真正的业务命令仍然回到受限上下文运行

### 它解决什么

它解决的是“把必须提权的动作集中到一个很小的初始化窗口里”，避免整个主程序一直带着高权限跑。

这是一种很典型的 helper / orchestrator 分工：

- orchestrator 决定“该做什么”
- helper 负责“用足够权限把它做完”

### 它不解决什么

它不代表“真正执行用户命令的进程也是提权的”。

恰恰相反，这种模式的目标通常就是：

> 只让 setup 过程短暂提权，不让最终命令长期提权。

它也不替代 ACL 或 token 设计。
如果 helper 把权限准备错了，后面的 restricted token 仍然可能拿到不该有的访问。

### Codex 怎么用

Codex 的 `setup_orchestrator.rs` 会把 read roots、write roots、代理端口、`allow_local_binding` 等信息序列化进 `ElevationPayload`。

然后分两种启动路径：

- 不需要提权时，直接 `Command::status()` 启动 helper，并隐藏窗口
- 需要提权时，用 `ShellExecuteExW` + `runas` 拉起 helper，并等待退出码

helper 入口在 `setup_main_win.rs`，会做这些事：

- 校验 setup 版本
- 准备 `.sandbox` 和日志目录
- 加载或创建 capability SID
- 处理离线/在线沙箱用户
- 安装或刷新防火墙规则
- 刷新读写 ACL
- 锁定 `.sandbox`、`.sandbox-bin`、`.sandbox-secrets`
- 写 marker 和用户工件

这就是 Codex 把“短暂提权初始化”和“长期受限执行”拆开的方式。

---

## Codex 的两种模式：Elevated 和 Unelevated 到底差在哪

这是最容易混淆的地方。

### `Elevated`

#### 它是什么

这是当前更完整的 setup 路径。配置上会映射到 `WindowsSandboxLevel::Elevated`。

#### 它解决什么

它能做一套较完整的系统准备：

- 检查 setup marker 和 sandbox users 工件是否匹配当前版本
- 在需要时触发提权 helper
- 准备在线/离线沙箱用户凭据
- 刷新 capability SID、ACL、防火墙和控制目录保护

从 `identity.rs` 看，`sandbox_setup_is_complete` 只是**粗粒度检查**；真正取凭据时还会额外看网络身份和离线代理设置是否漂移，不对就重新 setup。

#### 它不解决什么

它不是“做完 setup 就永远不用管了”。

- 读写 roots 变化时还要 refresh
- 防火墙目标端口变了也可能触发重新 setup
- 最终命令仍然应该在受限身份下执行

#### Codex 怎么用

Codex 会在 elevated 路径里：

- 先判断 setup 是否完成
- 必要时跑一次提权 helper
- 后续再拿到沙箱用户凭据并继续 refresh ACL
- 最终再用受限上下文启动实际命令

你可以把它理解成“先把沙箱环境搭好，再让命令进去跑”。

### `Unelevated` / `RestrictedToken`

#### 它是什么

这是兼容旧 feature 的较轻路径，配置上会映射到 `WindowsSandboxLevel::RestrictedToken`。

#### 它解决什么

它主要解决的是“不做完整提权 setup 的前提下，至少把工作区写权限和一些保护规则补起来”。

从 `run_windows_sandbox_legacy_preflight(...)` 可以直接看到，它会在 `WorkspaceWrite` 场景下：

- 确保 `codex_home` 存在
- 加载或创建 capability SID
- 为 allow 路径补 allow ACE
- 为 deny 路径补 deny ACE
- 处理 `NUL` 设备访问
- 保护工作区里的 `.codex` 和 `.agents`

#### 它不解决什么

从我能看到的源码范围里，它**不像 elevated 那样**完整负责：

- provision 在线/离线沙箱用户
- 安装离线防火墙规则
- 维护 marker + sandbox users 的整套 setup 生命周期

这也是它和 elevated 最核心的差别。

#### Codex 怎么用

Codex 把它当成 legacy/兼容路径。

- 新配置里写 `sandbox = "unelevated"` 会走这里
- 旧配置 `enable_experimental_windows_sandbox = true` 也会落到这里
- 它更依赖“restricted token + 预检 ACL shaping”

如果你只记一句话：

> `Elevated` 是“完整 setup + 受限执行”，`Unelevated` 更像“旧式预检 + 受限执行”。

---

## 配置怎么映射到这两种模式

当前源码里可以直接确认这些映射：

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

旧风格兼容入口：

```toml
[features]
enable_experimental_windows_sandbox = true
```

还可以直接确认两点：

- `sandbox = "elevated"` -> `WindowsSandboxLevel::Elevated`
- `sandbox = "unelevated"` -> `WindowsSandboxLevel::RestrictedToken`
- `sandbox_private_desktop` 默认值是 `true`

---

## 把整条链路串起来

如果你想用一句顺口的话记住 Codex 的原生 Windows 沙箱，可以记成：

> 先用 helper 把系统环境准备好，再用 restricted token + capability SID + ACL + 防火墙 + 私有 desktop 去运行真正命令。

更细一点就是：

1. `core/src/windows_sandbox.rs` 解析配置，决定走 `Elevated` 还是 `RestrictedToken`。
2. 如果是 elevated，orchestrator 评估是否需要重新 setup，并打包 `ElevationPayload`。
3. helper 在需要的权限上下文里刷新用户、SID、ACL、防火墙和 marker。
4. 实际命令进程拿着受限 token 启动。
5. 如果开启私有 desktop，`lpDesktop` 会被显式指到 `Winsta0\\<private desktop>`。

---

## 我保留的推断和不确定点

- **推断**：当前实现重点是“私有 desktop”，不是“新建 session”或“新建 window station”，因为我没有看到创建新 session / window station 的代码，只看到 `Winsta0\\...` 的 desktop 选择。
- **推断**：防火墙规则在真实 Windows UI 中的最终展示细节，我没有做运行验证；文中只把源码里能确认的规则模型写出来。
- **推断**：`Unelevated` 的定位来自它当前调用链和 `legacy_preflight` 的实现范围。就我这次静态阅读能确认的内容，它主要是 ACL 预检和 restricted token 路径，而不是完整 setup 生命周期。

---

## 参考源码

以下链接都锁定到同一个 commit：`47a9e2e084e21542821ab65aae91f2bd6bf17c07`。

- [`core/src/windows_sandbox.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/windows_sandbox.rs)
- [`windows-sandbox-rs/src/token.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/token.rs)
- [`windows-sandbox-rs/src/cap.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/cap.rs)
- [`windows-sandbox-rs/src/acl.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/acl.rs)
- [`windows-sandbox-rs/src/firewall.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/firewall.rs)
- [`windows-sandbox-rs/src/desktop.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/desktop.rs)
- [`windows-sandbox-rs/src/process.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/process.rs)
- [`windows-sandbox-rs/src/setup_orchestrator.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs)
- [`windows-sandbox-rs/src/setup_main_win.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_main_win.rs)
- [`windows-sandbox-rs/src/identity.rs`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/identity.rs)
