---
tags:
  - meta
  - changelog
tracks: graphify-obsidian-setup
updated: 2026-07-02
---

# graphify-obsidian-setup.md — changelog

Version history for [[graphify-obsidian-setup]]. Registry: [[VERSIONS]].

### 1.4.0 — 2026-07-02
- Init script now creates `investigations/` alongside the other type folders (`mkdir -p …{specs,decisions,knowledge,reference,plans,investigations}`).
- Hub heredoc now includes a `### Investigations (\`investigations/\`)` subsection in `## Notes`.
- Additive — no wiring change; re-run on older projects to pick up the new folder.

**Migration (1.4.0):**
- [ ] Run `mkdir investigations/` in any existing project vault folder to match.
- [ ] Add `### Investigations (\`investigations/\`)` to hub `## Notes` when convenient.
- [ ] Bump hub `setup_version` to `"1.4.0"`.

### 1.3.0 — 2026-06-26
- Moved the **graph-first `CLAUDE.md` directive from per-project to a global, conditional directive** in
  `~/.claude/CLAUDE.md` (fires only when the repo has a `graphify-out/`). Removed the per-project paste step
  from both guides; documented the global block as a one-time-per-machine prerequisite in [[How to Setup]]
  → *Machine setup* (so it can be reproduced on a new PC). The init script is unchanged on this point (it
  never touched `CLAUDE.md`).
- Net: a new project no longer needs a `CLAUDE.md` step — graph-first is inherited from the global directive.
- Migration for existing projects: optional — you may delete the now-redundant per-project `## Knowledge
  Graph` section from a repo's `CLAUDE.md` once the global directive is in place. Bump hub `setup_version` to `1.3.0`.

### 1.2.1 — 2026-06-26
- Clarification only (no new requirement): name **post-checkout** alongside post-commit as a hook
  `graphify hook install` creates, restate that neither fires on `git pull`, and (in [[How to Setup]]) note
  the init script stamps both `format_version` + `setup_version` on the hub. No migration — projects at
  1.2.0 are already compliant.

### 1.2.0 — 2026-06-26
- Added **`.graphifyignore`** to setup: create a corpus-tuning ignore file (exclude `node_modules/`, `dist/`,
  `build/`, `public/`, logs, generated source) before the first build so the graph indexes signal, not noise.
- Added a **`CLAUDE.md` graph-first directive** step: a "Knowledge Graph" section telling the assistant to
  read `GRAPH_REPORT.md` / `graphify query` before crawling files — the durable replacement for the fragile
  PreToolUse hook. Mirrored into [[How to Setup]] (now Steps 1–5; new Step 4).
- Additive — existing wiring still valid; re-run these two on older projects to reach 1.2.0.

### 1.1.0 — 2026-06-26
- Added daily-use guidance for **`git pull`**: the post-commit/checkout hook does not fire on a fast-forward
  pull, so the graph (and its Obsidian export) stay stale until the next commit. Run `graphify update .`
  (0 tokens, AST-only) to resync. Mirrored into [[How to Setup]].
- Additive — no change to the setup steps.

### 1.0.0 — 2026-06-24
- Initial setup template: output-location table (engine state at repo root, `graphify-auto/` in the vault),
  6-step wiring (hub → build → export + hook install → patch the Obsidian export into the hook → hub Graph
  section → verify), and daily-use guidance. Aligns with [[FORMAT]] 2.2.0.

<!--
TEMPLATE for future entries:

### X.Y.Z — YYYY-MM-DD
- <what changed>
- Bump doc_version in graphify-obsidian-setup.md frontmatter + the Registry row in VERSIONS.md
-->
