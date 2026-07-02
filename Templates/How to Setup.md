---
tags:
  - meta
  - howto
format_version: "2.2.0"
mirrors_setup: "1.4.0"     # human mirror of graphify-obsidian-setup.md (this doc has no independent version; keep in step)
updated: 2026-07-01
---

# How to Setup — Graphify + Obsidian on a New Project

**What this system does:** graphify indexes your codebase into a semantic knowledge graph so Claude can answer architecture questions ("what calls X?", "trace flow through Y") without reading every file. Obsidian is the vault where you store the human side — decisions, gotchas, plans, and "why" context that graphify doesn't capture. Together they form a two-layer knowledge system: the graph for code structure (WHAT/HOW), the vault for reasoning (WHY).

Step-by-step to wire this onto a project. For the *why* and the manual/adapt path, see [[graphify-obsidian-setup]].

**Per new project:** do **Phase 1 (manual scaffold — free)** then **Phase 2 (AI build — the only token cost)**. First time on a machine, do the one-time **Machine setup** to install the toolchain.

---

## Machine setup (one-time per PC — install + configure the toolchain)

Assumes a **clean PC** — nothing installed yet, no vault present. Do this once, in order (later items copy files out of the vault, so the vault has to come first). Versions verified on the reference machine in parentheses.

> [!info] Platform: macOS / Linux — on Windows, use **WSL2**
> This setup is Unix-based: the scaffolder is a bash script (uses `awk`, `mktemp`, symlinks, `chmod`, `~/.local/bin`), and the git hooks assume Unix paths. macOS and Linux run it as-is. **On Windows, run everything inside WSL2 (Ubuntu)** — install Claude Code, graphify, and the vault *inside* the WSL distro and it all works unchanged (use the `curl … astral.sh/uv` installer below, not `brew`). Native (non-WSL) Windows is **not** supported without porting the script to PowerShell and remapping paths.

**1. Claude Code** — the AI coding assistant this whole system runs inside. Install it from https://claude.ai/code (Mac/Windows desktop app) or via `npm install -g @anthropic-ai/claude-code`. Every `/graphify`, `/obsidian-audit`, and Claude skill invocation happens inside a Claude Code session.
```bash
claude --version   # confirm it's installed and on PATH
```

**2. Python 3.10+ and `uv`** (uv installs and manages graphify).
```bash
# macOS:
brew install uv
# Linux / WSL2 (Ubuntu):
curl -LsSf https://astral.sh/uv/install.sh | sh
uv --version             # (verified: uv 0.11.21, Python 3.13)
```

**3. graphify** — the graph engine. PyPI package is `graphifyy` (double-y); the CLI is `graphify`.
```bash
uv tool install graphifyy     # installs into ~/.local/share/uv/tools/, CLI symlinked to ~/.local/bin/graphify
graphify --help               # confirm it's on PATH  (verified: graphify 0.8.39)
```
Ensure `~/.local/bin` is on your `$PATH`. To upgrade later: `uv tool upgrade graphifyy`.

**4. Set `CLAUDE_VAULT`** — the vault path used by all scripts, skills, and setup commands. Add this to your `~/.zshrc` (or `~/.bashrc` on Linux/WSL2), then reload:
```bash
# Set this to wherever your Obsidian vault lives
echo 'export CLAUDE_VAULT="$HOME/Obsidian/Claude"' >> ~/.zshrc
source ~/.zshrc
echo $CLAUDE_VAULT   # confirm it's set
```
> Change the value if your vault lives elsewhere (e.g. `~/Library/Mobile Documents/.../Claude` for iCloud). Everything below uses `$CLAUDE_VAULT` — no hardcoded paths.

