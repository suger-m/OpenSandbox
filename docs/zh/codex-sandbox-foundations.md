# Codex Sandbox 技术基础入门

> 这篇文档不是按源码文件讲解，而是先补齐你在读 Codex 平台文档前最容易缺的操作系统和安全基础。  
> 目标读者是新手：你不需要先会写内核代码，也不需要先懂完整的 macOS、Linux、Windows 安全模型。  
> 你只需要先建立一个判断框架：一个 sandbox 到底在限制什么，它靠什么机制限制，又有什么边界。

如果你第一次接触这类系统，最容易把 sandbox 想成一个“大开关”:

- 开了，就安全。
- 关了，就不安全。

真实情况不是这样。实际平台通常是把很多小机制叠在一起:

- 谁来启动进程
- 进程能读写哪些路径
- 进程能不能创建网络连接
- 进程能不能提权
- 进程是不是被放进新的命名空间
- 某些图形界面或桌面对象是否单独隔离

你可以把它想成“多道门”而不是“一把锁”。

这篇文档会按概念来讲。每个概念都回答 4 个问题:

- 它是什么
- 它解决什么问题
- 它不解决什么问题
- 它在 Codex 里通常会出现在哪

最后一节会把这些概念重新映射回 3 篇平台文档:

- [`codex-sandbox-linux.md`](./codex-sandbox-linux)
- [`codex-sandbox-macos.md`](./codex-sandbox-macos)
- [`codex-sandbox-windows.md`](./codex-sandbox-windows)

---

## 先建立一张总图

先记住一句话:

> Codex 不是用单一机制做隔离，而是把“策略描述”翻译成各个平台真正能执行的限制组合。

一个非常粗略但很好用的阅读图是:

```text
高层策略
  -> 文件系统策略 / 网络策略
  -> helper 或 orchestrator 准备环境
  -> 进程启动
  -> 命名空间 / token / ACL / 防火墙 / seccomp / desktop 等机制真正落地
  -> 最终用户命令运行
```

如果你一会儿读源码时又迷糊了，就回到这张图上重新对位。

---

## 1. 进程与线程

### 1.1 进程 `process`

- 是什么  
  进程可以先理解成“一个独立运行中的程序实例”，它有自己的进程 ID、地址空间、打开的文件句柄、环境变量、权限上下文。
- 解决什么问题  
  平台需要一个明确的“被隔离对象”。你总得先有一个要运行的进程，后面才能谈文件权限、网络限制、提权限制。
- 不解决什么  
  仅仅把代码跑成一个进程，并不会自动隔离。一个普通进程如果没有额外限制，依然可能读主机文件、连外网、继承父进程权限。
- 在 Codex 里常出现在哪  
  最直接的入口是进程启动逻辑，比如 `spawn_child_async` 会拉起子进程；Windows 侧还会在受限 token 和桌面上下文里启动进程。  
  例子：shell tool 最终不是“执行一段字符串”，而是“启动一个受控的子进程”。

### 1.2 线程 `thread`

- 是什么  
  线程是进程里的执行单元。一个进程里可以有多个线程，它们共享同一个地址空间，但调度单位更细。
- 解决什么问题  
  有些限制希望只作用在“即将执行用户命令的那个执行路径”上，而不是整个父进程。这样平台自己还能继续工作，不会把自己也锁死。
- 不解决什么  
  线程不是安全边界。两个线程在同一进程里共享很多资源，所以“开了多线程”不等于“做了隔离”。
- 在 Codex 里常出现在哪  
  Linux 里有一句非常关键的话: `apply_sandbox_policy_to_current_thread`。意思是把 seccomp/Landlock 这类限制加到当前线程上，只让即将继承它的子进程受影响，而不是把整个 CLI 进程都弄残。  
  这也是为什么你会在 Linux helper 里看到 “apply on current thread” 这种说法。

### 1.3 进程 vs 线程，先怎么记

可以先这样粗略记:

- 进程像“一个独立房间”
- 线程像“房间里的人”

你通常不会拿“房间里的人”当主要隔离边界；你会先关房门，再决定房间里的人能做什么。

---

## 2. 系统调用与权限边界

