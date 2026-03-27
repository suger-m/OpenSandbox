# 从零看懂 Docker、系统知识与 Kubernetes

副标题：为阅读 OpenSandbox 运行时代码做准备

这篇教程面向这样一类读者：你能看一点 Python 或后端代码，也知道接口层大概在做什么，但一进入 `docker.py`、`kubernetes_service.py` 这种运行时代码，就会突然觉得每一行都像在操作某种陌生系统。问题通常不在于语法，而在于你还没有建立起“代码背后对应的真实动作”。

OpenSandbox 这一层代码，做的不是普通业务规则，而是在编排运行环境：拉镜像、启动容器或 Pod、改启动链路、注入文件和环境变量、限制资源、开放端口、接管网络、加 sidecar、在到期后回收。只要先把这些概念对上，再回头读源码，理解成本会明显下降。

## 1. 为什么运行时代码难读

普通业务代码经常在做请求解析、参数校验、查数据库、组装返回值。运行时代码处理的却是另一套问题：镜像从哪里来、容器为什么能启动、一个端口为什么能从外部访问、为什么有的进程会自动结束、为什么平台要在用户命令前面塞一层 bootstrap、为什么到了 Kubernetes 之后一个 sandbox 不再只是一个 container。

所以你卡住时，往往不是卡在 Python，而是卡在这句更底层的话上：“这段代码在现实里到底对应什么系统动作？”这篇文档的目标，就是先把这些动作讲清楚，而且每个关键概念都给你一条命令和一个最小 Python 例子，方便你边看边验证。

## 2. 怎么使用这篇教程

你不需要一次读完整篇。更推荐的方式是：

1. 先读某个概念的“是什么”和“为什么有”。
2. 直接复制那条 `docker` 或 `kubectl` 命令跑一下。
3. 再看 Python 小例子，理解“如果我要在程序里做同样的事，大概会长什么样”。
4. 最后看“在 OpenSandbox 里通常对应什么”，把它和源码重新连起来。

运行这些例子前，建议你至少准备好下面这些工具：

1. Docker：第 3 章和第 4 章的命令需要本机能正常运行 `docker`。
2. Python 3：所有 Python 小例子都默认可以用 `python` 直接执行，且只依赖标准库。
3. kubectl + 一个可用的 Kubernetes 集群：第 5 章默认面向 `kind`、`minikube`、Docker Desktop Kubernetes，或者任意你能访问的集群。

命令书写还有一个小约定：文中的 shell 示例默认按 Bash 风格书写。如果你在 Windows PowerShell 里跟着跑，最容易遇到的差异是：

1. 反斜杠续行 `\` 通常要改成一行，或者改成 PowerShell 自己的续行方式。
2. `mkdir -p`、`cat`、`${PWD}`、单引号 shell 片段这类写法，需要换成 PowerShell 等价形式。
3. `curl` 在 PowerShell 里经常会映射到 `Invoke-WebRequest`，如果你想要和 Bash 一样的行为，直接用 `curl.exe` 往往更省心。

如果你只是想验证概念，最简单的办法通常是把多行命令压成一行，再按你当前 shell 的语法做最少量调整。

如果你本机只有 Docker，没有本地 Kubernetes 集群，也没关系。第 3 章和第 4 章可以直接验证；第 5 章可以先当成“看得懂结构和命令意图”的阅读材料。

运行 Kubernetes 例子时，再多记一个约定：每张卡片的 `kubectl` 命令都会引用一个 `demo-*.yaml` 文件名，而下面的“最小 YAML”就是这个文件的内容。也就是说，你要先把那段 YAML 保存成同名文件，再执行上面的 `kubectl apply -f demo-*.yaml`。如果你只是想理解概念，不想真的跑，也可以只看命令和 YAML 的对应关系。

<!--
概念卡片模板
示例标题：x.x 概念名
是什么：
为什么有：
Docker / kubectl 命令：
```bash
...
```
Python 小例子：
```python
...
```
你会看到什么：
在 OpenSandbox 里通常对应什么：
-->

## 3. Docker 基础

这一章先解决你读 `server/opensandbox_server/services/docker.py` 时最容易反复碰到的词。你不用先学完整的 Docker，只要先把这几组概念和行为建立起来就够了。

### 3.1 image
是什么：
image 是容器的“模板”或“快照”。它里面装着程序、依赖和默认配置，但它自己不会跑起来。

为什么有：
有了 image，才能把同一套环境复制很多次。你先准备好模板，后面每次启动容器都从这个模板生成，结果更稳定。

Docker 命令：
```bash
docker image pull alpine:3.20
docker image inspect alpine:3.20 --format '{{.Id}}'
```

Python 小例子：
```python
import subprocess

subprocess.run(["docker", "image", "pull", "alpine:3.20"], check=True)
result = subprocess.run(
    ["docker", "image", "inspect", "alpine:3.20", "--format", "{{.Id}}"],
    check=True,
    capture_output=True,
    text=True,
)
print(result.stdout.strip())
```

你会看到什么：
第一次会下载 `alpine:3.20`，之后本地就有了。`inspect` 会输出一个 `sha256:...` 的 image ID，说明这个模板已经在本地可用了。

在 OpenSandbox 里通常对应什么：
`server/opensandbox_server/services/docker.py` 里会先检查 image 是否存在，不存在就先拉取，再继续创建 sandbox。

### 3.2 container
是什么：
container 是从 image 启动出来的“实例”。它真的会运行，有自己的进程、文件状态和生命周期。

为什么有：
同一个 image 可以启动很多个 container，每个都互相独立。这样一份模板就能支撑很多次运行，而且彼此不会混在一起。

Docker 命令：
```bash
docker run --name primer-container alpine:3.20 echo hello
docker container inspect primer-container --format '{{.Config.Image}} {{.Path}} {{json .Config.Cmd}}'
docker rm primer-container
```

Python 小例子：
```python
import subprocess

subprocess.run(
    ["docker", "run", "--name", "primer-container", "alpine:3.20", "echo", "hello"],
    check=True,
)

result = subprocess.run(
    [
        "docker",
        "container",
        "inspect",
        "primer-container",
        "--format",
        "{{.Config.Image}} {{.Path}} {{json .Config.Cmd}}",
    ],
    check=True,
    capture_output=True,
    text=True,
)
print(result.stdout.strip())

