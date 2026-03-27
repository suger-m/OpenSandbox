# Runtime Primer Expansion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expand the Chinese runtime primer into a beginner-friendly hands-on tutorial where each core concept includes runnable Docker or Kubernetes commands, a minimal Python example, an observation note, and a light OpenSandbox mapping.

**Architecture:** Keep a single tutorial document at `docs/zh/runtime-primer-for-reading-source.md`, but rewrite it into repeated concept cards with a fixed structure. Use parallel agents to draft independent concept groups, then integrate locally so tone, depth, and formatting remain consistent across the full tutorial.

**Tech Stack:** Markdown, Docker CLI, Kubernetes YAML and `kubectl`, Python standard library, local repository docs and source files.

---

### Task 1: Establish the Tutorial Skeleton

**Files:**
- Modify: `docs/zh/runtime-primer-for-reading-source.md`
- Reference: `docs/superpowers/specs/2026-03-26-runtime-primer-design.md`
- Reference: `docs/zh/index.md`

- [ ] **Step 1: Review the current primer and spec side by side**

Run: `Get-Content -Raw docs/zh/runtime-primer-for-reading-source.md`
Run: `Get-Content -Raw docs/superpowers/specs/2026-03-26-runtime-primer-design.md`
Expected: The current primer is prose-heavy, while the design doc requires concept cards with examples and observation notes.

- [ ] **Step 2: Rewrite the top-level heading structure in the primer**

Replace the current section scaffold with this shape near the top of the document:

```md
# 从零看懂 Docker、系统知识与 Kubernetes

副标题：为阅读 OpenSandbox 运行时代码做准备

## 1. 为什么运行时代码难读

## 2. 怎么使用这篇教程

## 3. Docker 基础

## 4. 一点系统知识

## 5. Kubernetes 基础

## 6. 把这些概念映射回 OpenSandbox

## 7. 一页术语速查表

## 8. 推荐阅读顺序
```

- [ ] **Step 3: Add a reusable concept-card template comment for editors**

Insert this HTML comment once above the first concept section:

```md
<!--
概念卡片模板
### x.x 概念名
是什么：
为什么有：
Docker / kubectl 命令：
Python 小例子：
你会看到什么：
在 OpenSandbox 里通常对应什么：
-->
```

- [ ] **Step 4: Run a quick outline check**

Run: `rg -n "^## |^### " docs/zh/runtime-primer-for-reading-source.md`
Expected: The file shows the eight top-level sections and leaves room for concept-card subsections.

- [ ] **Step 5: Commit the skeleton**

```bash
git add docs/zh/runtime-primer-for-reading-source.md
git commit -m "docs: restructure runtime primer skeleton"
```

### Task 2: Expand Docker Fundamentals

**Files:**
- Modify: `docs/zh/runtime-primer-for-reading-source.md`
- Reference: `server/opensandbox_server/services/docker.py`
- Reference: `components/execd/bootstrap.sh`

- [ ] **Step 1: Add concept cards for `image` and `container`**

Add two concept sections under `## 3. Docker 基础` following this pattern:

```md
### 3.1 image

是什么：

为什么有：

Docker 命令：

```bash
docker image pull alpine:3.20
docker image ls alpine:3.20
```

Python 小例子：

```python
import subprocess