### 2.1 系统调用 `system call`

- 是什么  
  系统调用是用户态程序向操作系统内核请求服务的入口，比如 `open`、`read`、`write`、`connect`、`socket`、`execve`。
- 解决什么问题  
  它给操作系统一个统一的“闸门”。程序不能直接自己去改内核状态，必须通过 syscall 请求。
- 不解决什么  
  syscall 只是入口，不是策略本身。你知道程序调用了 `connect`，不代表系统就一定会允许。
- 在 Codex 里常出现在哪  
  Linux 的 seccomp 过滤器本质上就是在拦 syscall。比如限制 `connect`、`bind`、`socket` 等网络相关调用。  
  如果你看到 `SYS_connect`、`SYS_socket`、`SYS_ptrace` 这些名字，通常就是在“拦哪类系统调用”。

### 2.2 权限边界 `privilege boundary`

- 是什么  
  权限边界指的是“低权限一侧不能随便跨过去拿高权限能力”的那条线。比如普通进程不能随便变管理员，受限 token 不能自动恢复成完整 token。
- 解决什么问题  
  它防止“我先在受限环境里跑，后面再偷偷升级权限”。
- 不解决什么  
  权限边界不自动等于资源边界。一个低权限进程如果仍被允许访问某路径、某 socket、某桌面对象，它依然能碰那些资源。
- 在 Codex 里常出现在哪  
  Linux 里常和 `no_new_privs` 绑定出现；Windows 里则经常体现在 restricted token、logon SID、ACL、private desktop 这些组合上。  
  读文档时，凡是看到“不要让后续执行拿到更高权限”，本质都在谈 privilege boundary。

---

## 3. 策略、执行和文件系统控制

### 3.1 文件系统读写控制

- 是什么  
  文件系统读写控制就是定义“这个进程能读哪些路径，能写哪些路径，哪些子路径要继续保护”。
- 解决什么问题  
  避免命令随便修改仓库、凭据目录、配置目录，或者把整个系统盘当工作区乱写。
- 不解决什么  
  它不负责网络，也不负责 CPU/内存限制，更不负责 UI 隔离。
- 在 Codex 里常出现在哪  
  高层会先把允许读写的根目录表达成策略；Linux 用 bubblewrap 构建只读视图再给可写 root 开洞；macOS 用 Seatbelt 规则表达；Windows 用 ACL 和 capability SID 把哪些目录可写编码进去。  
  例子：允许写工作区，但把 `.git`、`.codex` 继续锁成只读。

### 3.2 策略 `policy` vs 执行 `enforcement`

- 是什么  
  `policy` 是“我想允许什么、禁止什么”的描述。`enforcement` 是“操作系统到底怎么把这件事做出来”。
- 解决什么问题  
  把“意图”与“平台实现”拆开后，Codex 才能在 Linux、macOS、Windows 上表达相似语义，但用不同底层机制落地。
- 不解决什么  
  有策略不代表已经强制执行；有强制执行也不代表表达得准确。两边都要对上才算真的安全。
- 在 Codex 里常出现在哪  
  协议层有 `SandboxPolicy`、`FileSystemSandboxPolicy`、`NetworkSandboxPolicy` 这类结构；平台层再分别翻译成 Seatbelt、bubblewrap/seccomp、Windows token/ACL/firewall/desktop。  
  这是理解整套文档最关键的一组概念。

### 3.3 为什么这组概念很重要

新手常见误解是:

- “有 policy 就够了”
- “有 enforcement 就够了”

都不对。  
你真正想要的是:

- 规则表达得清楚
- 规则真的被 OS 执行
- 规则执行失败时不要静默放过

---

## 4. DAC、MAC、ACL、token、SID

这一组是 Windows 文档里最容易把人看晕的，也是理解“谁有权访问什么”的基础。

### 4.1 DAC `Discretionary Access Control`

- 是什么  
  DAC 可以先理解成“资源拥有者或其代表来决定谁能访问”。传统文件权限大多属于这类思路。
- 解决什么问题  
  它提供日常的“谁能读、谁能写、谁能执行”的基本权限模型。