subprocess.run(["docker", "rm", "primer-container"], check=True)
```

你会看到什么：
终端先打印 `hello`，然后这个容器会退出，但它还会留在本地，直到你删除它。`inspect` 能看到它是从哪个 image 来的，以及实际执行了什么命令。

在 OpenSandbox 里通常对应什么：
`server/opensandbox_server/services/docker.py` 里会把 image、`entrypoint`、`command`、`labels` 和 `environment` 组装好，再 `create_container` 和 `start`。

### 3.3 entrypoint
是什么：
`entrypoint` 是容器启动时先执行的程序或脚本。你可以把它理解成“入口壳子”：容器一起来，先跑它，再把 `command` 交给它处理。Docker 里如果你显式写了 `--entrypoint`，就是在覆盖镜像原本的入口。

为什么有：
它的作用是让平台把自己的启动逻辑放在用户程序前面。比如先做初始化、先启动守护进程、先挂代理、先做权限或网络准备，再把控制权交给真正的业务程序。这样业务镜像不用知道平台内部细节。

Docker 命令：
```bash
docker run --rm \
  --entrypoint /bin/sh \
  alpine:3.20 \
  -c 'printf "entrypoint=%s\n" "$0"; printf "command arg1=%s\n" "$1"; printf "command arg2=%s\n" "$2"' \
  shell-as-argv0 hello world
```

Python 小例子：
```python
from pathlib import Path
import subprocess
import sys
import tempfile

with tempfile.TemporaryDirectory() as tmp:
    script = Path(tmp) / "show_argv.py"
    script.write_text(
        "import sys\n"
        "print('argv:', sys.argv)\n",
        encoding="utf-8",
    )

    entrypoint = [sys.executable, str(script)]
    command = ["hello", "world"]

    print("final argv:", entrypoint + command)
    subprocess.run(entrypoint + command, check=True)
```

你会看到什么：
输出里会先看到入口程序本身被执行，然后它拿到了后面的参数。也就是说，`entrypoint` 先启动，`command` 是它收到的参数之一。

在 OpenSandbox 里通常对应什么：
`server/opensandbox_server/services/docker.py` 里真正传给 Docker 的 `entrypoint` 不是用户程序，而是 `/opt/opensandbox/bootstrap.sh`。这个脚本会先拉起 `execd`，再把后续流程交出去，所以 OpenSandbox 里“平台先接管启动，再交还控制权”主要就是靠 `entrypoint` 完成的。

### 3.4 command
是什么：
`command` 是交给 `entrypoint` 的内容。它通常是参数列表，也可以在没有自定义 `entrypoint` 的时候直接就是要执行的程序。更准确地说，`entrypoint` 决定“谁来启动”，`command` 决定“启动后具体做什么”。

为什么有：
把两者拆开以后，平台就能复用同一个入口壳子，而只替换真正要跑的内容。用户换 `command`，就能跑不同任务；平台也能在 wrapper 里先做准备，然后把用户请求的命令原封不动地交下去。

Docker 命令：
```bash
docker run --rm \
  python:3.11-alpine \
  python -c 'import sys; print("argv =", sys.argv)' \
  alpha beta
```

Python 小例子：
```python
from pathlib import Path
import subprocess
import sys
import tempfile

with tempfile.TemporaryDirectory() as tmp:
    script = Path(tmp) / "show_command.py"
    script.write_text(
        "import sys\n"
        "print('command args:', sys.argv[1:])\n",
        encoding="utf-8",
    )

    entrypoint = [sys.executable, str(script)]
    command = ["alpha", "beta"]

    subprocess.run(entrypoint + command, check=True)
```

你会看到什么：
输出会显示 `alpha`、`beta` 被当成了程序收到的参数。这里最关键的是：`command` 本身不是另一个入口，它是交给入口程序的“要执行什么”。

在 OpenSandbox 里通常对应什么：
`server/opensandbox_server/services/docker.py` 里，用户请求中的启动列表会被整理成 `bootstrap_command`，再作为 Docker 的 `command` 传进去。随后 `components/execd/bootstrap.sh` 会先启动 `execd`，如果有 `BOOTSTRAP_CMD` 或 `-c`，就走 shell 方式执行；否则就直接 `exec "$@"` 把这段 `command` 交给最终程序。也就是说，在 OpenSandbox 里 `command` 就是平台真正要“交出去”的那部分用户启动内容。

### 3.5 env
是什么：
`env` 是容器启动时注入进去的环境变量。它会出现在容器内进程的环境里，像 `APP_ENV=prod`、`PORT=8080` 这种键值对，程序一启动就能读到。

为什么有：
它适合放“这个进程运行时需要知道，但不想写死在镜像里”的配置，比如端口、环境名、开关、token 的来源提示等。这样同一个镜像可以在不同环境里复用。

Docker 命令：
```bash
docker run --rm -e DEMO_NAME=OpenSandbox alpine:3.20 sh -c 'printf "DEMO_NAME=%s\n" "$DEMO_NAME"'
```

Python 小例子：
```python
import os
import subprocess

env = os.environ.copy()
env["DEMO_NAME"] = "OpenSandbox"

subprocess.run(
    [
        "docker",
        "run",
        "--rm",
        "-e",
        "DEMO_NAME",
        "alpine:3.20",
        "sh",
        "-c",
        'printf "DEMO_NAME=%s\\n" "$DEMO_NAME"',
    ],
    check=True,
    env=env,
)
```

你会看到什么：
容器里会打印出 `DEMO_NAME=OpenSandbox`。这说明 `env` 是“给进程看的”，而不是写在镜像里的固定内容。

在 OpenSandbox 里通常对应什么：
`request.env` 会被直接注入到 sandbox 容器里；有些运行时配置、策略参数或辅助组件的开关也会通过环境变量传进去。

### 3.6 labels
是什么：
`labels` 是挂在 Docker 对象上的元数据标签。它不是容器进程的环境变量，容器里的程序一般不会通过 `echo $xxx` 直接读到它。更像是平台自己贴在容器外面的“检索标签”。

为什么有：
平台需要用它来做查询、关联和回收，比如标记某个容器属于哪个 sandbox、什么时候过期、是不是手动清理、是不是 sidecar、对应哪个挂载信息。这样就算程序停了，平台也还能靠标签找到它。

Docker 命令：
```bash
docker run -d --name label-demo \
  --label sandbox.id=demo \
  --label runtime.type=docker \
  alpine:3.20 sleep 300

