---
tags:
  - meta
  - howto
format_version: "2.2.0"
mirrors_setup: "1.5.0"     # human mirror of graphify-obsidian-setup.md (this doc has no independent version; keep in step)
updated: 2026-07-05
---

# How to Setup — Graphify + Obsidian on a New Project

**What this system does:** graphify indexes your codebase into a semantic knowledge graph so Claude can answer architecture questions ("what calls X?", "trace flow through Y") without reading every file. Obsidian is the vault where you store the human side — decisions, gotchas, plans, and "why" context that graphify doesn't capture. Together they form a two-layer knowledge system: the graph for code structure (WHAT/HOW), the vault for reasoning (WHY).

Step-by-step to wire this onto a project. For the *why* and the manual/adapt path, see [[graphify-obsidian-setup]].

**Per new project:** do **Phase 1 (manual scaffold — free)** then **Phase 2 (AI build — the only token cost)**. First time on a machine, do the one-time **Machine setup** to install the toolchain.

---

## Machine setup (one-time per PC — install + configure the toolchain)

Assumes a **clean PC** — nothing installed yet, no vault present. Do this once, in order (later items copy files out of the vault, so the vault has to come first). Versions verified on the reference machine in parentheses.

> [!info] Platform notes
> **macOS / Linux:** all steps run as written in Terminal.
> **Windows (native):** use PowerShell 7+ for all commands. The scaffolder script (Step 6) is bash-only — Windows users skip it and use the Git Bash or manual fallback noted in Phase 1 Step 1. Git hooks run fine on Windows because Git for Windows bundles bash.

**1. Claude Code** — the AI coding assistant this whole system runs inside. Install it from https://claude.ai/code (Mac/Windows desktop app) or via `npm install -g @anthropic-ai/claude-code`. Every `/graphify`, `/obsidian-audit`, and Claude skill invocation happens inside a Claude Code session.

> **All platforms**
```bash
claude --version   # confirm it's installed and on PATH
```

**2. Python 3.10+ and `uv`** (uv installs and manages graphify).

> **macOS**
```bash
brew install uv
```

> **Linux**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

> **Windows (PowerShell)**
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

> **All platforms** — verify install
```bash
uv --version             # (verified: uv 0.11.21, Python 3.13)
```

**3. graphify** — the graph engine. PyPI package is `graphifyy` (double-y); the CLI is `graphify`.

> **All platforms**
```bash
uv tool install graphifyy
graphify --help               # confirm it's on PATH  (verified: graphify 0.8.39)
```

PATH note: uv adds the tools bin to PATH automatically. If `graphify` isn't found, restart your terminal. Manual fix:
- **macOS / Linux:** ensure `~/.local/bin` is in `$PATH`
- **Windows:** ensure `%APPDATA%\uv\bin` is in your `Path` environment variable

To upgrade later: `uv tool upgrade graphifyy`.

**4. Locate your vault root — `<VAULT-ROOT>`.** This is the folder that holds **`FORMAT.md` + a `Projects/` subdir** (usually beside an `.obsidian/` folder). Every command below writes `<VAULT-ROOT>` as a placeholder — **substitute your actual path** (the same way you substitute `<PROJECT>` / `<REPO>`). No environment variable is required.

Known roots (hints, not defaults):

| Setup | `<VAULT-ROOT>` |
|---|---|
| macOS (default) | `~/Obsidian/Claude` |
| macOS (iCloud) | `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Claude` |
| One Windows box | `C:\Users\vince\Desktop\Claude` |

> [!tip] Optional convenience — a shell variable so commands paste verbatim
> Substituting `<VAULT-ROOT>` in every command is tedious. If you prefer, set a shell variable for this
> session and paste `$VAULT` in its place:
> ```bash
> VAULT="$HOME/Obsidian/Claude"   # bash/zsh — your actual vault root
> ```
> ```powershell
> $VAULT = "$HOME\Obsidian\Claude"   # PowerShell
> ```
> This is a **convenience only** — the vault *path* is the source of truth, not any variable. (You may still
> persist a `CLAUDE_VAULT` env var if you like; nothing below depends on it being set.)