- 不解决什么  
  DAC 对“就算拥有者想放开，也必须继续限制”的场景不够强。它更像常规权限管理，不是强制型安全模型。
- 在 Codex 里常出现在哪  
  Windows 的 DACL/ACE、常规文件权限思路，读起来都带有 DAC 味道。  
  理解这点后，你就知道为什么 Codex 需要再叠 token、capability SID、private desktop 这些机制。

### 4.2 MAC `Mandatory Access Control`

- 是什么  
  MAC 可以先理解成“系统强制规定边界，不由普通资源拥有者随意改”。Linux seccomp、macOS Seatbelt 读起来都更接近这种强制味道。
- 解决什么问题  
  防止“对象拥有者自己把门打开”。对于沙箱尤其重要，因为平台想要的是可预期的硬边界。
- 不解决什么  
  MAC 不会自动定义所有业务权限，也不会替你决定哪些目录本来就该共享。
- 在 Codex 里常出现在哪  
  macOS 的 Seatbelt、Linux 的 seccomp 和 namespace，都更像平台在借助 OS 的强制能力收紧边界。

### 4.3 ACL `Access Control List`

- 是什么  
  ACL 是“访问控制列表”。你可以把它理解成挂在资源上的一张名单，列出哪些身份被允许或被拒绝做什么。
- 解决什么问题  
  它比简单的 `rwx` 位更细，可以按不同 SID、不同权限位做精细控制。
- 不解决什么  
  ACL 只管“这个对象怎么授权”，不负责进程网络，也不负责提权阻断。
- 在 Codex 里常出现在哪  
  Windows 的 `acl.rs` 会检查或重写 DACL，给某些 SID 增加允许写、拒绝写等规则。  
  例子：允许 sandbox 身份写工作目录，但继续拒绝写某些敏感子目录。

### 4.4 token

- 是什么  
  Windows token 可以理解成“进程拿着的一张身份和权限凭证包”，里面有用户、组、特权、默认 DACL 等信息。
- 解决什么问题  
  系统需要知道“这个进程到底是谁，它天然带着哪些权限”。
- 不解决什么  
  token 本身不等于文件规则。你还需要 ACL 去说“哪些对象愿意给这张 token 访问”。
- 在 Codex 里常出现在哪  
  Windows 路径里会先拿当前 token，再通过 `CreateRestrictedToken` 做受限版 token，然后用它去启动最终进程。

### 4.5 SID

- 是什么  
  SID 是 Windows 里标识身份的字符串形式 ID，可以对应用户、组、登录会话、能力标签等。
- 解决什么问题  
  它让 ACL、token、桌面对象、防火墙规则都能用同一种身份标识来对齐。
- 不解决什么  
  SID 只是“你是谁”的标签，不是“你能做什么”的完整答案。
- 在 Codex 里常出现在哪  
  你会看到 `Everyone`、logon SID、capability SID，甚至防火墙规则也会按 SID 绑定用户范围。

### 4.6 capability SID

- 是什么  
  capability SID 可以先理解成“平台自定义的一种能力标签身份”。它不是普通人类用户，而是“这类沙箱能力”的身份代号。
- 解决什么问题  
  平台希望把“只读能力”“工作区写能力”“按当前工作区细分的写能力”编码成可授权对象。
- 不解决什么  
  capability SID 不会自动让任何路径可写，仍然要配合 ACL 或 token 使用。
- 在 Codex 里常出现在哪  
  Windows 的 `cap.rs` 会生成并持久化 `readonly`、`workspace`、`workspace_by_cwd` 这类 capability SID；`token.rs` 会把它们塞进 restricted token；`acl.rs` 再按这些身份授予目录权限。  
  例子：不同工作区拿到不同 capability SID，这样某个 workspace 的规则不会误伤另一个 workspace。

---

## 5. Linux 常见基础件

### 5.1 namespace

- 是什么  
  namespace 是 Linux 用来“让进程看到一个缩小版世界”的机制。常见有 user namespace、pid namespace、network namespace。
- 解决什么问题  
  它让进程误以为自己看到的是一个独立环境。比如看不到宿主的 PID 空间，或者不再直接共享宿主网络。
