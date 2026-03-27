# Codex Sandbox Docs Design

## Goal

Create a Chinese documentation set that helps a newcomer understand how the `openai/codex` sandbox works across macOS, Linux, and Windows, with a strong emphasis on how the design maps to real Codex source code.

## Deliverables

The documentation set will add:

- `docs/zh/codex-sandbox-overview.md`
- `docs/zh/codex-sandbox-macos.md`
- `docs/zh/codex-sandbox-linux.md`
- `docs/zh/codex-sandbox-windows.md`
- `docs/zh/codex-sandbox-mechanisms.md`

It will also update:

- `docs/zh/index.md`

## Audience

Readers who can read some backend code but do not yet have a solid mental model for OS sandboxing, container-adjacent isolation concepts, or the way Codex maps policy to per-platform enforcement.

## Content Shape

This set combines two reading routes:

1. Route A: `overview -> platform deep dive`
2. Route B: `mechanisms` as a horizontal comparison reference

Each platform deep dive should consistently cover:

- what problem the platform sandbox solves
- the minimum background needed
- Codex's strategy on that platform
- the source-code call chain
- the key mechanisms and their limits
- small configuration or code examples
- common misconceptions
- a "how to go back to source" reading path

The mechanisms comparison should consistently cover:

- filesystem isolation
- process and privilege control
- network restriction
- identity or user boundary
- UI or desktop boundary where applicable
- legacy and fallback paths
- cross-platform tradeoffs

## Source Discipline

Primary sources should come from the official Codex repository and official OpenAI Codex documentation. The docs should prefer commit-pinned GitHub links when pointing at source files.

## Constraints

- Explain concepts from first principles, but do not turn the docs into a generic OS security textbook.
- Stay focused on how Codex actually implements the sandbox.
- Avoid presenting uncertain claims as facts, especially on Windows.
- Keep the prose tutorial-like and approachable for beginners.