subprocess.run(["docker", "image", "inspect", "alpine:3.20"], check=True)
```

你会看到什么：

在 OpenSandbox 里通常对应什么：
```
```

Repeat the same format for `container` with a minimal `docker run --rm alpine:3.20 echo hello`.

- [ ] **Step 2: Add concept cards for `entrypoint` and `command`**

Use examples that clearly compare default command behavior and explicit override behavior:

```bash
docker run --rm alpine:3.20 echo from-command
docker run --rm --entrypoint /bin/sh alpine:3.20 -c "echo from-entrypoint"
```

Use a Python example that runs both commands with `subprocess.run(...)` and comments on the difference.

- [ ] **Step 3: Add concept cards for `env` and `labels`**

Use examples like:

```bash
docker run --rm -e DEMO_NAME=opensandbox alpine:3.20 sh -c 'echo $DEMO_NAME'
docker run -d --name label-demo --label sandbox.id=demo --label runtime.type=docker nginx:alpine
docker inspect label-demo --format '{{ json .Config.Labels }}'
docker rm -f label-demo
```

Use Python examples based on `os.environ` and `subprocess.check_output(...)`.

- [ ] **Step 4: Add concept cards for `volumes`, `port binding`, and `network mode`**

Use examples like:

```bash
docker run --rm -v ${PWD}:/workspace alpine:3.20 ls /workspace
docker run --rm -d -p 8000:80 --name port-demo nginx:alpine
docker run --rm --network bridge alpine:3.20 ip addr
```

Use Python examples for:
- writing a file into a bind mount
- calling a local HTTP server via `urllib.request`
- comparing network settings via `docker inspect`

- [ ] **Step 5: Run a Markdown and snippet sanity check**

Run: `rg -n "Docker 命令：|Python 小例子：|你会看到什么：|在 OpenSandbox 里通常对应什么：" docs/zh/runtime-primer-for-reading-source.md`
Expected: Each Docker concept card contains the four required subsections.

- [ ] **Step 6: Commit the Docker fundamentals expansion**

```bash
git add docs/zh/runtime-primer-for-reading-source.md
git commit -m "docs: add hands-on docker fundamentals to runtime primer"
```

### Task 3: Expand Systems Concepts

**Files:**
- Modify: `docs/zh/runtime-primer-for-reading-source.md`
- Reference: `components/execd/bootstrap.sh`
- Reference: `server/opensandbox_server/services/docker.py`

- [ ] **Step 1: Add concept cards for process startup and foreground/background**

Use commands that show one process keeping a container alive versus backgrounding a helper:

```bash
docker run --rm alpine:3.20 sh -c 'echo start && sleep 2 && echo end'
docker run --rm alpine:3.20 sh -c 'sleep 30 & echo helper-started && wait'
```

Use Python examples with `subprocess.Popen(...)` and `proc.wait(timeout=...)`.

- [ ] **Step 2: Add concept cards for timeouts and resource limits**

Use commands like:

```bash
docker run --rm alpine:3.20 sh -c 'sleep 5'
docker run --rm --memory 128m --cpus 0.5 alpine:3.20 sh -c 'cat /sys/fs/cgroup/memory.max 2>/dev/null || true'
```

Use Python examples that:
- terminate a long-running process after a timeout
- inspect exit codes from a constrained container run

- [ ] **Step 3: Add concept cards for network isolation and sidecar**

Use examples like:

```bash
docker network create primer-net
docker run --rm --network primer-net alpine:3.20 ping -c 1 127.0.0.1
docker run --rm --network none alpine:3.20 ip addr
```

For sidecar, show a simple `docker compose`-style explanation in Markdown plus two standalone `docker run` commands sharing a network:

```bash
docker run -d --name app-demo --network primer-net nginx:alpine
docker run --rm --network primer-net alpine:3.20 wget -qO- http://app-demo
```

Use Python examples with `socket`, `subprocess`, and a short explanation of the auxiliary-container pattern.

- [ ] **Step 4: Run a systems coverage check**

Run: `rg -n "进程启动|前台|后台|超时|资源限制|网络隔离|sidecar" docs/zh/runtime-primer-for-reading-source.md`
Expected: Each required systems concept appears as its own subsection.

- [ ] **Step 5: Commit the systems section**

```bash
git add docs/zh/runtime-primer-for-reading-source.md
git commit -m "docs: add systems concepts to runtime primer"
```

### Task 4: Expand Kubernetes Fundamentals

**Files:**
- Modify: `docs/zh/runtime-primer-for-reading-source.md`
- Reference: `server/opensandbox_server/services/k8s/kubernetes_service.py`
- Reference: `server/opensandbox_server/services/k8s/batchsandbox_provider.py`
- Reference: `server/opensandbox_server/services/k8s/agent_sandbox_provider.py`

- [ ] **Step 1: Add concept cards for `Pod` and `init container`**

Use minimal YAML blocks and `kubectl` commands like:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: primer-pod
spec:
  containers:
    - name: app
      image: nginx:alpine
```