- 不解决什么  
  namespace 不会自动阻止所有危险 syscall，也不会自动定义哪些目录可写。
- 在 Codex 里常出现在哪  
  bubblewrap 会显式加 `--unshare-user`、`--unshare-pid`、必要时 `--unshare-net`。  
  例子：`ProxyOnly` 或 `Isolated` 网络模式，本质都依赖新的网络命名空间。

### 5.2 seccomp

- 是什么  
  seccomp 是 Linux 内核的 syscall 过滤机制。你可以把它理解成“某些系统调用一律不让过，或者只允许满足条件的调用通过”。
- 解决什么问题  
  当文件系统视图已经搭好之后，seccomp 可以再把网络、调试、某些高风险能力继续收紧。
- 不解决什么  
  seccomp 不擅长表达“这棵目录树可写，那棵目录树只读”这种文件系统语义。它主要是拦 syscall。
- 在 Codex 里常出现在哪  
  Linux helper 的内层阶段会在 bubblewrap 之后安装 seccomp。源码里能看到对 `connect`、`bind`、`listen`、`socket`、`ptrace`、`io_uring_*` 等 syscall 的限制。  
  你可以把它记成“外层搭环境，内层堵入口”。

### 5.3 `no_new_privs`

- 是什么  
  `no_new_privs` 是 Linux 内核标志。意思是: 这个进程以及它后面的执行链，不能再通过 setuid 等机制获得更高权限。
- 解决什么问题  
  防止“先装作受限，后面再偷偷提权”。同时，seccomp 通常也要求先打开它。
- 不解决什么  
  它不会决定文件能不能读写，也不会单独完成网络隔离。
- 在 Codex 里常出现在哪  
  Linux 的 `landlock.rs` 里会在需要安装 seccomp 或 legacy Landlock 路径时设置 `PR_SET_NO_NEW_PRIVS`。  
  非常重要的一点是：Codex 不会无脑总是先开它，因为太早开启可能影响依赖 setuid 的 bubblewrap 流程。

---

## 6. 网络、socket 与回环

### 6.1 socket

- 是什么  
  socket 是进程进行 IPC 或网络通信的端点。最常见的是 TCP/UDP 上的 IP socket。
- 解决什么问题  
  它给程序一个标准接口去连外部服务、监听端口、与本机其他进程通信。
- 不解决什么  
  “有 socket”不等于“能连任何地方”。真正能不能连，要看防火墙、sandbox 规则、seccomp、Seatbelt 等。
- 在 Codex 里常出现在哪  
  Linux seccomp 会直接限制 `socket`、`connect`、`bind` 等；macOS Seatbelt 也会决定允许哪些网络 outbound/bind 行为。  
  读源码时看到 `AF_INET`、`AF_INET6`，通常都在谈 IP socket。

### 6.2 Unix socket

- 是什么  
  Unix socket 是“同一台机器上的本地进程通信端点”，通常表现为一个路径，而不是 `IP:port`。
- 解决什么问题  
  有些本地代理、守护进程、数据库连接更适合走它，因为它只在本机内部使用。
- 不解决什么  
  Unix socket 不是外网通信，也不是自动更安全。放开过多 Unix socket，一样可能绕过你原本想收紧的网络路径。
- 在 Codex 里常出现在哪  
  macOS Seatbelt 会区分 `AF_UNIX`，并支持“全部允许”或“按路径白名单允许”的 Unix socket 策略。  
  这是读 macOS 文档时非常容易忽略的一层。

### 6.3 loopback / 回环

- 是什么  
  loopback 是“连回本机自己”的网络地址，例如 `127.0.0.1`、`::1`、`localhost`。
- 解决什么问题  
  平台有时要让进程只和本机代理说话，而不是直接访问外网。loopback 就是很常见的一条“中间通道”。
- 不解决什么  
  允许 loopback 不等于允许互联网，也不等于允许所有本地端口。
- 在 Codex 里常出现在哪  
  macOS Seatbelt 会从代理环境变量里提取 loopback 端口，只允许访问这些推断出来的 `localhost:port`；Windows 防火墙还会区分“阻断非 loopback”和“只对 loopback 代理端口留白名单”。  
  例子：允许 `localhost:3128` 给本地代理用，不代表允许任意公网连接。

