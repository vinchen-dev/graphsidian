---
tags:
  - meta
  - changelog
tracks: graphify-obsidian-setup
updated: 2026-07-05
---

# graphify-obsidian-setup.md — changelog

Version history for [[graphify-obsidian-setup]]. Registry: [[VERSIONS]].

### 1.6.0 — 2026-07-05
- **`$CLAUDE_VAULT` env var fully deprecated** (1.5.0 had demoted it to an optional first probe). The vault
  root is now resolved from the **running Obsidian instance via the obsidian CLI**:
  `obsidian vault="Claude" eval code="app.vault.adapter.basePath"` (strip the leading `=> `; verify
  `FORMAT.md` exists in the result). Discovery order is now: (1) obsidian CLI, (2) filesystem search,
  (3) ask the user. Never read or set `$CLAUDE_VAULT`.
- `graphify-obsidian-init` no longer honors `$CLAUDE_VAULT`: it resolves `VAULT_BASE` via the obsidian CLI
  and falls back to `~/Obsidian/Claude` if Obsidian isn't running.
- The three vault skills (`obsidian-recall`, `obsidian-audit`, `obsidian-init`) resolve the vault the same
  way — see their "Locating the vault" sections; template copies re-synced from `~/.claude/skills/`.
- Additive — correctly-wired projects need no re-wiring (the hook's export path was already hardcoded at
  scaffold time).

**Migration (1.6.0):**
- [ ] Remove any persisted `CLAUDE_VAULT` env var (`[Environment]::SetEnvironmentVariable("CLAUDE_VAULT", $null, "User")` on Windows).
- [ ] Bump hub `setup_version` to `"1.6.0"` when convenient — no wiring changes needed.

### 1.5.0 — 2026-07-05
- **Cross-platform / portability pass** (template was macOS-centric). Changes:
  - **Vault root is no longer hardcoded** to `~/Obsidian/Claude`, and no longer hinges on an env var. Added
    `<VAULT-ROOT>` as a placeholder resolved by a **discover-first protocol**: (1) use `$CLAUDE_VAULT` only if
    already set and valid, (2) else search common locations for the folder holding `FORMAT.md` + `Projects/`
    (POSIX + PowerShell snippets given), (3) else **ask the user** — never guess or create a new vault. Known
    roots listed as hints only (macOS `~/Obsidian/Claude`; a Windows box `C:\Users\vince\Desktop\Claude`).
  - Added a **Platform / shell** note: bash blocks are macOS/Linux; Windows uses PowerShell/Git Bash;
    embedded absolute paths get the *resolved* `<VAULT>`, not a literal `~/Obsidian/...`.
  - Added a **Husky / `core.hooksPath`** warning (Step 3): `graphify hook install` can error on a
    Windows-style `core.hooksPath`, hooks live in `.husky/` not `.git/hooks/`, how to detect an
    already-installed block, how to install into `.husky/` without disabling Husky's own hooks, and that
    graphify's block coexists with lint-staged.
  - **Step 4** now says to patch `.git/hooks/post-commit` **or `.husky/post-commit`** (whichever holds the
    `graphify-hook-start` marker), and the embedded export path carries a Windows example + a note that the
    hook runs detached so `$CLAUDE_VAULT` can't be relied on — hardcode the resolved path.
  - **Step 1** gained a PowerShell variant (brace-expansion `mkdir` isn't supported there).
  - Bumped `aligns_with_format` 2.2.0 → 2.4.0 (layout references — incl. `investigations/` — match current FORMAT).
- Additive/clarifying — no change to correctly-wired projects. A project already at 1.4.0 stays valid; bump
  its hub `setup_version` to `1.5.0` opportunistically (nothing to re-wire unless it uses Husky and was
  missing the `.husky/` hook or the Obsidian export patch).

**Migration (1.5.0):**
- [ ] No action for POSIX projects wired correctly at 1.4.0 — bump hub `setup_version` to `"1.5.0"` when convenient.
- [ ] If a project uses Husky/`core.hooksPath`: confirm the graphify rebuild block lives in `.husky/post-commit`
      (not `.git/hooks/`) and that the **Obsidian export patch** (Step 4) is present in it.

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