docker inspect label-demo --format '{{ json .Config.Labels }}'
docker rm -f label-demo
```

Python 小例子：
```python
import json
import subprocess

subprocess.run(
    [
        "docker",
        "run",
        "-d",
        "--name",
        "label-demo",
        "--label",
        "sandbox.id=demo",
        "--label",
        "runtime.type=docker",
        "alpine:3.20",
        "sleep",
        "300",
    ],
    check=True,
)

try:
    labels = subprocess.check_output(
        [
            "docker",
            "inspect",
            "label-demo",
            "--format",
            "{{ json .Config.Labels }}",
        ],
        text=True,
    )
    print(json.loads(labels))
finally:
    subprocess.run(["docker", "rm", "-f", "label-demo"], check=True)
```

你会看到什么：
`docker inspect` 会显示一份标签字典，比如 `sandbox.id` 和 `runtime.type`。它们不会像环境变量那样出现在容器进程里，而是保存在 Docker 元数据中。

在 OpenSandbox 里通常对应什么：
`request.metadata` 会被写成容器 labels，另外还会附上 sandbox id、过期时间、手动清理标记、挂载引用、sidecar 关联等运行时元数据，方便查询、路由和清理。

### 3.7 volumes
是什么：
`volume` 可以理解成“把容器外面的存储接进容器里”。最常见的两种是：把宿主机目录挂进去，或者把 Docker 自己管理的数据卷挂进去。这样容器里的程序读写文件时，就不是只写在容器自己的临时文件系统里了。

为什么有：
如果没有挂载，容器删掉后，里面新写的文件通常也就没了。挂载最重要的两个价值是持久化数据，以及让宿主机和容器共享文件。

Docker 命令：
```bash
docker run --rm \
  -v "${PWD}:/workspace" \
  alpine:3.20 \
  sh -lc 'echo "hello from container" > /workspace/hello-from-container.txt && ls /workspace && cat /workspace/hello-from-container.txt'

python -c "from pathlib import Path; print(Path('hello-from-container.txt').read_text(encoding='utf-8').strip())"
```

Python 小例子：
```python
import subprocess
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    host_dir = Path(tmp)

    subprocess.run(
        [
            "docker", "run", "--rm",
            "-v", f"{host_dir}:/data",
            "alpine:3.20",
            "sh", "-lc",
            'echo "hello from container" > /data/hello.txt',
        ],
        check=True,
    )

    print((host_dir / "hello.txt").read_text(encoding="utf-8").strip())
```

你会看到什么：
容器里写到 `/data/hello.txt` 的内容，会直接出现在宿主机目录或临时目录里。也就是说，容器和宿主机“看的是同一份数据”。

在 OpenSandbox 里通常对应什么：
OpenSandbox 会把请求里的 `volumes` 转成 Docker 的 `binds`。常见来源包括宿主机路径、Docker named volume，以及运行时准备好的 OSSFS 挂载路径。

### 3.8 port binding
是什么：
`port binding` 就是把“容器里的端口”映射到“宿主机的端口”。最常见的写法是 `宿主机端口:容器端口`。比如 `-p 18080:8000` 的意思是：访问宿主机 `18080`，实际会转发到容器里的 `8000`。

为什么有：
容器里的服务默认只在容器网络里可见。你如果想从本机浏览器、`curl`，或者别的程序访问它，就通常要做端口映射。

Docker 命令：
```bash
docker run --rm -d \
  --name port-demo \
  -p 18080:8000 \
  python:3.11-alpine \
  python -m http.server 8000

docker port port-demo
python -c "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:18080').status)"

docker rm -f port-demo
```

Python 小例子：
```python
import subprocess
import time
import urllib.request

subprocess.run(
    [
        "docker", "run", "--rm", "-d",
        "--name", "port-demo-py",
        "-p", "18080:8000",
        "python:3.11-alpine",
        "python", "-m", "http.server", "8000",
    ],
    check=True,
)

try:
    time.sleep(2)
    data = urllib.request.urlopen("http://127.0.0.1:18080").read(120)
    print(data.decode("utf-8", errors="ignore"))
finally:
    subprocess.run(["docker", "rm", "-f", "port-demo-py"], check=False)
```

你会看到什么：
虽然 HTTP 服务实际跑在容器的 `8000` 端口上，但你在宿主机访问的是 `127.0.0.1:18080`。这就是端口映射生效了。

在 OpenSandbox 里通常对应什么：
在非 `host` 网络下，OpenSandbox 通常会给沙箱容器的 `44772` 和 `8080` 分配宿主机端口，并把这个映射结果记下来，后面生成 endpoint 时会用到。

### 3.9 network mode
是什么：
`network mode` 决定容器“用什么方式接入网络”。读 OpenSandbox 源码时，先记住两个最重要的就够了：`bridge` 是默认模式，容器有自己的网络空间，通常要配合 `-p` 才方便从宿主机访问；`host` 是容器直接共享宿主机网络，通常不需要 `-p`。

为什么有：
不同模式是在“隔离性”和“访问方式简单程度”之间做取舍。`bridge` 更像“容器有自己的小网络”，更常见，也更容易理解隔离；`host` 更像“容器直接站在宿主机网络上”，访问简单，但端口更容易冲突。

Docker 命令：
```bash
# bridge（默认）
docker run --rm -d \
  --name net-bridge \
  -p 18081:8000 \
  python:3.11-alpine \
  python -m http.server 8000

python -c "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:18081').status)"
docker rm -f net-bridge

# host（通常在 Linux 上更容易直接复现；Docker Desktop 上行为可能不同）
docker run --rm -d \
  --name net-host \
  --network host \
  python:3.11-alpine \
  python -m http.server 8000

python -c "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:8000').status)"
docker rm -f net-host
```

Python 小例子：
```python
import subprocess
import time
import urllib.request

# bridge
subprocess.run(
    [
        "docker", "run", "--rm", "-d",
        "--name", "net-bridge-py",
        "-p", "18081:8000",
        "python:3.11-alpine",
        "python", "-m", "http.server", "8000",
    ],
    check=True,
)

try:
    time.sleep(2)
    print("bridge:", urllib.request.urlopen("http://127.0.0.1:18081").status)