### 6.4 firewall rule

- 是什么  
  防火墙规则是操作系统级网络放行/阻断规则。
- 解决什么问题  
  当你需要对“这个身份的进程到底能发到哪些地址和端口”做系统级约束时，它很直接。
- 不解决什么  
  防火墙规则不决定文件访问，也不决定 token 权限，更不替代 seccomp。
- 在 Codex 里常出现在哪  
  Windows 的 `firewall.rs` 会创建或更新规则，阻断离线沙箱用户的非 loopback 出站流量，并在 proxy-only 场景下对 loopback TCP/UDP 再做更细限制。  
  这里能明显看到“规则要可重复应用、可验证、失败时尽量别变成放开状态”。

---

## 7. Windows 图形会话相关概念

如果你只接触过命令行系统，这组概念最容易陌生。但读 Windows 文档时会经常碰到。

### 7.1 desktop / session / private desktop

- 是什么  
  在 Windows 里，`session` 可以粗略理解为一次登录会话的大环境；`desktop` 是这个会话里的桌面对象；`private desktop` 则是单独创建的一块桌面上下文。
- 解决什么问题  
  它把 GUI 交互环境也变成可隔离对象。这样受限进程不一定非得和默认交互桌面共享同一套对象。
- 不解决什么  
  private desktop 不会自动限制文件和网络。它主要是 GUI/窗口对象层面的隔离。
- 在 Codex 里常出现在哪  
  `desktop.rs` 会创建私有桌面并给当前 logon SID 授权；`process.rs` 会把这个桌面名写进 `STARTUPINFO.lpDesktop`。  
  例子：Windows 里如果 restricted token 启动进程时不明确设置 `lpDesktop`，有些进程可能直接起不来，所以这里既是隔离，也是兼容性处理。

---

## 8. helper / orchestrator 模式

### 8.1 helper

- 是什么  
  helper 是平台为了落地某种系统级操作而单独拉起的辅助程序。
- 解决什么问题  
  把“高权限或高复杂度的系统准备动作”从主流程里拆出去，主程序只负责描述需求，helper 负责执行系统调用和平台 API。
- 不解决什么  
  helper 不是安全本身。如果 helper 写错了，它照样可能把本来该收紧的东西放开。
- 在 Codex 里常出现在哪  
  Linux 的 sandbox helper、本质上也是“先准备环境再执行目标命令”的专用程序；Windows 则有专门的 setup helper 可执行文件。

### 8.2 orchestrator

- 是什么  
  orchestrator 是“调度者”。它不一定亲手做底层动作，但它决定何时调用哪个 helper、携带什么 payload、失败时怎么回报。
- 解决什么问题  
  把复杂步骤串起来，并让流程具备可重试、可诊断、可持久化的控制逻辑。
- 不解决什么  
  orchestrator 不是底层 enforcement 本身。它只是把 enforcement 组织起来。
- 在 Codex 里常出现在哪  
  Windows 的 `setup_orchestrator.rs` 很典型: 它收集 read/write roots、代理端口、`allow_local_binding` 等信息，序列化 payload，再去拉起 setup helper。  
  你可以把它理解成“主程序和真正系统修改者之间的调度层”。

### 8.3 helper / orchestrator 模式为什么重要

这是平台代码里非常常见的一种分工:

- 上层只说需求
- 中间层组织流程
- 下层真正碰系统边界

理解了这个模式，你读平台代码时就不会再问:

- “为什么不直接在主流程里改 ACL？”
- “为什么不直接在 CLI 里把防火墙配完？”

因为那样主流程会又重、又难测、又难迁移。

---

## 9. fail-closed 与 fallback

### 9.1 fail-closed

- 是什么  
  `fail-closed` 的意思是: 如果平台不能确认“安全规则已经正确生效”，那就宁可更保守，也不要默默放开。
- 解决什么问题  
  防止“本来想限制，结果失败后反而无限制运行”。
- 不解决什么  
  fail-closed 不能保证功能一定可用。它常常会牺牲可用性来换安全边界。
