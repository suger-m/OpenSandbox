# Runtime Primer Design

## Background

The user wants a beginner-friendly tutorial that explains the minimum Docker, systems, and Kubernetes concepts needed before reading OpenSandbox runtime source files such as `server/opensandbox_server/services/docker.py` and `server/opensandbox_server/services/k8s/*.py`.

The first version of the tutorial was too shallow. It explained basic concepts, but it did not give the reader enough concrete ways to verify what each concept means. The revised document should therefore shift from a prose-only introduction to a concept-by-concept hands-on tutorial with immediately runnable examples.

The intended audience is still not a platform engineer. The document should work for readers who can follow basic Python and backend code, but get lost when runtime code starts talking about containers, process startup, port mapping, resource limits, timeouts, and sidecars.

## Goals

- Keep the document as a general-intro tutorial rather than a source-code walkthrough.
- Explain concepts in plain Chinese first.
- For every core concept, include a directly runnable `docker` command and a minimal Python example.
- Add a short “what you should observe” note after each example so readers know what behavior to look for.
- Add light OpenSandbox mapping so readers can connect the concepts back to the repo without turning the document into a line-by-line code guide.
- Make the document useful as a pre-reading primer before `docker.py` and Kubernetes runtime files.

## Non-Goals

- Not a full Docker tutorial or operations handbook.
- Not a full Kubernetes operations guide.
- Not a detailed OpenSandbox architecture document; existing architecture docs already cover that.
- Not a step-by-step debugging guide for a specific bug.
- Not a production hardening guide for Docker or Kubernetes clusters.

## Recommended Location

- Main tutorial: `docs/zh/runtime-primer-for-reading-source.md`
- Discovery link: add or maintain an entry in `docs/zh/index.md`

## Revised Structure

1. Why runtime source is hard to read
2. How to use this primer
   - expected reader background
   - local prerequisites
   - how to run the examples
3. Docker basics
   - image
   - container
   - entrypoint vs command
   - env
   - labels
   - volumes
   - port binding
   - network mode
4. A little systems knowledge
   - process startup
   - foreground/background
   - timeouts
   - resource limits
   - network isolation
   - sidecar
5. Kubernetes basics
   - Pod
   - init container
   - sidecar
   - labels vs annotations
   - namespace
   - Service / endpoint exposure
   - RuntimeClass
   - workload provider
6. Map the concepts back to OpenSandbox
7. Quick glossary and suggested reading order

## Section Template

Each concept section should use the same teaching template so the user can quickly recognize the pattern:

1. What it is
2. Why it exists
3. Runnable `docker` or `kubectl` example
4. Minimal Python example
5. What you should observe
6. Light OpenSandbox mapping

For Kubernetes topics that cannot be run on a plain Docker-only machine, the section should still include the minimal YAML and `kubectl` command, but must clearly mark the environment requirement.

## Writing Style

- Chinese, tutorial-first, low jargon.
- Prefer mental models over formal definitions.
- Keep the first paragraph of each concept short and intuitive.
- Use short, realistic commands that readers can paste directly.
- Prefer tiny examples over large scripts.
- Use beginner-friendly images when possible: `busybox`, `alpine`, `python:3.11-slim`, and similarly small official images.
- Prefer Python standard library examples such as `subprocess`, `os.environ`, `http.server`, and `socket`.
- Explicitly call out common confusions such as:
  - image vs container
  - entrypoint vs command
  - sandbox TTL vs command timeout
  - container vs Pod
  - sidecar as an auxiliary container, not a feature flag

## Example Constraints

- Every Docker example should be runnable on a local machine with Docker installed.
- Every Python example should be self-contained and not require third-party packages unless there is a strong reason.
- Kubernetes examples should be clearly labeled as requiring a local cluster or remote cluster.
- Example output should be described in prose rather than copied as large static terminal dumps.
- Examples should teach one concept at a time; avoid mixing multiple new ideas in a single example unless the comparison itself is the lesson.

## Parallel Authoring Strategy

Because the revised document spans many mostly independent concepts, it can be expanded in parallel as long as every contributor follows the same section template and stays within a well-defined scope.

Recommended parallel work split:

- Docker fundamentals cluster:
  - `image` and `container`
  - `entrypoint` and `command`
  - `env` and `labels`
  - `volumes`
  - `port binding` and `network mode`
- Systems concepts cluster:
  - process startup and foreground/background
  - timeouts and resource limits
  - network isolation and sidecar
- Kubernetes cluster:
  - `Pod` and `init container`
  - Kubernetes `sidecar` and `labels` vs `annotations`
  - `namespace`, Service / endpoint exposure, and `RuntimeClass`
  - `workload provider` plus OpenSandbox-specific mapping notes
- Integration work kept local:
  - rewrite the tutorial introduction
  - normalize tone and section template
  - remove overlap between sections
  - add glossary and reading order
  - keep OpenSandbox mappings short and accurate

## Source Baseline

- `docs/architecture.md`
- `docs/single_host_network.md`
- `docs/secure-container.md`
- `docs/zh/opensandbox-first-pass-overview.md`
- `server/opensandbox_server/services/docker.py`
- `server/opensandbox_server/services/k8s/kubernetes_service.py`
- `server/opensandbox_server/services/k8s/batchsandbox_provider.py`
- `server/opensandbox_server/services/k8s/agent_sandbox_provider.py`
- `components/execd/bootstrap.sh`
- `components/execd/pkg/runtime/command.go`

## Review Notes

- The tutorial should remain beginner-first even after adding many examples.
- OpenSandbox references should be short anchoring notes, not the main narrative.
- The examples must be small enough that readers can experiment without setting up the entire OpenSandbox stack.
- The final document should help readers build a usable mental model and then return to the source with less fear.
