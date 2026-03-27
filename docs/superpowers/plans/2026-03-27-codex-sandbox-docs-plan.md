# Codex Sandbox Docs Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a detailed Chinese documentation set that explains the Codex sandbox across macOS, Linux, and Windows, plus a mechanisms comparison guide and navigation entry.

**Architecture:** Draft the overview, mechanisms comparison, and platform deep dives in parallel from a shared structure, then integrate them locally into a consistent tutorial set with commit-pinned source references. Keep the writing beginner-friendly while staying grounded in actual Codex source files.

**Tech Stack:** Markdown, VitePress docs structure, official Codex GitHub source, official OpenAI Codex documentation

---

### Task 1: Establish the shared doc template

**Files:**
- Create: `docs/superpowers/specs/2026-03-27-codex-sandbox-docs-design.md`
- Create: `docs/superpowers/plans/2026-03-27-codex-sandbox-docs-plan.md`

- [ ] Write the design doc and plan doc that define the file set, audience, and fixed section template.
- [ ] Use the design doc as the contract for all subagent drafting work.

### Task 2: Draft the overview and mechanisms guide

**Files:**
- Create: `docs/zh/codex-sandbox-overview.md`
- Create: `docs/zh/codex-sandbox-mechanisms.md`

- [ ] Write the overview doc that introduces `SandboxPolicy`, `spawn`, and per-platform enforcement at a high level.
- [ ] Write the mechanisms doc that compares filesystem, network, privilege, identity, and fallback behavior across macOS, Linux, and Windows.
- [ ] Include commit-pinned links to key Codex files.

### Task 3: Draft the macOS deep dive

**Files:**
- Create: `docs/zh/codex-sandbox-macos.md`

- [ ] Explain the minimum Seatbelt background needed for a newcomer.
- [ ] Show how Codex generates and applies the Seatbelt profile.
- [ ] Link the explanation to the relevant Codex source files.

### Task 4: Draft the Linux deep dive

**Files:**
- Create: `docs/zh/codex-sandbox-linux.md`

- [ ] Explain the `bubblewrap + seccomp + no_new_privs` pipeline.
- [ ] Explain what `Landlock` still does in the codebase and why it is now a legacy path.
- [ ] Show how the helper re-enters itself for the inner seccomp stage.

### Task 5: Draft the Windows deep dive

**Files:**
- Create: `docs/zh/codex-sandbox-windows.md`

- [ ] Explain the difference between elevated and unelevated Windows sandbox modes.
- [ ] Cover restricted tokens, capability SIDs, ACL shaping, firewall rules, helper setup, and private desktop in beginner-friendly language.
- [ ] Keep claims aligned with source and official docs.

### Task 6: Integrate docs and navigation

**Files:**
- Modify: `docs/zh/index.md`

- [ ] Add a navigation entry that points readers from the Chinese index to the new Codex sandbox docs.
- [ ] Make the reading order obvious from the index.

### Task 7: Verify structure and readability

**Files:**
- Modify: `docs/zh/codex-sandbox-overview.md`
- Modify: `docs/zh/codex-sandbox-macos.md`
- Modify: `docs/zh/codex-sandbox-linux.md`
- Modify: `docs/zh/codex-sandbox-windows.md`
- Modify: `docs/zh/codex-sandbox-mechanisms.md`
- Modify: `docs/zh/index.md`

- [ ] Check links, headings, and relative navigation.
- [ ] Check that each doc follows the shared structure and stays beginner-friendly.
- [ ] Fix mismatched terminology or duplicated explanations across docs.