finally:
    subprocess.run(["docker", "rm", "-f", "net-bridge-py"], check=False)

# host
result = subprocess.run(
    [
        "docker", "run", "--rm", "-d",
        "--name", "net-host-py",
        "--network", "host",
        "python:3.11-alpine",
        "python", "-m", "http.server", "8000",
    ],
    capture_output=True,
    text=True,
)

if result.returncode == 0:
    try:
        time.sleep(2)
        print("host:", urllib.request.urlopen("http://127.0.0.1:8000").status)
    finally:
        subprocess.run(["docker", "rm", "-f", "net-host-py"], check=False)
else:
    print("host 模式在当前环境没有直接跑通，这在 Docker Desktop 上并不罕见。")
```

你会看到什么：
在 `bridge` 模式下，你通常是访问映射后的宿主机端口，比如 `18081`。在 `host` 模式下，容器服务更像直接出现在宿主机网络里，比如直接访问 `8000`。

在 OpenSandbox 里通常对应什么：
OpenSandbox 会看 `[docker] network_mode`。如果是 `host`，endpoint 更接近直接返回宿主机端口；如果是 `bridge`，通常会先做 host port 映射，再把可访问地址返回出去。

## 4. 一点系统知识

这一章补的是“容器和平台为什么会这样设计”的底层直觉。很多时候你不是不懂 Docker，而是不知道进程、超时和隔离这些概念为什么会反复出现在运行时代码里。

### 4.1 进程启动
是什么：
进程启动可以理解成“谁先成为入口进程，谁再把后续子进程拉起来”。在容器里，这通常就是 `PID 1`、entrypoint、以及它后面派生出来的业务进程之间的关系。

为什么有：
因为容器不是直接“有个镜像就会自己运行”，而是要靠一个明确的入口把辅助逻辑和业务逻辑串起来。常见场景包括先启动代理、日志器、执行守护进程，再把控制权交给真正的用户命令。

Docker 命令：
```bash
docker run --rm alpine:3.20 sh -c '
  echo "PID1 is $$"
  sleep 2 &
  echo "child pid is $!"
  wait
  echo "all done"
'
```

Python 小例子：
```python
import os
import subprocess
import sys

print("parent pid:", os.getpid())
child = subprocess.Popen(
    [sys.executable, "-c", "import os,time; print('child pid:', os.getpid()); time.sleep(1)"]
)
print("spawned child:", child.pid)
child.wait()
print("child exited")
```

你会看到什么：
Docker 例子里，shell 自己就是这个容器里的入口进程，它再拉起一个 `sleep` 子进程。Python 例子里也一样：先有父进程，再有它启动的子进程。

在 OpenSandbox 里通常对应什么：
Docker 路径会先把 `execd` 注入容器，再把入口改成 bootstrap；bootstrap 先启动 `execd`，再 `exec` 用户 entrypoint。K8s 路径则先用 init container 把 `execd` 和 `bootstrap.sh` 放进共享卷，再让主容器用包装后的 command 启动。

### 4.2 前台和后台
是什么：
前台执行是“调用方一直等到它结束”，后台执行是“先把任务放出去，自己先返回”。两者的差别不在于命令本身，而在于调用方要不要阻塞等待。

为什么有：
因为有些命令是短作业，适合直接等结果；有些命令是长作业，比如启动 Web 服务、训练任务、持续监听进程，更适合先拿到一个会话或进程标识，之后再查状态和日志。

Docker 命令：
```bash
docker run --rm alpine:3.20 sh -c '
  echo "foreground start"
  sleep 1
  echo "foreground done"

  sleep 3 &
  echo "background pid=$!"
  echo "shell continues immediately"
  wait $!
  echo "background done"
'
```

Python 小例子：
```python
import subprocess
import sys
import time

print("foreground:")
subprocess.run([sys.executable, "-c", "import time; time.sleep(1); print('done')"], check=True)

print("background:")
proc = subprocess.Popen([sys.executable, "-c", "import time; time.sleep(2); print('done later')"])
print("returned immediately, pid =", proc.pid)
time.sleep(0.5)
print("still running:", proc.poll() is None)
proc.wait()
print("exit code:", proc.returncode)
```

你会看到什么：
前台部分会卡住，直到命令跑完才继续。后台部分会立刻拿到一个 PID，然后主程序可以先做别的事，最后再决定是否等待它结束。

在 OpenSandbox 里通常对应什么：
前台命令会流式返回 stdout/stderr，并在 `cmd.Wait()` 后结束请求。后台命令会先返回会话 ID，后续通过状态接口和日志接口继续观察。

### 4.3 超时
是什么：
超时是“给某个操作设一个截止时间，到点就取消或回收”。它不是“这个操作慢一点”，而是“慢到这个界限就不再等”。

为什么有：
没有超时，挂死的命令、永远不退出的服务、卡住的创建流程都会一直占着资源。超时让系统能自动止损，并把“还没结束”和“已经失败”区分开。

Docker 命令：
```bash
cid="$(docker run -d --rm alpine:3.20 sh -c 'trap "echo TERM received; exit 124" TERM; sleep 30 & wait')"
sleep 2
docker stop --time 1 "$cid"
```

Python 小例子：
```python
import subprocess
import sys

try:
    subprocess.run(
        [sys.executable, "-c", "import time; time.sleep(5)"],
        timeout=1,
        check=True,
    )
except subprocess.TimeoutExpired as exc:
    print("command timeout after", exc.timeout, "second(s)")

print("注意：命令超时是单次执行的截止时间，不等于整个 sandbox 的 TTL。")
```

你会看到什么：
Docker 例子里，容器里的任务本来会睡很久，但外部在几秒后就把它停掉了。Python 例子里，子进程在 1 秒左右就会抛出 `TimeoutExpired`。

在 OpenSandbox 里通常对应什么：
要分清两件事：`sandbox TTL` 是整个沙箱的生存时间，创建时会算成 `expires_at` 并在到点后回收；`command timeout` 是单次命令执行的截止时间，`execd` 会把它转成 `context.WithTimeout`。K8s 还额外有“等待沙箱 Ready”的创建超时。

### 4.4 资源限制
是什么：
资源限制就是给进程或容器划边界，比如最多多少内存、多少 CPU、多少进程数。它的目标不是让程序更快，而是防止单个任务把整台机器拖垮。

为什么有：
沙箱里跑的通常是不完全可信、也不完全可预期的代码。没有限制，死循环、内存爆炸、fork 炸弹、异常并发都会直接影响宿主机和其他沙箱。

Docker 命令：
```bash
docker run --rm \
  --cpus 0.5 \
  --memory 128m \
  --pids-limit 64 \
  alpine:3.20 sh -c '
    echo "memory:"
    cat /sys/fs/cgroup/memory.max 2>/dev/null || cat /sys/fs/cgroup/memory/memory.limit_in_bytes
    echo "cpu:"
    cat /sys/fs/cgroup/cpu.max 2>/dev/null || true
    echo "pids:"
    cat /sys/fs/cgroup/pids.max 2>/dev/null || true
  '