```bash
kubectl apply -f primer-pod.yaml
kubectl get pod primer-pod -o wide
```

Use Python examples that write the YAML to a temp file and call `kubectl` with `subprocess.run(...)`.

- [ ] **Step 2: Add concept cards for Kubernetes `sidecar` and `labels` vs `annotations`**

Use a Pod YAML with two containers for the sidecar example and a metadata block such as:

```yaml
metadata:
  labels:
    sandbox-id: demo
  annotations:
    opensandbox.io/endpoint: "http://demo"
```

Use `kubectl get pod -o jsonpath=...` examples and a Python example that reads JSON output.

- [ ] **Step 3: Add concept cards for `namespace`, Service / endpoint exposure, and `RuntimeClass`**

Use commands like:

```bash
kubectl create namespace primer-demo
kubectl get svc -n primer-demo
kubectl get runtimeclass
```

Include a small Service YAML and a Python example that shells out to `kubectl`.

- [ ] **Step 4: Add a concept card for `workload provider`**

Explain it as a platform abstraction, not a native Kubernetes term. Include a short pseudo-structure block:

```text
API request -> Kubernetes service -> provider -> Pod / workload resource
```

Use the Python example section to show a simple strategy pattern snippet:

```python
class WorkloadProvider:
    def create(self, request):
        raise NotImplementedError
```

- [ ] **Step 5: Run a Kubernetes section check**

Run: `rg -n "Pod|init container|annotations|namespace|RuntimeClass|workload provider" docs/zh/runtime-primer-for-reading-source.md`
Expected: All required Kubernetes concepts are present with example blocks.

- [ ] **Step 6: Commit the Kubernetes expansion**

```bash
git add docs/zh/runtime-primer-for-reading-source.md
git commit -m "docs: add kubernetes basics to runtime primer"
```

### Task 5: Add OpenSandbox Mapping and Reading Guidance

**Files:**
- Modify: `docs/zh/runtime-primer-for-reading-source.md`
- Modify: `docs/zh/index.md`
- Reference: `docs/zh/opensandbox-first-pass-overview.md`
- Reference: `server/opensandbox_server/api/proxy.py`
- Reference: `server/opensandbox_server/services/docker.py`
- Reference: `server/opensandbox_server/services/k8s/kubernetes_service.py`

- [ ] **Step 1: Rewrite the OpenSandbox mapping section to match the new card style**

Add concise bullets tying concepts back to code, for example:

```md
- `entrypoint` / `command`：在 Docker 路径里，平台会先接管启动链路，再把真正的用户程序交给容器执行。
- `labels`：平台会把 sandbox 标识、过期时间、端口信息等写成运行时元数据，便于后续查询和清理。
- `sidecar`：在需要额外网络控制时，辅助容器承担横切能力，而不是让主容器自己负责所有事情。
```

- [ ] **Step 2: Rewrite the glossary into a tighter two-column table**

Keep the glossary concise and beginner-friendly, using entries like:

```md
| 术语 | 一句话理解 |
| --- | --- |
| image | 容器模板 |
| container | 镜像启动后的运行实例 |
| Pod | Kubernetes 里最小的调度单元 |
```

- [ ] **Step 3: Rewrite the reading-order section with concrete repo paths**

Use this sequence:

```md
1. 先看这篇 primer，建立概念。
2. 再看 [`docs/architecture.md`](../architecture.md) 和 [`docs/single_host_network.md`](../single_host_network.md)。
3. 然后看 [`server/opensandbox_server/services/docker.py`](../../server/opensandbox_server/services/docker.py)。
4. 接着看 [`server/opensandbox_server/services/k8s/kubernetes_service.py`](../../server/opensandbox_server/services/k8s/kubernetes_service.py)。
5. 最后看 [`components/execd/bootstrap.sh`](../../components/execd/bootstrap.sh) 和 [`server/opensandbox_server/api/proxy.py`](../../server/opensandbox_server/api/proxy.py)。
```