- 在 Codex 里常出现在哪  
  Linux 的 managed network 场景会强调保持 fail-closed；Windows 防火墙在 proxy-only 过渡时会先安装更广的阻断规则，再逐步缩到允许的代理端口，避免中途漏开。  
  这是平台安全设计里非常值得盯的一条主线。

### 9.2 fallback

- 是什么  
  fallback 是“首选方案失败时，退到兼容方案或保守方案继续跑”。
- 解决什么问题  
  现实世界里环境不统一。某些机器不支持挂 `/proc`，某些老路径还没完全迁走，所以系统需要退路。
- 不解决什么  
  fallback 不等于 fail-open。一个好的 fallback 应该退到“更兼容但仍可接受”的路径，而不是“干脆全放开”。
- 在 Codex 里常出现在哪  
  Linux 里如果 bubblewrap 预检发现挂 `/proc` 不通，会自动退到 `--no-proc`；Landlock 在今天更多是 legacy fallback；Windows setup 也会有 refresh-only 路径和错误报告路径。  
  阅读源码时，看到 `legacy`、`preflight`、`retry`、`refresh`、`fallback` 这类词，通常都在讲这类兼容性分支。

### 9.3 fail-closed vs fallback，别混

两者经常同时出现，但不是一回事:

- `fail-closed` 关心“失败后边界是不是还收着”
- `fallback` 关心“失败后有没有第二条可走的路”

理想状态是:

- 有 fallback
- fallback 仍然 fail-closed

最糟糕的是:

- 有 fallback
- 但 fallback 直接把限制放没了

---

## 10. 把这些概念串成一个小例子

假设平台要执行一条“允许写工作区，但不许碰 `.git`；默认不准外网，只允许走本地代理”的命令。

你现在应该能把它拆开看了:

1. 平台先写出 `policy`
2. helper / orchestrator 把 policy 翻译成各平台参数
3. 进程被启动
4. 文件系统读写控制落地
5. 网络策略落地
6. Linux 可能再加 seccomp 和 `no_new_privs`
7. Windows 可能再换 restricted token、capability SID、ACL、防火墙规则、private desktop

也就是说，这不是“一个 sandbox 功能”，而是“多个机制一起把边界拼出来”。

---

## 11. 读 Codex 文档时最常见的误解

- “只要有 policy，安全边界就已经存在。”  
  不对，policy 只是描述，enforcement 才是真正执行。

- “只要进程被放进 sandbox，就什么都不能做。”  
  不对，sandbox 通常是精细放行，不是全禁。

- “namespace 就是容器。”  
  不对，namespace 是容器常用的一部分，不等于完整容器。

- “seccomp 能解决一切。”  
  不对，它主要拦 syscall，不会替你定义目录树权限。

- “Windows 的 token 就等于 Linux 的 namespace。”  
  不对，它们都在收边界，但对象和语义完全不同。

- “允许 loopback 就等于允许联网。”  
  不对，loopback 只是回本机。

- “fallback 就是不安全。”  
  也不一定。关键看 fallback 是不是仍然 fail-closed。

---

## 12. 这些概念在 Codex 里通常对应哪里

下面这张表只做“对路感”，不是让你现在就去背源码。

| 概念 | 在 Codex 里常看到的落点 |
| --- | --- |
| process | `core/src/spawn.rs`、`windows-sandbox-rs/src/process.rs` |
| thread | Linux `apply_sandbox_policy_to_current_thread` 相关逻辑 |
| system call | Linux `seccomp` 规则里对 `connect`/`socket`/`ptrace` 等 syscall 的限制 |
| privilege boundary | Linux `PR_SET_NO_NEW_PRIVS`，Windows restricted token |
| 文件系统读写控制 | `protocol` 层策略 + macOS Seatbelt / Linux bubblewrap / Windows ACL |
| policy vs enforcement | `protocol` 层与各平台实现层之间的分工 |
| namespace | Linux bubblewrap 的 `--unshare-user` / `--unshare-pid` / `--unshare-net` |
| seccomp | Linux `landlock.rs` 中的 seccomp 安装 |
| `no_new_privs` | Linux `prctl(PR_SET_NO_NEW_PRIVS, ...)` |
| MAC vs DAC | macOS/Linux 强制型边界 vs Windows ACL/DACL 这类授权味道 |
| ACL | `windows-sandbox-rs/src/acl.rs` |
| token | `windows-sandbox-rs/src/token.rs` |
| SID / capability SID | `windows-sandbox-rs/src/token.rs`、`cap.rs` |
| firewall rule | `windows-sandbox-rs/src/firewall.rs` |
| socket / Unix socket / loopback | `sandboxing/src/seatbelt.rs`、Linux seccomp 网络规则 |
| desktop / session / private desktop | `windows-sandbox-rs/src/desktop.rs`、`process.rs` |
| helper / orchestrator | Linux sandbox helper、Windows `setup_orchestrator.rs` |
| fail-closed / fallback | Linux proc mount preflight、legacy Landlock、Windows firewall 过渡逻辑 |