```

Python 小例子：
```python
try:
    import resource
except ImportError:
    print("resource 模块只在类 Unix 平台可用")
else:
    limit = 256 * 1024 * 1024  # 256 MiB
    resource.setrlimit(resource.RLIMIT_AS, (limit, limit))
    print("RLIMIT_AS:", resource.getrlimit(resource.RLIMIT_AS))
```

你会看到什么：
Docker 例子会读出 cgroup 里的 CPU、内存、PID 上限。Python 例子在支持 `resource` 的平台上会打印新的地址空间限制，这更像“单进程级别”的资源边界。

在 OpenSandbox 里通常对应什么：
Docker 路径会把请求里的 `cpu`、`memory` 解析成 `nano_cpus`、`mem_limit`，并可叠加 `pids_limit` 和安全选项。K8s 路径会把资源写进 Pod `resources.limits/requests`，主容器和 `execd` init container 还能分别配置。

### 4.5 网络隔离
是什么：
网络隔离就是决定“这个进程在哪个网络命名空间里、能被谁访问、能访问谁”。最常见的边界包括只在容器内可见、只在某个 Docker network 内可见、或者对宿主机/外网可见。

为什么有：
如果每个沙箱的所有端口都直接暴露到宿主机，既不安全，也很难扩展。隔离后，默认情况下端口只在沙箱内部或受控网络里可见，需要的时候再通过代理或映射暴露出去。

Docker 命令：
```bash
docker network create primer-net
docker run -d --rm --name primer-web --network primer-net python:3.12-alpine sh -c 'python -m http.server 8000'
docker run --rm --network primer-net alpine:3.20 sh -c 'wget -qO- http://primer-web:8000 | head -n 1'
docker rm -f primer-web
docker network rm primer-net
```

Python 小例子：
```python
import socket

srv = socket.socket()
srv.bind(("127.0.0.1", 0))
srv.listen()
host, port = srv.getsockname()
print("listening on:", host, port)
print("这个地址只监听 localhost，不会自动暴露给外部网络")
srv.close()
```

你会看到什么：
Docker 例子里，第二个容器能在同一个私有网络里访问 `primer-web`，但这个服务并没有发布到宿主机端口。Python 例子里，监听地址明确绑定在 `127.0.0.1`，说明可见范围被限制在本机回环接口。

在 OpenSandbox 里通常对应什么：
Docker `bridge` 模式下，通常只对外映射 `execd` 代理端口 `44772` 和可选 `8080`，其他端口通过 `/proxy/{port}` 转发；`host` 模式则更直接。K8s 路径通常返回 Pod IP 或 gateway ingress 地址。

### 4.6 sidecar
是什么：
sidecar 是一种模式，不是某个特定产品开关。它指的是“让一个辅助容器和主容器一起部署、共享部分运行时环境，用来承载代理、网络控制、日志采集、证书刷新这类横切能力”。

为什么有：
因为有些能力不应该塞进主业务镜像里。把它们拆成 sidecar，主容器可以更专注于业务逻辑，辅助逻辑也能独立升级、替换和加固。

Docker 命令：
```bash
docker run -d --rm --name primer-app python:3.12-alpine sh -c 'python -m http.server 8000'
docker run --rm --network container:primer-app alpine:3.20 sh -c 'wget -qO- http://127.0.0.1:8000 | head -n 1'
docker rm -f primer-app
```

Python 小例子：
```python
import threading
import time

def sidecar():
    for _ in range(3):
        print("helper: collecting/controlling")
        time.sleep(0.5)

def main_app():
    for i in range(3):
        print("app: working", i)
        time.sleep(0.5)

t = threading.Thread(target=sidecar, daemon=True)
t.start()
main_app()
t.join()
```

你会看到什么：
Docker 例子里，第二个容器虽然不是应用本身，但因为共享了应用容器的 network namespace，所以可以直接访问 `127.0.0.1:8000`。Python 例子里，主逻辑和辅助逻辑并行存在，辅助逻辑没有改写主逻辑，只是在旁边提供支持。

在 OpenSandbox 里通常对应什么：
OpenSandbox 的 egress sidecar 就是这个模式的一个实例：Docker 路径里主沙箱容器会加入 sidecar 的网络命名空间，K8s 路径里则把 `egress` 容器追加进同一个 Pod。主容器负责业务进程，sidecar 负责网络控制这类横切职责。

## 5. Kubernetes 基础

如果你已经理解了 Docker，再看 Kubernetes 时最容易误解的一点是：Kubernetes 的主语常常不是“单个 container”，而是“一个 Pod 及其周边资源”。这一章不是让你学会运维 Kubernetes，而是让你先看懂源码里那些概念在说什么。

### 5.1 Pod
是什么：
Pod 是 Kubernetes 里最小的调度和运行单元。你可以先把它理解成“一个会被一起调度、一起拿到网络和卷的容器组”。如果你先学的是 Docker，那最接近的直觉是：Pod 比单个容器高一层，它通常包住一个主容器，也可以同时包住别的辅助容器。

为什么有：
Kubernetes 调度、网络、存储挂载都需要一个统一的边界，所以它不是直接调“单容器”，而是调 Pod。只要你有一个可用的本地集群，比如 `kind`、`minikube` 或 Docker Desktop Kubernetes，这一节里的例子都可以跑。

kubectl 命令：
```bash
# 先把下面的 YAML 保存为 demo-pod.yaml
kubectl apply -f demo-pod.yaml
kubectl get pod demo-pod -o wide
kubectl logs demo-pod
kubectl delete -f demo-pod.yaml
```

最小 YAML：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo pod-ready; sleep 3600"]
```

