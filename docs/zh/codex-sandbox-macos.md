# Codex 在 macOS 上如何使用 Seatbelt

如果你对进程、权限边界、socket、loopback 这些概念还不熟，建议先读 [Codex Sandbox 技术基础入门](./codex-sandbox-foundations.md)。

> 这篇文章先讲 macOS 原生沙箱，再讲 Codex 如何把它用起来。目标不是带你背源码，而是先建立一个可靠的心智模型：macOS 沙箱到底在限制什么，为什么文件规则和网络规则经常看起来“不像一个简单开关”，以及 Codex 在这个平台上具体做了哪些映射。

## 这个平台机制解决什么问题

在命令行里运行模型生成的命令时，最现实的问题不是“它能不能做事”，而是“它最多能做到哪一步”。

如果完全不加限制，一个普通的构建命令和一个危险命令看起来都只是“启动了一个子进程”。对 macOS 来说，Seatbelt 这类原生沙箱解决的是下面这类边界问题：

- 哪些路径可以读，哪些路径可以写。
- 网络是完全放开、完全禁止，还是只允许一小部分本机通信。
- 子进程能不能继续继承这些限制。
- 即使工具链本身很复杂，也尽量让限制落在操作系统层，而不是只靠应用自己“自觉”。

你可以先把它理解成一句话：

```text
先默认拒绝，再按文件、网络、进程行为补充少量允许规则。
```

这也是为什么 macOS 沙箱很适合拿来包住“可以执行任意命令”的代理型工具。它不负责判断命令“好不好”，但它可以把进程活动压缩到一组明确边界内。

## 先补最少背景

这里有四个名词，最好先分清。

- `Seatbelt`：macOS 的沙箱机制，核心思路是按规则限制进程能力。
- `sandbox-exec`：一个启动器，用来加载沙箱 profile，再启动目标命令。
- `SBPL`：Seatbelt Profile Language，Seatbelt 的规则语言。
- `profile`：一段 SBPL 文本，本质上就是“允许什么、拒绝什么”的规则文件。

最小例子可以长这样：

```lisp
(version 1)
(deny default)
(allow file-read* (subpath "/Users/me/project"))
```

这段规则的意思不是“只允许打开一个目录项”，而是“默认拒绝，然后允许读取这个路径及其子路径”。

如果通过 `sandbox-exec` 启动，形式通常像这样：

```bash
/usr/bin/sandbox-exec \
  -p '(version 1) (deny default) (allow file-read* (subpath "/tmp"))' \
  -- /bin/cat /tmp/hello.txt
```

再往前走一步，profile 里的路径也可以参数化：

```bash
/usr/bin/sandbox-exec \
  -p '(version 1) (deny default) (allow file-read* (subpath (param "ROOT")))' \
  -DROOT=/Users/me/project \
  -- /bin/ls
```

这就是后面 Codex 很依赖的一点：规则模板可以固定，但具体路径可以在启动时再注入。

## macOS 原生机制怎么工作

### 1. 默认拒绝，然后补允许规则

Seatbelt 最常见的写法就是先来一句：

```lisp
(deny default)
```

意思是：先把默认能力收紧，后面所有 `allow` 都是在这个基础上慢慢开口子。

这和很多“应用层白名单”不一样。应用层白名单常常只是业务逻辑里的 if 判断；Seatbelt 是操作系统在资源访问层面做限制，所以命令本身就算想读、想写、想联网，也要先过 profile 这一关。

### 2. 规则不是“只有文件”

Seatbelt 不只管文件。一个 profile 往往会同时出现：

- `file-read*`、`file-write*`
- `network-outbound`、`network-inbound`、`network-bind`
- `process-exec`、`process-fork`
- `mach-lookup`
- `sysctl-read`

这很好理解。现实里的命令行工具不是只碰文件系统，它们还会：

- 启动子进程
- 查询系统信息
- 访问本地守护进程
- 读终端设备
- 和网络栈打交道

所以 profile 通常不是“一条文件规则就完事”，而是一个最小可运行环境。

### 3. 子进程会继承限制