- [ ] **Step 4: Normalize the index entry**

Replace the current English-only index link with a Chinese entry:

```md
## 运行时入门

- [从零看懂 Docker、系统知识与 Kubernetes](./runtime-primer-for-reading-source)
```

- [ ] **Step 5: Commit the mapping and index updates**

```bash
git add docs/zh/runtime-primer-for-reading-source.md docs/zh/index.md
git commit -m "docs: polish runtime primer navigation and mappings"
```

### Task 6: Parallel Drafting Execution

**Files:**
- Modify: `docs/zh/runtime-primer-for-reading-source.md`
- Reference: `docs/superpowers/specs/2026-03-26-runtime-primer-design.md`
- Reference: `docs/superpowers/plans/2026-03-26-runtime-primer-expansion.md`

- [ ] **Step 1: Dispatch independent drafting agents with fixed scopes**

Use separate agents for:
- Docker fundamentals cluster
- systems concepts cluster
- Kubernetes cluster
- OpenSandbox mapping cluster

Each agent prompt should include:

```text
Follow the concept-card format exactly:
1. 是什么
2. 为什么有
3. 命令示例
4. Python 小例子
5. 你会看到什么
6. 在 OpenSandbox 里通常对应什么
Do not edit files directly. Return paste-ready Markdown only for your assigned concepts.
```

- [ ] **Step 2: Integrate agent output locally**

Keep only one writer for the final Markdown file. Normalize:
- heading depth
- image versions
- example length
- wording of observation notes
- OpenSandbox mapping detail level

- [ ] **Step 3: Resolve overlap before finalizing**

Delete repeated explanations such as:
- `image` vs `container` restated in later sections
- `sidecar` repeated with different definitions
- duplicated warnings about Kubernetes environment requirements

- [ ] **Step 4: Commit the integrated draft**

```bash
git add docs/zh/runtime-primer-for-reading-source.md
git commit -m "docs: integrate parallel runtime primer drafts"
```

### Task 7: Verification and Final Review

**Files:**
- Modify: `docs/zh/runtime-primer-for-reading-source.md`
- Modify: `docs/zh/index.md`

- [ ] **Step 1: Run structural verification commands**

Run: `rg -n "^### " docs/zh/runtime-primer-for-reading-source.md`
Expected: There is a dedicated concept subsection for every required topic.

Run: `rg -n "Docker 命令：|kubectl|Python 小例子：|你会看到什么：|在 OpenSandbox 里通常对应什么：" docs/zh/runtime-primer-for-reading-source.md`
Expected: The card structure is repeated consistently across the tutorial.

- [ ] **Step 2: Run link and path spot checks**

Run: `rg -n "runtime-primer-for-reading-source|docker.py|kubernetes_service.py|bootstrap.sh|proxy.py" docs/zh/runtime-primer-for-reading-source.md docs/zh/index.md`
Expected: The primer and index reference the intended repo paths and navigation targets.

- [ ] **Step 3: Perform a readability pass**

Read the final file and ensure:
- commands are pasteable
- Python examples are minimal
- Kubernetes environment notes are explicit
- OpenSandbox mappings stay short
- no subsection is just abstract prose without an example

- [ ] **Step 4: Inspect the final diff**

Run: `git diff -- docs/zh/runtime-primer-for-reading-source.md docs/zh/index.md`
Expected: The diff shows a large tutorial expansion with consistent card formatting and no accidental unrelated changes.

- [ ] **Step 5: Commit the verified final version**

```bash
git add docs/zh/runtime-primer-for-reading-source.md docs/zh/index.md
git commit -m "docs: finalize hands-on runtime primer"
```