Python 小例子：
```python
import json
import subprocess

result = subprocess.run(
    ["kubectl", "get", "pod", "demo-pod", "-o", "json"],
    check=True,
    capture_output=True,
    text=True,
)
pod = json.loads(result.stdout)
print("phase:", pod["status"]["phase"])
print("podIP:", pod["status"].get("podIP"))
```

你会看到什么：
你会看到一个叫 `demo-pod` 的 Pod 进入 `Running`，并拿到一个集群内 IP。`kubectl get pod -o wide` 能看到它的节点和 Pod IP，`kubectl logs` 能看到容器启动时打印的 `pod-ready`。

在 OpenSandbox 里通常对应什么：
OpenSandbox 的 K8s 运行时最终关心的是“某个 sandbox 对应的工作负载是否已经有 Pod、是否已分配 IP、是否可访问”。

### 5.2 init container
是什么：
`init container` 是主容器启动前必须先跑完的准备容器。它适合做“先准备好文件、二进制、配置、证书、缓存”这类一次性工作。

为什么有：
这样能把“准备环境”和“真正跑业务”分开，主容器就不用自己负责启动前的杂事。只要有普通本地集群就能跑，不需要额外插件。

kubectl 命令：
```bash
# 先把下面的 YAML 保存为 demo-init.yaml
kubectl apply -f demo-init.yaml
kubectl get pod demo-init
kubectl describe pod demo-init
kubectl logs demo-init -c app
kubectl delete -f demo-init.yaml
```

最小 YAML：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-init
spec:
  volumes:
    - name: work
      emptyDir: {}
  initContainers:
    - name: prepare
      image: busybox:1.36
      command: ["sh", "-c", "echo prepared > /work/message.txt"]
      volumeMounts:
        - name: work
          mountPath: /work
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "cat /work/message.txt; sleep 3600"]
      volumeMounts:
        - name: work
          mountPath: /work
```

Python 小例子：
```python
import json
import subprocess

result = subprocess.run(
    ["kubectl", "get", "pod", "demo-init", "-o", "json"],
    check=True,
    capture_output=True,
    text=True,
)
pod = json.loads(result.stdout)
status = pod["status"]["initContainerStatuses"][0]["state"]["terminated"]
print("init exitCode:", status["exitCode"])
print("init reason:", status["reason"])
```

你会看到什么：
Pod 会先执行 `prepare`，把文件写进共享卷，然后主容器 `app` 再启动并读出 `prepared`。`describe pod` 里能看到 init container 先完成，`logs -c app` 能看到主容器读到那行文本。

在 OpenSandbox 里通常对应什么：
OpenSandbox 会先用 init container 把 `execd` 和 `bootstrap.sh` 放进共享卷，再让主容器通过它们启动。

### 5.3 Kubernetes 里的 sidecar
是什么：
`sidecar` 是和主容器一起常驻在同一个 Pod 里的辅助容器。它和主容器共享 Pod 的网络、卷，有时负责日志、代理、证书刷新、流量控制等配套能力。

为什么有：
很多辅助能力不应该塞进主程序镜像里，拆成 sidecar 更清晰，也更容易替换。普通本地集群就能跑这一节的例子；如果你要做网络代理类 sidecar，才会涉及更高权限和更复杂的集群配置。

kubectl 命令：
```bash
# 先把下面的 YAML 保存为 demo-sidecar.yaml
kubectl apply -f demo-sidecar.yaml
kubectl get pod demo-sidecar
kubectl logs demo-sidecar -c sidecar --tail=5
kubectl logs demo-sidecar -c app --tail=5
kubectl delete -f demo-sidecar.yaml
```

最小 YAML：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-sidecar
spec:
  volumes:
    - name: shared
      emptyDir: {}
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /shared/app.log; sleep 2; done"]
      volumeMounts:
        - name: shared
          mountPath: /shared
    - name: sidecar
      image: busybox:1.36
      command: ["sh", "-c", "touch /shared/app.log; tail -f /shared/app.log"]
      volumeMounts:
        - name: shared
          mountPath: /shared
```

Python 小例子：
```python
import json
import subprocess

result = subprocess.run(
    ["kubectl", "get", "pod", "demo-sidecar", "-o", "json"],
    check=True,
    capture_output=True,
    text=True,
)
pod = json.loads(result.stdout)
for item in pod["status"]["containerStatuses"]:
    print(item["name"], "ready=", item["ready"])
```

你会看到什么：
主容器不断往共享卷里的日志文件写时间戳，sidecar 持续 `tail -f` 这份文件。你会明显看到“主容器做业务，sidecar 做辅助”的分工。

在 OpenSandbox 里通常对应什么：
OpenSandbox 最常见的 sidecar 场景是可选的 `egress` 容器，用来承接网络出口控制。

### 5.4 labels 和 annotations
是什么：
`labels` 是给资源打的可筛选标签，适合做分组、选择器、归类；`annotations` 是附加说明，适合存放不参与筛选的元数据。一个常见记法是：labels 更适合“让系统挑选”，annotations 更适合“让系统记住”，但这更像常见分工，不是所有平台都会严格按这个边界来放信息。

为什么有：
Kubernetes 里很多联动都靠 labels，比如 Service 选 Pod；而 annotations 则适合放更长、更自由的附加信息。普通本地集群就能直接验证。

kubectl 命令：
```bash
# 先把下面的 YAML 保存为 demo-meta.yaml
kubectl apply -f demo-meta.yaml
kubectl get pod demo-meta --show-labels
kubectl get pods -l app=demo
kubectl get pod demo-meta -o jsonpath='{.metadata.annotations.example\.com/owner}'; echo
kubectl delete -f demo-meta.yaml
```

最小 YAML：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-meta
  labels:
    app: demo
    tier: tutorial
  annotations:
    example.com/owner: primer
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Python 小例子：
```python
import json
import subprocess

result = subprocess.run(
    ["kubectl", "get", "pod", "demo-meta", "-o", "json"],
    check=True,
    capture_output=True,
    text=True,
)
pod = json.loads(result.stdout)
print("labels:", pod["metadata"]["labels"])
print("annotations:", pod["metadata"]["annotations"])
```

你会看到什么：
`kubectl get pods -l app=demo` 会按 label 选出这个 Pod；annotation 不参与筛选，但能被单独读出来。你会直观看到两者用途不同。