Seatbelt 不是“只给最外层那个命令套个壳”。如果 profile 允许 `process-exec` 和 `process-fork`，子进程仍然会运行，但仍然处在同一套沙箱约束下面。

这点很重要。比如一个被沙箱包住的 shell 再去调用 `git`、`python`、`clang`，它们不是自动脱离规则，而是继续在规则里运行。

## 文件规则怎么理解

文件规则最容易讲清楚，因为它本质上就是“路径匹配 + 动作限制”。

### 1. 常见的路径匹配方式

- `literal`：只匹配这个精确路径。
- `subpath`：匹配某个目录及其后代。
- `regex`：按正则匹配路径。

例如：

```lisp
(allow file-read* (literal "/tmp/a.txt"))
(allow file-read* (subpath "/Users/me/project"))
```

第一条只允许读 `/tmp/a.txt`，第二条允许读整个项目树。

### 2. “允许一个根目录”不等于“下面一切都应该可写”

这是非常常见的误解。

很多系统都会把规则理解成：

```text
工作区可写
```

但对安全实现来说，更实际的表达通常是：

```text
工作区大部分可写
某些元数据子路径保持只读
```

比如：

```text
允许写 /Users/me/project
但保护 /Users/me/project/.git
也保护 /Users/me/project/.codex
```

这样做的目的不是“让规则显得复杂”，而是避免代理改写仓库元数据、钩子目录或自己的控制目录。

### 3. carve-out 的思路

SBPL 里常见的写法是先允许一个大范围，再用条件把某些子路径排除掉。

可以把它想成这样的伪规则：

```text
允许写 ROOT
要求目标不在 ROOT/.git
要求目标不在 ROOT/.codex
```

这种思路比“给每个允许路径写一大堆碎规则”更适合工作区场景，因为工作区里的普通源码文件通常很多，但真正敏感的往往只有少数特殊目录。

## 网络规则怎么理解

网络规则比文件规则更容易让人误会，因为“允许联网”在实际系统里很少只是单个布尔值。

### 1. outbound、inbound、bind 是三件事

把这三个概念分开会轻松很多：

- `network-outbound`：向外连。
- `network-inbound`：接受进入的连接。
- `network-bind`：在本机某个地址或端口上监听。

一个进程可以“允许本地监听”，但仍然“不允许访问公网”。这不是矛盾，而是不同方向的权限。

### 2. loopback 不等于外网

`localhost`、`127.0.0.1`、`::1` 都是 loopback，也就是“连回本机”。

如果 profile 允许：

```lisp
(allow network-outbound (remote ip "localhost:8080"))
```

它表达的是“可以访问本机 8080 端口”，不是“可以访问互联网”。

这个区别在代理场景特别重要。很多开发环境会把 HTTP 代理、调试服务、端口转发器都挂在本机 loopback 上。如果你把 loopback 误解成“外网已放开”，后面读源码就会一直看错。

### 3. Unix socket 也是网络规则的一部分

Unix socket 不是 TCP 端口，它更像“同一台机器上的本地 IPC 通道”，路径通常长这样：

```text
/var/run/example.sock
```

你可以把它理解成“通过文件路径定位的本地通信端点”。

很多数据库、代理、系统服务都用 Unix socket。所以一个看似“没有开公网”的会话，仍然可能需要访问某些本地 socket 才能正常工作。

一个典型的允许规则会像这样：

```lisp
(allow system-socket (socket-domain AF_UNIX))
(allow network-outbound (remote unix-socket (subpath (param "SOCK"))))
```

### 4. 网络规则经常和系统服务绑定在一起

在 macOS 上，正常联网不只是“开放 TCP”这么简单。进程还可能需要：

- 访问 DNS 或网络配置服务
- 查询证书和信任链
- 使用系统提供的缓存目录
- 与若干 `mach` 服务交互

所以一个“能联网”的 profile 往往还会附带一些 `mach-lookup`、`sysctl-read`、缓存目录写入等辅助规则。否则命令明明看上去被允许联网，运行时却仍然会失败。

## macOS 沙箱能做什么，不能做什么

把能力边界讲清楚，比背语法更重要。

### 它能做什么