**5. Obsidian + the vault** (do this before items 6 & 9 — they copy bundled files out of it). Install the Obsidian app (https://obsidian.md), then sync/restore your vault to your `<VAULT-ROOT>` path, then "Open folder as vault". Claude reads/writes the vault over the **filesystem** — no Obsidian plugin required.

> **macOS / Linux**
```bash
ls "<VAULT-ROOT>"   # confirm vault is present (should show FORMAT.md, Projects/, Templates/, etc.)
```

> **Windows (PowerShell)**
```powershell
ls "<VAULT-ROOT>"   # confirm vault is present
```

**6. `graphify-obsidian-init`** — the one-command scaffolder (used in Phase 1). It's a **personal helper script**, not part of graphify, and it's bundled in the vault. Copy it to your local bin and make it executable:

> **macOS / Linux**
```bash
cp "<VAULT-ROOT>/Templates/bin/graphify-obsidian-init" ~/.local/bin/
chmod +x ~/.local/bin/graphify-obsidian-init
graphify-obsidian-init --help   # confirm it runs and is on PATH
```

> **Windows (native):** this script is bash-only, so install and run it from **Git Bash** — open **"Git Bash"** from the Start menu. Do **not** use `bash` from PowerShell/CMD: on Windows that launches **WSL**, a separate Linux VM with different `/mnt/c/...` paths and its own filesystem. Git Bash instead maps your drive as `/c/...` and integrates with native Git. One-time install (run inside Git Bash):
> ```bash
> # local bin on PATH, persisted in ~/.bashrc. Use a /c/... style path for your
> # vault root (Git Bash maps C:\ as /c/) to avoid backslash issues in cp.
> VAULT="/c/Users/<you>/Desktop/Claude"          # your <VAULT-ROOT>, in /c/... form
> echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
> source ~/.bashrc
> # install the scaffolder
> mkdir -p ~/.local/bin
> cp "$VAULT/Templates/bin/graphify-obsidian-init" ~/.local/bin/
> chmod +x ~/.local/bin/graphify-obsidian-init
> graphify-obsidian-init --help   # confirm it runs and is on PATH
> ```
> Also confirm `graphify` itself is reachable inside Git Bash (`command -v graphify`) — the git hook the scaffolder installs calls `graphify` by name. If it's missing, add its directory to PATH in `~/.bashrc` the same way.
> Prefer not to use Git Bash? **Skip this step** and use the PowerShell fallback in Phase 1 Step 1 (let Claude scaffold).

> If the script is ever missing, the manual steps in [[graphify-obsidian-setup]] reproduce exactly what it does — you can run those instead (see Phase 2's fallback note).

**7. Extraction backend.** graphify's semantic pass (docs/PDFs) uses **Claude subagents by default** — free within your session. If `GEMINI_API_KEY` / `GOOGLE_API_KEY` is set, it routes through Gemini instead; **unset it** to use Claude (or `uv tool install 'graphifyy[gemini]'` if you deliberately want Gemini). The code (AST) pass is always local and 0-token regardless.

**8. Global graph-first directive.** Graph-first behaviour is provided **globally**, not per-project — so you never paste it into individual repos. `CLAUDE.md` is a plain text file (create it if it doesn't exist). Open it in any editor and add this block:

| Platform | Path |
|---|---|
| macOS / Linux | `~/.claude/CLAUDE.md` |
| Windows | `%USERPROFILE%\.claude\CLAUDE.md` |

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

> **macOS / Linux**
```bash
mkdir -p ~/.claude/skills
cp -r "<VAULT-ROOT>/Templates/skills/obsidian-audit" ~/.claude/skills/
cp -r "<VAULT-ROOT>/Templates/skills/obsidian-init" ~/.claude/skills/
cp -r "<VAULT-ROOT>/Templates/skills/graphify" ~/.claude/skills/
cp -r "<VAULT-ROOT>/Templates/skills/obsidian-format-update" ~/.claude/skills/
cp -r "<VAULT-ROOT>/Templates/skills/obsidian-migrate-projects" ~/.claude/skills/
```

> **Windows (PowerShell)**
```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\skills"
Copy-Item -Recurse "<VAULT-ROOT>\Templates\skills\obsidian-audit" "$HOME\.claude\skills\"
Copy-Item -Recurse "<VAULT-ROOT>\Templates\skills\obsidian-init" "$HOME\.claude\skills\"
Copy-Item -Recurse "<VAULT-ROOT>\Templates\skills\graphify" "$HOME\.claude\skills\"
Copy-Item -Recurse "<VAULT-ROOT>\Templates\skills\obsidian-format-update" "$HOME\.claude\skills\"
Copy-Item -Recurse "<VAULT-ROOT>\Templates\skills\obsidian-migrate-projects" "$HOME\.claude\skills\"
```

Then open your `CLAUDE.md` (path in Step 8) and add these two trigger blocks (paste them after the graph-first block from Step 8):

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

Substitute `<PROJECT>` (e.g. `finance-ai`), `<REPO>` (path to the code), and `<VAULT-ROOT>` (your vault root from *Machine setup* Step 4) in the per-project steps below. `<VAULT>` is shorthand for `<VAULT-ROOT>/Projects/<PROJECT>`.

---

The flow is two phases on purpose — **do all the deterministic work manually first (free), then bring in the AI only for the build.** That keeps token spend to the single step that genuinely needs it.

## Phase 1 — Manual scaffold (do it yourself — deterministic, **0 AI tokens**)

Plain terminal work; no Claude session, no tokens. Get the project fully wired before spending anything on the AI.

### Step 1. Scaffold the vault + hooks

> **macOS / Linux**
```bash
cd <REPO>
graphify-obsidian-init <PROJECT>

# Example:
cd ~/Desktop/Projects/finance-ai
graphify-obsidian-init finance-ai
```

> **Windows (native) — two options (pick one):**
> - **Git Bash (recommended):** open **"Git Bash"** from the Start menu (not `bash` in PowerShell — that's WSL). `cd` to your repo using a `/c/...` path, then run the scaffolder:
>   ```bash
>   cd /c/Users/<you>/Desktop/Projects/<PROJECT>
>   graphify-obsidian-init <PROJECT>
>   ```
>   Requires the one-time Git Bash install in *Machine setup* Step 6.
> - **Fallback (PowerShell):** skip the script entirely and let Claude scaffold it — in Phase 2, hand Claude the AI guide (see fallback note in Step 3).

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

> **macOS / Linux**
```bash
grep -qxF 'graphify-out/' .gitignore || { echo '' >> .gitignore; echo 'graphify-out/' >> .gitignore; }
```

> **Windows (PowerShell)**
```powershell
if (-not (Select-String -Path .gitignore -Pattern 'graphify-out/' -Quiet 2>$null)) { Add-Content .gitignore "`ngraphify-out/" }
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

### Step 3. Open Claude Code

Open Claude Code in `<REPO>` (run `claude` from the repo directory, or open the folder in the Claude Code desktop app). The graph build and vault population happen in Step 5 when you hand Claude the AI guide — nothing to run here yet.

> **No per-project CLAUDE.md step.** Graph-first behaviour comes from the global `CLAUDE.md`
> directive (see *Machine setup*), which auto-applies to any repo that has a `graphify-out/`. Nothing to paste here.

> [!note] Use Claude subagents, not Gemini
> If `GEMINI_API_KEY` / `GOOGLE_API_KEY` is set, graphify routes semantic extraction through Gemini. Unset it first to use Claude subagents instead (Step 5 triggers the build).

### Step 4. Verify

Make a trivial commit, then check the hook log (the hook runs in the background — give it a few seconds):

> **macOS / Linux**
```bash
tail ~/.cache/graphify-rebuild.log
```

> **Windows (PowerShell)**
```powershell
Get-Content "$HOME\.cache\graphify-rebuild.log" -Tail 10
```

Expect both lines to appear:
```
[graphify hook] N file(s) changed - rebuilding graph...
[graphify hook] Obsidian vault updated -> <VAULT>/graphify-auto/
```

> [!warning] If the log is empty or shows no "Obsidian vault updated" line
> 1. Check the hook is executable: `ls -l <REPO>/.git/hooks/post-commit` — should show `-rwxr-xr-x`
> 2. Check graphify is on PATH inside the hook's environment: `which graphify`
> 3. Re-run `graphify-obsidian-init <PROJECT>` (it's idempotent) to re-patch the hook
> 4. As a manual fallback: `cd <REPO> && graphify update . && graphify export obsidian --dir "<VAULT>/graphify-auto/"`

### Step 5. Run the AI build + initial vault scan

Open Claude Code in `<REPO>` and give it the AI guide. Because Phase 1's scaffolder already created the vault, hub, and hook, tell Claude to skip those steps:

```
Read <VAULT-ROOT>/Templates/graphify-obsidian-setup.md. The scaffolder already ran (vault folders, hub, and hook are done). Start at Step 2 (build graph), then continue from Step 5 onward. Project name: <PROJECT>.
```

> **Windows / scaffolder skipped:** if Phase 1 used the fallback instead, omit the "scaffolder already ran" sentence — Claude will follow all steps from Step 1.

Claude will build the graph, add the hub Graph section, create the gotcha note, and run `/obsidian-init` to populate the vault with initial specs, knowledge, and reference notes.

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