在 OpenSandbox 里通常对应什么：
在 OpenSandbox 的 Kubernetes 路径里，sandbox ID 和清理策略这类需要筛选或关联的内容更常放在 labels；部分端点信息和附加配置可能放在 annotations。要注意，Docker 路径没有 annotations，所以不少运行时元数据在 Docker 里仍然会继续放在 labels 上。

### 5.5 namespace
是什么：
`namespace` 是同一个集群里的逻辑隔离边界。你可以把它理解成“一个集群里的多个独立工作区”，每个工作区里都可以有自己的 Pod、Service、Secret 等资源名。

为什么有：
这样团队、环境、系统组件就不用全挤在同一个全局名字空间里。普通本地集群就支持；但要注意，namespace 是逻辑隔离，不等于天然的强安全边界。

kubectl 命令：
```bash
# 先把下面的 YAML 保存为 demo-namespace.yaml
kubectl apply -f demo-namespace.yaml
kubectl get ns demo-ns
kubectl get pod -n demo-ns
kubectl delete -f demo-namespace.yaml
```

最小 YAML：
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-ns
---
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: demo-ns
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Python 小例子：
```python
import json
import subprocess

result = subprocess.run(
    ["kubectl", "get", "pods", "-n", "demo-ns", "-o", "json"],
    check=True,
    capture_output=True,
    text=True,
)
data = json.loads(result.stdout)
print([item["metadata"]["name"] for item in data["items"]])
```

你会看到什么：
`demo-pod` 只会出现在 `demo-ns` 里；如果不带 `-n demo-ns`，你通常在默认 namespace 下看不到它。

在 OpenSandbox 里通常对应什么：
OpenSandbox 通常会在配置里指定一个统一的 Kubernetes namespace，把 sandbox 相关资源都放进去。

### 5.6 Service 和 endpoint 暴露
是什么：
Service 是 Kubernetes 给一组 Pod 提供稳定访问入口的方式。Pod IP 可能会变，但 Service 的名字和虚拟地址更稳定；“endpoint 暴露”则是指你最终怎么访问这组 Pod，比如集群内 DNS、`port-forward`、NodePort、Ingress 等。

为什么有：
如果直接记 Pod IP，Pod 一重建地址就变了；用 Service 更适合做稳定访问。普通本地集群就能跑下面的例子；如果你想从宿主机直接访问，最简单的是 `kubectl port-forward`。如果你想用 Ingress，还需要集群里先装好 ingress controller。

kubectl 命令：
```bash
# 先把下面的 YAML 保存为 demo-service.yaml
kubectl apply -f demo-service.yaml
kubectl get pod demo-web
kubectl get svc demo-web
kubectl get endpoints demo-web
kubectl port-forward service/demo-web 8080:80
```

最小 YAML：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-web
  labels:
    app: demo-web
spec:
  containers:
    - name: web
      image: nginx:1.27-alpine
---
apiVersion: v1
kind: Service
metadata:
  name: demo-web
spec:
  selector:
    app: demo-web
  ports:
    - port: 80
      targetPort: 80
```

Python 小例子：
```python
from urllib.request import urlopen

# 先在另一个终端运行：
# kubectl port-forward service/demo-web 8080:80
with urlopen("http://127.0.0.1:8080", timeout=3) as resp:
    print(resp.status)
    print(resp.read(60).decode("utf-8", errors="replace"))
```

你会看到什么：
Service 会根据 selector 把流量转到带 `app=demo-web` 的 Pod。`kubectl get endpoints` 能看到它后面挂着哪个 Pod IP；`port-forward` 之后，你从本机访问 `127.0.0.1:8080` 就能拿到网页内容。

在 OpenSandbox 里通常对应什么：
OpenSandbox 的 `get_endpoint` 不是死绑一种暴露方式，而是由底层 provider 返回可访问地址，可能是 gateway 入口、Pod IP，或者某个服务域名。

### 5.7 RuntimeClass
是什么：
`RuntimeClass` 是 Kubernetes 里“这个 Pod 应该用哪种容器运行时实现来跑”的声明，比如 gVisor、Kata 一类安全运行时。它是 Pod spec 里的一个选择开关，但真正怎么跑，取决于节点上有没有配置对应 handler。

为什么有：
这样平台可以把“业务要不要用更强隔离”从镜像和业务代码里拆出来，交给集群运行时层处理。这里要特别注意集群要求：创建 `RuntimeClass` 资源本身不难，但想让引用它的 Pod 真正跑起来，节点必须已经配置好同名 runtime handler。很多默认本地集群可以创建对象，但不一定真的支持运行。

kubectl 命令：
```bash
# 先把下面的 YAML 保存为 demo-runtimeclass.yaml
kubectl apply -f demo-runtimeclass.yaml
kubectl get runtimeclass demo-runtime
kubectl describe runtimeclass demo-runtime
kubectl delete -f demo-runtimeclass.yaml
```

最小 YAML：
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: demo-runtime
handler: demo-handler
```

Python 小例子：
```python
import json
import subprocess

result = subprocess.run(
    ["kubectl", "get", "runtimeclass", "demo-runtime", "-o", "json"],
    check=True,
    capture_output=True,
    text=True,
)
rc = json.loads(result.stdout)
print("name:", rc["metadata"]["name"])
print("handler:", rc["handler"])
```

你会看到什么：
你会看到集群里多了一个 `RuntimeClass` 对象，但这不代表任何 Pod 现在就能用它成功启动。如果你后续让 Pod 写上 `runtimeClassName: demo-runtime`，而节点没有 `demo-handler`，Pod 会启动失败。

在 OpenSandbox 里通常对应什么：
OpenSandbox 会把 `secure_runtime` 配置解析成 `runtimeClassName`，并在创建工作负载时写进 Pod spec。

### 5.8 workload provider
是什么：
`workload provider` 不是 Kubernetes 原生术语，而是平台内部的一层抽象。它的意思是：上层只说“我要创建一个 sandbox 工作负载”，下层 provider 决定实际去创建和管理哪种资源，比如直接管 Pod，或者管某个自定义 CRD。

为什么有：
平台往往想保持统一的生命周期接口，但底层实现可能不止一种。理解这个概念本身不需要集群；如果要真正运行某个具体 provider，就要满足它自己的集群要求，比如安装对应 CRD 和 controller。对 OpenSandbox 来说，这一点很重要，因为它支持不止一种 Kubernetes 工作负载后端。