- 限制进程可读写的路径范围。
- 限制是否能发起网络连接、监听本地端口、访问 Unix socket。
- 让子进程继承同一套限制。
- 把“默认拒绝，按需开放”落到操作系统层。

### 它不能替代什么

- 它不是虚拟机，也不是容器。进程仍然跑在当前 macOS 用户空间里。
- 它不负责判断命令意图。`rm` 和 `pytest` 在它看来都是进程行为，区别只在访问资源时是否被允许。
- 它不自动回滚文件改动。一个被允许写入的文件，写坏了就是真的写坏了。
- 它不等于 TCC、SIP、签名、公证这些机制。那些是别的系统层次。
- 它不解决 CPU、内存、时间消耗这类所有资源问题。Seatbelt 的强项是“访问控制”，不是完整资源调度。

### 一个很重要的现实判断

如果某段路径已经被允许读取，那么机密是否泄露，主要就不再是 Seatbelt 的问题，而是“你到底允许读了什么”。
如果某段路径已经被允许写入，那么完整性风险也主要取决于“你到底把哪些目录设成了可写”。

所以真正有用的经验通常不是“沙箱有没有开”，而是“规则边界是不是足够小、足够明确”。

## Codex 在 macOS 上怎么应用这些机制

理解完原生机制，再看 Codex 的实现会顺很多。

### 1. Codex 先描述意图，再翻译成 Seatbelt

Codex 不是一开始就直接拼 SBPL。它先在协议层描述更抽象的策略，例如：

- 完全不限制
- 只读
- 工作区可写
- 已经处在外部沙箱中

然后再把这类“意图层策略”投影成 macOS 需要的文件规则和网络规则。

可以从这里回看定义：