**5. Obsidian + the vault** (do this before items 6 & 9 — they copy bundled files out of it). Install the Obsidian app (https://obsidian.md), then sync/restore your vault to the path you set in `$CLAUDE_VAULT`, then "Open folder as vault". Claude reads/writes the vault over the **filesystem** — no Obsidian plugin required.
```bash
ls "$CLAUDE_VAULT"   # confirm vault is present (should show FORMAT.md, Templates/, etc.)
```

**6. `graphify-obsidian-init`** — the one-command scaffolder (used in Phase 1). It's a **personal helper script**, not part of graphify, and it's bundled in the vault. Copy it to your local bin and make it executable:
```bash
cp "$CLAUDE_VAULT/Templates/bin/graphify-obsidian-init" ~/.local/bin/
chmod +x ~/.local/bin/graphify-obsidian-init
graphify-obsidian-init --help   # confirm it runs and is on PATH
```
> If the script is ever missing, the manual steps in [[graphify-obsidian-setup]] reproduce exactly what it does — you can run those instead (see Phase 2's fallback note).

**7. Extraction backend.** graphify's semantic pass (docs/PDFs) uses **Claude subagents by default** — free within your session. If `GEMINI_API_KEY` / `GOOGLE_API_KEY` is set, it routes through Gemini instead; **unset it** to use Claude (or `uv tool install 'graphifyy[gemini]'` if you deliberately want Gemini). The code (AST) pass is always local and 0-token regardless.

**8. Global graph-first directive.** Graph-first behaviour is provided **globally**, not per-project — so you never paste it into individual repos. `~/.claude/CLAUDE.md` is a plain text file (create it if it doesn't exist). Open it in any editor and add this block:

```markdown
# Knowledge Graph (graph-first — only when the repo has one)
If — and only if — the current repo contains a `graphify-out/` directory, it has a Graphify knowledge graph. In that case, before answering architecture/structure questions or grepping the tree to "find where X happens," consult the graph first:
- Read `graphify-out/GRAPH_REPORT.md` first — god nodes, community map, suggested questions.
- `graphify query "<question>"` — semantic search over the graph.
- `graphify path "A" "B"` / `graphify explain "X"` / `graphify affected "X"` — shortest path / a node's neighbors / reverse-impact.
Read raw source only once the graph has pointed you at the right files. If there is no `graphify-out/` directory, ignore this section entirely.
```

This fires **only** when a repo actually has a `graphify-out/`, so it's harmless in non-graphify projects.

**9. Skills (`/obsidian-audit`, `/obsidian-init`, `/graphify`, `obsidian-format-update`, `obsidian-migrate-projects`).** These are the slash commands and reference skills that power capture/recall, graph-building, and vault maintenance inside Claude Code. All are bundled in the vault — copy them to `~/.claude/skills/`:
```bash
mkdir -p ~/.claude/skills
cp -r "$CLAUDE_VAULT/Templates/skills/obsidian-audit" ~/.claude/skills/
cp -r "$CLAUDE_VAULT/Templates/skills/obsidian-init" ~/.claude/skills/
cp -r "$CLAUDE_VAULT/Templates/skills/graphify" ~/.claude/skills/
cp -r "$CLAUDE_VAULT/Templates/skills/obsidian-format-update" ~/.claude/skills/
cp -r "$CLAUDE_VAULT/Templates/skills/obsidian-migrate-projects" ~/.claude/skills/
```

Then open `~/.claude/CLAUDE.md` and add these two trigger blocks (paste them after the graph-first block from Step 7):

```markdown
# graphify
- **graphify** (`~/.claude/skills/graphify/SKILL.md`) - any input to knowledge graph. Trigger: `/graphify`
When the user types `/graphify`, invoke the Skill tool with `skill: "graphify"` before doing anything else.

# obsidian-audit
- **obsidian-audit** (`~/.claude/skills/obsidian-audit/SKILL.md`) - audit a session for what's worth persisting to the Obsidian vault. Trigger: `/obsidian-audit`
When the user types `/obsidian-audit`, invoke the Skill tool with `skill: "obsidian-audit"` before doing anything else.
```

> The bundled copies (script + skills) are **snapshots** for re-install; the live versions this machine runs are in `~/.local/bin/` and `~/.claude/skills/`. If you edit either, refresh its vault copy too.

---

Substitute `<PROJECT>` (e.g. `finance-ai`) and `<REPO>` (path to the code) in the per-project steps below.

---

The flow is two phases on purpose — **do all the deterministic work manually first (free), then bring in the AI only for the build.** That keeps token spend to the single step that genuinely needs it.

## Phase 1 — Manual scaffold (do it yourself — deterministic, **0 AI tokens**)

Plain terminal work; no Claude session, no tokens. Get the project fully wired before spending anything on the AI.

### Step 1. Scaffold the vault + hooks

```bash
cd <REPO>
graphify-obsidian-init <PROJECT>

# Example:
cd ~/Desktop/Projects/finance-ai
graphify-obsidian-init finance-ai
```

Scaffolds the vault, installs the git hooks (**post-commit** + **post-checkout** — both 0-token rebuilds),
and patches post-commit to export into Obsidian. It creates the project **hub** with both version fields
stamped — `format_version` (current [[FORMAT]]) and `setup_version` (current [[graphify-obsidian-setup]],
see [[VERSIONS]]), and a `knowledge/git-hook-graphify.md` gotcha note. Look for:
- `✓ vault folders ready`
- `✓ created knowledge note …git-hook-graphify.md`
- `✓ graphify hook installed`
- `✓ patched hook → exports into Obsidian on each commit`

> [!warning] If it says `no '_rebuild_code(' anchor` or `hook patch failed`
> graphify's hook layout changed — do the manual hook edit in **Step 4** of [[graphify-obsidian-setup]].

### Step 2. Gitignore the engine folder + tune the corpus

```bash
grep -qxF 'graphify-out/' .gitignore || { echo '' >> .gitignore; echo 'graphify-out/' >> .gitignore; }
```

`graphify-out/` is a build artifact (graph.json + cache) — keep it out of git.

Then create a **`.graphifyignore`** at the repo root so the graph skips noise (big UI bundles, build output,
downloaded assets, generated files). Start from this and adjust for the repo:

```
# .graphifyignore
node_modules/
dist/
build/
public/
*.log
graphify-out/
```

## Phase 2 — AI build (in Claude — the **only** token-spending part)

Now open Claude Code in `<REPO>` (run `claude` from the repo directory, or open the folder in the Claude Code desktop app). Phase 1 did all the deterministic wiring for free; the AI is needed only for the semantic graph build.

### Step 3. Build the graph

> [!note] Use Claude subagents, not Gemini
> If `GEMINI_API_KEY` / `GOOGLE_API_KEY` is set, graphify routes semantic extraction through Gemini. Unset it first to use Claude subagents instead.

Inside the Claude Code session, type:

```
/graphify
```

AST extraction is free; semantic extraction of doc files (subagents) + the first Obsidian export are the token cost. One time; cached after — later commits rebuild for 0 tokens via the hook.

> **No per-project CLAUDE.md step.** Graph-first behaviour comes from the global `~/.claude/CLAUDE.md`
> directive (see *Machine setup*), which auto-applies to any repo that has a `graphify-out/`. Nothing to paste here.

> **Fallback (only if Phase 1's script wasn't available):** you can have Claude do the scaffolding too by handing it the AI guide — *"Read `$CLAUDE_VAULT/Templates/graphify-obsidian-setup.md` and follow it to wire up this project (`<PROJECT>`); stamp versions from `$CLAUDE_VAULT/VERSIONS.md`."* This costs **more tokens** (the AI redoes deterministic work the script does for free), so prefer the Phase 1 script and use this only when it's missing.

### Step 4. Verify

Make a trivial commit, then:

```bash
tail ~/.cache/graphify-rebuild.log
```

Expect both lines to appear (the hook runs in the background — give it a few seconds after the commit returns):
```
[graphify hook] N file(s) changed - rebuilding graph...
[graphify hook] Obsidian vault updated -> $CLAUDE_VAULT/Projects/<PROJECT>/graphify-auto/
```

> [!warning] If the log is empty or shows no "Obsidian vault updated" line
> 1. Check the hook is executable: `ls -l <REPO>/.git/hooks/post-commit` — should show `-rwxr-xr-x`
> 2. Check graphify is on PATH inside the hook's environment: `which graphify`
> 3. Re-run `graphify-obsidian-init <PROJECT>` (it's idempotent) to re-patch the hook
> 4. As a manual fallback: `cd <REPO> && graphify update . && graphify export obsidian --dir "$CLAUDE_VAULT/Projects/<PROJECT>/graphify-auto/"`

### Step 5. Run the AI build + initial vault scan

Open Claude Code in `<REPO>` and give it the AI guide:

```
Open obsidian://open?vault=Claude&file=Templates%2Fgraphify-obsidian-setup and follow it to wire up this project (<PROJECT>).
```

This is the only token-spending step. It builds the graph (`/graphify`) and populates the vault (`/obsidian-init`).

---

## After setup — daily use

| You want… | Do this |
|-----------|---------|
| "What calls X? / trace flow through Y" | `graphify query "<question>"` (in the repo) |
| Keep the graph fresh (your own work) | nothing — every commit auto-updates it (zero tokens) |
| Resync after `git pull` (teammate's code) | `graphify update .` — free AST rebuild. A pull updates the **code** but not the graph until your next commit, so run this if you need a fresh graph before then |
| Refresh after editing **docs** (`.md`) | run `/graphify` again (code is automatic; doc concepts aren't) |
| Persist a decision / gotcha / plan | `/obsidian-audit` |
| Browse the graph | open `graphify-out/graph.html`, or `graphify-auto/` in Obsidian |
| Add a new note type / folder to the vault format | invoke `obsidian-format-update` skill — complete file checklist so nothing is missed |
| Migrate existing projects after a format/setup version bump | invoke `obsidian-migrate-projects` skill — hub-only bumps for MINOR changes, full migration for MAJOR |