---

## 13. 回到 3 篇平台文档，该怎么读

### 13.1 读 Linux 文档时，你重点带着哪些概念去看

优先带着这些词去看:

- process vs thread
- system call
- namespace
- seccomp
- `no_new_privs`
- fail-closed vs fallback
- policy vs enforcement

为什么  
因为 Linux 路径最像“先搭一个新的运行视图，再在 syscall 层继续收紧”。你如果不先懂 namespace 和 seccomp，就会把整条链路看成一团参数。

建议读法  
先看“为什么要分外层 bubblewrap 和内层 seccomp”，再看 `/proc` fallback，最后再看 legacy Landlock 为什么现在只是后备路径。

### 13.2 读 macOS 文档时，你重点带着哪些概念去看

优先带着这些词去看:

- policy vs enforcement
- 文件系统读写控制
- socket vs Unix socket
- loopback
- MAC
- fail-closed

为什么  
macOS 路径的关键不在“新身份”或“新 namespace”，而在“怎么把策略翻译成 Seatbelt 能理解的规则文本”。

建议读法  
先把它看成“规则生成器”，再去看为什么它会区分 loopback 端口、本地绑定、Unix socket 白名单。

### 13.3 读 Windows 文档时，你重点带着哪些概念去看

优先带着这些词去看:

- token
- SID
- capability SID
- ACL
- firewall rule
- desktop / session / private desktop
- helper / orchestrator
- privilege boundary

为什么  
Windows 路径不像 Linux 那样主要靠 namespace，也不像 macOS 那样主要靠单一规则语言。它更像“身份、授权、网络、桌面对象”多层一起配合。

建议读法  
先抓住 `restricted token + capability SID + ACL + firewall + private desktop` 这条主线，再去看 setup orchestrator 为什么存在。

---

## 14. 如果你现在只记 6 句话

如果这篇文档读完你只想带走最少版本，就记这 6 句:

1. sandbox 不是一个开关，而是多层机制叠加。
2. policy 说“想要什么”，enforcement 说“系统怎么做出来”。
3. 进程才是主要隔离对象，线程通常不是安全边界本身。
4. Linux 重点看 namespace + seccomp + `no_new_privs`。
5. macOS 重点看 Seatbelt 如何把文件和网络策略翻译成规则。
6. Windows 重点看 token、SID、ACL、防火墙和 private desktop 的组合。

---

## 15. 二级参考：如果你想对照 Codex 源码

下面这些链接都固定到同一个 commit，适合在你读完平台文档后再回头对照。这里把源码引用放在最后，是为了让“技术概念优先，源码定位其次”。

- `SandboxPolicy` / `WritableRoot` / 分离后的 filesystem/network policy  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs>  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/permissions.rs>

- 进程启动入口  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs>

- Linux: bubblewrap、namespace、seccomp、`no_new_privs`、fallback  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/bwrap.rs>  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/linux_run_main.rs>  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/linux-sandbox/src/landlock.rs>

- macOS: Seatbelt、loopback、Unix socket  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs>

- Windows: token、capability SID、ACL、防火墙、desktop、orchestrator  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/token.rs>  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/cap.rs>  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/acl.rs>  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/firewall.rs>  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/desktop.rs>  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/process.rs>  
  <https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs>