- [`SandboxPolicy`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L953)
- [`FileSystemSandboxPolicy`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/permissions.rs#L136-L341)

### 2. 在 macOS 路径里，Codex 把文件和网络拆开处理

真正进入 Seatbelt 之前，Codex 会分别准备：

- 文件系统策略
- 网络策略

在 macOS 启动路径里，这两部分会被送到 Seatbelt 参数生成逻辑，再由 `/usr/bin/sandbox-exec` 启动目标命令：

- [`spawn_command_under_seatbelt`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/seatbelt.rs#L1-L35)
- [`create_seatbelt_command_args_for_policies`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L292-L421)

这个设计很像本文前面讲的思路：

```text
意图层 policy
  -> 文件规则 / 网络规则
  -> SBPL profile + -D 参数
  -> /usr/bin/sandbox-exec -- command
```

### 3. Codex 固定使用 `/usr/bin/sandbox-exec`

这不是一个无关紧要的细节。

Codex 在实现里明确固定了 `sandbox-exec` 的绝对路径，而不是去 `PATH` 里找同名程序。这样做是为了减少“PATH 被劫持，结果启动了错误二进制”的风险。

对应实现：

- [`MACOS_PATH_TO_SEATBELT_EXECUTABLE`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L18-L27)

### 4. Codex 的 profile 不是手写一整块，而是分段拼装

Codex 在 macOS 上会把 profile 拆成几部分再拼起来：

- 基础 profile
- 文件读规则
- 文件写规则
- 网络规则
- 必要时附加的平台默认只读规则

相关实现和模板在这里：

- [基础 profile](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt_base_policy.sbpl)
- [网络附加规则](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt_network_policy.sbpl)
- [平台默认只读规则](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/restricted_read_only_platform_defaults.sbpl)

这也是为什么你在源码里看到的不是“一坨超长字符串”，而是“模板 + 计算出来的规则段 + `-DKEY=VALUE` 参数”。

### 5. 文件侧最关键的策略是：工作区可写，但保留敏感子路径只读

Codex 不会把“工作区可写”粗暴实现成“整棵树毫无保留地可写”。
它会为 writable root 自动补上一些默认保护子路径，例如：

- `.git`
- `.agents`
- `.codex`

相关逻辑在：

- [`default_read_only_subpaths_for_writable_root`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/permissions.rs#L1098-L1157)

这正好对应前面讲的 carve-out 思路。也就是说，Codex 对 macOS 的利用方式不是“想办法把 Seatbelt 用得更复杂”，而是“把工作区语义翻译成 Seatbelt 能表达的路径规则”。

### 6. 网络侧最关键的策略是：尽量精确到 loopback 端口和 Unix socket

如果会话需要代理或本地 IPC，Codex 不会简单地把网络全开，而是会尽量推导：

- 代理是否指向 loopback
- loopback 的端口是多少
- 是否允许本地监听
- 哪些 Unix socket 路径应该被放行

相关实现集中在这里：

- [`proxy_loopback_ports_from_env`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L34-L67)
- [`unix_socket_policy`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L160-L197)
- [`dynamic_network_policy_for_network`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L205-L256)

一个符合本文心智模型的理解方式是：

```text
如果只需要本机代理，就只尽量开放本机代理所需的那一小圈通信。
```

### 7. Codex 还会通过环境变量标记“这个进程在沙箱里”

被沙箱启动的子进程会带上：

- `CODEX_SANDBOX=seatbelt`
- 在网络被禁用时再加 `CODEX_SANDBOX_NETWORK_DISABLED=1`

对应实现：

- [`CODEX_SANDBOX_ENV_VAR` 和 `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/spawn.rs#L8-L23)

这不是 Seatbelt 本身的要求，而是 Codex 给子进程和测试逻辑补充的上下文信号。

## 常见误解

### 误解 1：沙箱一开，就等于“完全安全”

不对。沙箱只能约束被允许访问的边界，不能替你定义正确边界。

如果你把整个家目录都设成可读，那泄露风险依旧很大。
如果你把关键工程目录都设成可写，那破坏风险也依旧很大。

### 误解 2：`allow_local_binding` 就是“允许联网”

不对。它更接近“允许在本机回环地址上监听和通信”。

它和“是否能访问公网”不是一回事。

### 误解 3：loopback 就是网络已经放开

不对。`localhost` 只代表本机。
允许 `localhost:3000`，并不代表允许访问 `example.com:443`。

### 误解 4：Unix socket 不算网络

在编程语义上它确实不像 TCP，但在 Seatbelt 规则里，它就是需要单独表达的一类通信能力。
如果本地代理或服务依赖 Unix socket，不放行它，一样会失败。

### 误解 5：工作区可写，就应该允许改 `.git`

恰恰相反。对代理系统来说，`.git`、`.codex`、`.agents` 这类目录通常正是最值得默认保护的地方。

### 误解 6：Seatbelt 就是容器

不对。Seatbelt 是 macOS 的访问控制机制，不是完整容器运行时。
它不会给你一个新的内核，也不会自动隔离所有系统资源。

## 想顺着源码回读，可以按这个顺序

如果你现在再去看 Codex 源码，建议按“从概念到落地”的顺序读，而不是一上来就钻进最长的 Seatbelt 文件。

1. 先看 [`SandboxPolicy`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/protocol.rs#L787-L953)，先弄清 Codex 想表达哪些高层策略。
2. 再看 [`FileSystemSandboxPolicy`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/protocol/src/permissions.rs#L136-L341)，理解“工作区可写但保留只读 carve-out”是怎么建模的。
3. 然后看 [`spawn_command_under_seatbelt`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/core/src/seatbelt.rs#L1-L35)，确认 macOS 入口怎样把这些策略送进 Seatbelt。
4. 接着看 [`create_seatbelt_command_args_for_policies`](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt.rs#L292-L421)，看 profile 是怎么拼出来的。
5. 最后再回头读 [基础 profile](https://github.com/openai/codex/blob/47a9e2e084e21542821ab65aae91f2bd6bf17c07/codex-rs/sandboxing/src/seatbelt_base_policy.sbpl)，这时你会更容易分清哪些规则是在“让系统最小可运行”，哪些规则是在“给工作区和网络开口子”。

如果读到一半开始混乱，回到本文最前面的主线就够了：

```text
先定义边界
再把边界翻译成 Seatbelt 规则
最后用 sandbox-exec 带着这些规则启动命令
```

只要这条线没丢，后面的源码细节就会越来越容易读。