kubectl 命令：
```bash
# 先把下面的 YAML 保存为 demo-provider-result.yaml
kubectl apply -f demo-provider-result.yaml
kubectl get pod demo-provider-result
kubectl delete -f demo-provider-result.yaml
```

最小 YAML：
```yaml
# workload provider 不是原生 Kubernetes 资源；
# 下面的 Pod 只是“某个 provider 最终创建出的工作负载”的最小代表。
apiVersion: v1
kind: Pod
metadata:
  name: demo-provider-result
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Python 小例子：
```python
providers = {
    "batchsandbox": "sandbox.opensandbox.io/v1alpha1 BatchSandbox",
    "agent-sandbox": "agents.x-k8s.io/v1alpha1 Sandbox",
}

choice = "batchsandbox"
print(f"{choice} -> {providers[choice]}")
```

你会看到什么：
在集群里你通常看到的是 provider 最终创建或管理的结果，比如 Pod，或者某个 CRD 以及它背后的 Pod；你看不到一个叫 “workload provider” 的 Kubernetes 原生对象。

在 OpenSandbox 里通常对应什么：
OpenSandbox 里它就是 K8s 运行时的一层适配接口，上层统一走同一套生命周期逻辑，底下再切到 `batchsandbox` 或 `agent-sandbox`。

## 6. 把这些概念映射回 OpenSandbox

现在把前面的概念轻量映射回源码。读 OpenSandbox 时，你可以优先这样翻译：

- `image`：用户给的是运行模板，平台要先确认模板存在，再基于它启动实例。
- `container`：真正被 Docker 管理的 sandbox 实例，不是抽象的“环境”。
- `entrypoint` / `command`：平台先接管启动链路，再把真正的用户命令交出去。Docker 路径里这一层主要落在 [`components/execd/bootstrap.sh`](../../components/execd/bootstrap.sh)。
- `env`：给进程看的运行时配置；`labels` / `annotations`：给平台自己查询和管理的运行时元数据，但不要把两者的分工理解成绝对规则。特别是在 Docker 路径里，很多额外元数据仍然只能落在 labels 上。
- `volumes`：让沙箱和外部存储建立关系；K8s 下常常还会和 `init container`、共享卷一起出现。
- `port binding` / Service / endpoint：它们都在回答同一个问题，“外部到底怎么访问这个 sandbox”。
- `network mode` / 网络隔离 / sidecar：它们都和“流量怎么进出、谁负责控制网络”相关。
- `Pod`：Kubernetes 里常见的最小运行单元；`workload provider`：OpenSandbox 为不同 Kubernetes 落地方式加的一层平台抽象。

如果你准备回去读源码，推荐按这条线走：

- Docker 路径先看 [`server/opensandbox_server/services/docker.py`](../../server/opensandbox_server/services/docker.py)，重点盯住 image、entrypoint、command、labels、volumes、network mode。
- K8s 路径再看 [`server/opensandbox_server/services/k8s/kubernetes_service.py`](../../server/opensandbox_server/services/k8s/kubernetes_service.py)，重点盯住 Pod、init container、sidecar、annotations、endpoint。
- 执行链路最后再看 [`server/opensandbox_server/api/proxy.py`](../../server/opensandbox_server/api/proxy.py) 和 `execd` 相关代码，理解“请求怎么转进沙箱内部”。

## 7. 一页术语速查表

| 术语 | 一句话理解 |
| --- | --- |
| image | 容器模板 |
| container | 镜像启动后的运行实例 |
| entrypoint | 容器启动时先执行的入口程序 |
| command | 交给入口程序的具体执行内容 |
| env | 启动时注入给进程的环境变量 |
| labels | 平台写给自己看的元数据标签 |
| volume | 挂进容器的外部存储 |
| port binding | 把容器端口映射到外部可访问端口 |
| network mode | 容器接入网络的方式 |
| 进程启动 | 谁先成为入口进程、谁再拉起其他进程 |
| 前台 / 后台 | 调用方是阻塞等待，还是先返回再异步观察 |
| 超时 | 给操作设置截止时间，到点就取消或回收 |
| 资源限制 | 给 CPU、内存、进程数等划边界 |
| 网络隔离 | 决定谁能访问谁、流量怎样进出 |
| sidecar | 和主容器一起部署的辅助容器模式 |
| Pod | Kubernetes 里最小的调度和运行单元 |
| init container | 主容器启动前先跑完的准备容器 |
| annotations | 不参与筛选的附加元数据 |
| namespace | 集群里的逻辑工作区 |
| Service | 给一组 Pod 提供稳定访问入口 |
| RuntimeClass | 指定 Pod 底层使用哪种运行时实现 |
| workload provider | 平台对不同 Kubernetes 工作负载实现的抽象 |

## 8. 推荐阅读顺序

如果你的目标是“先看懂 OpenSandbox 的运行时代码”，推荐顺序可以很简单：

1. 先看这篇 primer，把 Docker、系统知识、Kubernetes 的核心概念对上。
2. 再看 [`docs/architecture.md`](../architecture.md) 和 [`docs/single_host_network.md`](../single_host_network.md)，补整体视角。
3. 然后看 [`server/opensandbox_server/services/docker.py`](../../server/opensandbox_server/services/docker.py)，重点抓创建路径和启动链路。
4. 接着看 [`server/opensandbox_server/services/k8s/kubernetes_service.py`](../../server/opensandbox_server/services/k8s/kubernetes_service.py)，把 Pod、init container、provider 这些词重新对上。
5. 最后看 [`components/execd/bootstrap.sh`](../../components/execd/bootstrap.sh) 和 [`server/opensandbox_server/api/proxy.py`](../../server/opensandbox_server/api/proxy.py)，把“平台如何把请求送进沙箱内部”补完整。

如果你后面再次觉得某段源码看不懂，不妨先暂停一下，问自己三个问题：

1. 这段代码在操作什么现实对象，是 image、container、Pod、进程，还是网络？
2. 它在解决什么平台问题，是启动、暴露端口、限制资源、自动清理，还是网络控制？
3. 它是在执行主业务逻辑，还是平台为了接管运行时插进去的一层辅助逻辑？

很多时候，只要这三个问题答出来，源码就不再是黑盒了。
