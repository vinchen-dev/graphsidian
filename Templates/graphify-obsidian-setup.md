---
tags:
  - meta
  - template
doc_version: "1.9.0"
aligns_with_format: "2.6.0"
updated: 2026-08-03
---

# Agent Runbook: Set up Graphify + Obsidian on a Project

**You are an agent (Claude Code, Codex, or equivalent). This document is the ONLY thing you need.**
It is self-contained: read it top to bottom and follow it to wire the two-layer knowledge system —
an Obsidian vault (human WHY: decisions, gotchas, plans) plus a graphify knowledge graph (machine
WHAT/HOW: code structure) — onto the repository the user pointed you at.

The user hands you one short prompt (see [[How to Setup]]); that prompt sends you here. Do not ask the
user to run a pile of commands — you run them. Only stop to ask when this runbook explicitly says to
(vault not found, or a missing prerequisite).

See also: [[FORMAT]] (vault layout). Registry: [[VERSIONS]] · history: [[versions/setup]].

> This document is versioned independently — its own `doc_version` (above), tracked in [[VERSIONS]].
> `aligns_with_format` records which [[FORMAT]] version its vault-layout references match.

---

## Prerequisites — the user installs these by hand (see `README.md`)

Before this runbook can succeed, the user must have completed the manual, one-time software install from
the vault's **`README.md`**. You do **not** install this software — verify it, and if anything is missing,
**stop and point the user at `README.md`** rather than trying to install it yourself:

| Prerequisite | Check | If missing |
|---|---|---|
| Obsidian app + the vault synced locally | vault folder holding `FORMAT.md` + `Projects/` exists (Step 0) | Stop → README "Install Obsidian" |
| `uv` + `graphify` on PATH | `graphify --help` succeeds | Stop → README "Install Graphify" |
| `graphify install` has been run | `/graphify` skill is registered for the agent | Stop → README "Register Graphify" |

`graphify install` is what registers the **`/graphify`** skill and graph-first behaviour — that skill is
**not** bundled in this vault, so don't try to copy it from `Templates/skills`. This runbook installs only
the vault's own bundled skills (Step 1) — `/graphify` is never one of them.

Substitute throughout:
- `<PROJECT>` — project name (e.g. `finance-ai`) — usually the repo's folder name
- `<REPO>` — absolute path to the code you were opened in
- `<VAULT-ROOT>` — the Obsidian vault root (the folder that holds `FORMAT.md` + `Projects/`). **Machine-specific — discover it in Step 0, don't assume.**
- `<VAULT>` — the project's folder inside that root: `<VAULT-ROOT>/Projects/<PROJECT>`

> [!note] Platform / shell
> Command blocks are written for **macOS/Linux (bash)**; a **Windows (PowerShell)** variant follows where the
> syntax differs. Where a step embeds an **absolute path inside code** (the hook's Obsidian export in Step 6),
> substitute the *resolved* `<VAULT>` path for that machine — never a literal `~/Obsidian/...`.

---

## Step 0 — Resolve the vault root (do this first)

`<VAULT-ROOT>` is a **placeholder for a real absolute path** that differs per machine. Resolve it once, here,
and use the result everywhere `<VAULT>` appears. Try in order:

1. **Ask the running Obsidian instance** (preferred — always current, even after the vault moves):
   ```bash
   obsidian vault="Claude" eval code="app.vault.adapter.basePath"
   ```
   Requires Obsidian open with the CLI enabled. Strip the leading `=> ` from the output; verify the returned
   folder contains `FORMAT.md` before using it. (`vault="Claude"` assumes the default vault name — if this
   vault is named something else, adjust the `vault=` value or just fall through to the search below, which
   identifies the vault by content regardless of its folder name.)
2. **Otherwise search for it.** The vault root is the folder that contains **both** `FORMAT.md` and a
   `Projects/` subdir, usually beside an `.obsidian/` folder. Probe common locations:
   ```bash
   # POSIX — first hit wins
   for d in ~/Obsidian/Claude ~/Desktop/Claude ~/Documents/Claude \
            ~/Documents/Obsidian/Claude "$HOME"/*/Claude; do
     [ -f "$d/FORMAT.md" ] && [ -d "$d/Projects" ] && echo "VAULT-ROOT=$d" && break
   done
   ```
   ```powershell
   # Windows PowerShell
   'Obsidian\Claude','Desktop\Claude','Documents\Claude','Documents\Obsidian\Claude' |
     ForEach-Object { Join-Path $HOME $_ } |
     Where-Object { (Test-Path (Join-Path $_ 'FORMAT.md')) -and (Test-Path (Join-Path $_ 'Projects')) } |
     Select-Object -First 1
   ```
   If the common-location probe misses, run a **bounded recursive search** — this finds the vault by content on
   *any* machine, whatever the folder is named or wherever it lives:
   ```bash
   # POSIX — first FORMAT.md that sits beside a Projects/ folder wins
   find "$HOME" -maxdepth 6 -type f -name FORMAT.md 2>/dev/null | while read -r f; do
     [ -d "$(dirname "$f")/Projects" ] && echo "VAULT-ROOT=$(dirname "$f")" && break
   done
   ```
   ```powershell
   # Windows PowerShell
   Get-ChildItem $HOME -Recurse -Depth 6 -Filter FORMAT.md -File -ErrorAction SilentlyContinue |
     Where-Object { Test-Path (Join-Path $_.DirectoryName 'Projects') } |
     Select-Object -First 1 -ExpandProperty DirectoryName
   ```
3. **If it still can't be found — ask the user for the vault-root path.** Don't guess or create a new vault;
   a wrong path silently populates the wrong place.

Known roots so far (hints, not defaults): macOS `~/Obsidian/Claude`; one Windows box
`C:\Users\vince\Desktop\Claude`. The legacy `$CLAUDE_VAULT` env var is **deprecated** — never read or set it.

---

## Step 1 — Install the bundled vault skills (once per machine)

The vault bundles five skills in `<VAULT-ROOT>/Templates/skills/`. Copy them into the active agent's user
skill directory, then register their triggers. **Idempotent — if a skill folder is already present and current,
skip the copy.** (`/graphify` is handled by `graphify install`, not here.)

**First, identify which agent you are** and use its row below. The table covers Claude Code and Codex; if you
are a **different agent**, use *your own* equivalent user-level skill directory and global instruction file
(the analogue of the paths below). If you have **no skill mechanism at all**, install nothing in Steps 1–2 —
instead tell the user exactly which files to add and where, then continue with the project wiring from Step 3.

| Agent | Skill dir | Global instruction file |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `~/.claude/CLAUDE.md` (Win: `%USERPROFILE%\.claude\CLAUDE.md`) |
| Codex | `~/.agents/skills/` | `~/.codex/AGENTS.md` (Win: `%USERPROFILE%\.codex\AGENTS.md`) |

The five bundled skills: **`obsidian-setup`** (the `/obsidian-setup` entry point that reruns this runbook —
including the Step 8 codebase scan), **`obsidian-audit`**, **`obsidian-recall`**, **`obsidian-format-update`**,
**`obsidian-migrate-projects`**.

> **Claude Code — macOS / Linux**
```bash
mkdir -p ~/.claude/skills
for s in obsidian-setup obsidian-audit obsidian-recall obsidian-format-update obsidian-migrate-projects; do
  cp -r "<VAULT-ROOT>/Templates/skills/$s" ~/.claude/skills/
done
```

> **Codex — macOS / Linux** (same, into `~/.agents/skills`)
```bash
mkdir -p ~/.agents/skills
for s in obsidian-setup obsidian-audit obsidian-recall obsidian-format-update obsidian-migrate-projects; do
  cp -r "<VAULT-ROOT>/Templates/skills/$s" ~/.agents/skills/
done
```

> **Windows (PowerShell)** — set `$dst` to `"$HOME\.claude\skills"` (Claude Code) or `"$HOME\.agents\skills"` (Codex)
```powershell
$dst = "$HOME\.claude\skills"
New-Item -ItemType Directory -Force $dst | Out-Null
'obsidian-setup','obsidian-audit','obsidian-recall','obsidian-format-update','obsidian-migrate-projects' |
  ForEach-Object { Copy-Item -Recurse -Force "<VAULT-ROOT>\Templates\skills\$_" $dst }
```

> The bundled copies are **snapshots** for re-install; the live versions this machine runs are the ones in
> `~/.claude/skills/` and/or `~/.agents/skills/`. If you edit one, refresh its vault copy to keep them in sync.

## Step 2 — Ensure the global directives are present (once per machine)

Two behavioural directives live **globally**, not per-project, so they auto-apply to any repo. Check the agent's
global instruction file (table above) and add whichever is missing. **Idempotent — skip any block already present.**

> **Do not add per-skill `/`-trigger blocks here.** The bundled skills installed in Step 1 are auto-discovered
> from each `SKILL.md`'s `description` frontmatter — the agent loads a skill when its trigger is typed *without*
> any block in the global file. Keep the global file to just the two behavioural directives below; per-skill
> trigger blocks are redundant clutter.

**1. Graph-first.** `graphify install` may already have added a graph-first directive; if so, leave it. Otherwise add:

```markdown
# Knowledge Graph (graph-first — only when the repo has one)
If — and only if — the current repo contains a `graphify-out/` directory, it has a Graphify knowledge graph. In that case, **for any task that depends on how the codebase is structured or how its parts relate** — answering architecture/structure questions, locating where something happens, planning or assessing the impact of a change, drawing a diagram / canvas / visualization, explaining a flow, onboarding (this list is illustrative, not exhaustive) — consult the graph first, before reaching for prose, `grep`, or reading files:
- Read `graphify-out/GRAPH_SUMMARY.md` first — lean map: top god nodes, one-liner per community, entry points. Open the full `graphify-out/GRAPH_REPORT.md` only if the summary doesn't answer. If `GRAPH_SUMMARY.md` doesn't exist (older graph), fall back to `GRAPH_REPORT.md`.
- `graphify query "<question>"` — semantic search over the graph.
- `graphify path "A" "B"` / `graphify explain "X"` / `graphify affected "X"` — shortest path between two nodes / a node's neighbors / reverse-impact ("what breaks if I change X").
Read raw source only once the graph has pointed you at the right files. **If there is no `graphify-out/` directory, ignore this section entirely.**
```

**2. Vault recall.** Add the recall directive so the agent checks the vault before re-deriving a decision or debugging from scratch:

```markdown
# Vault Recall (obsidian-recall — before investigating/debugging or re-deriving)
When you are about to investigate or debug an issue, answer why something is failing/erroring, or answer "why did we choose X", "have we seen this bug/error before", or "what's the plan for Y" — and the project has a vault hub at `~/Obsidian/Claude/Projects/<project>/<project>.md` — **before deriving from scratch**, load and follow the `obsidian-recall` skill: read the project hub and scan its `### Investigations`, `### Knowledge`, and `### Decisions` hooks for a matching symptom or question. If a hook matches, open that one note first — it may already hold the root cause, resolution, or rationale. If nothing matches, proceed normally. Also runs on explicit `/obsidian-recall`.
```

---

## Output Locations — what goes where (read before wiring the project)

Graphify produces two kinds of output. They live in **different places on purpose** — do not try to move
everything into Obsidian.

| What | Purpose | Lives in |
|------|---------|----------|
| `graph.json` | Source of truth that `graphify query` reads | **Repo root** `graphify-out/` |
| `cache/`, dated run folders, `memory/` | Incremental-rebuild state (lets the hook skip re-scanning unchanged files) | **Repo root** `graphify-out/` |
| `manifest.json`, `cost.json` | Engine bookkeeping | **Repo root** `graphify-out/` |
| `GRAPH_REPORT.md`, `graph.html` | Human-readable report + interactive viz | Repo root (optionally copied to vault — see below) |
| `graphify-auto/` (one `.md` per code entity) | Browsable graph nodes for Obsidian | **`<VAULT>/graphify-auto/`** |

> [!warning] Do not move `graphify-out/` into the vault
> `graphify query` resolves `graph.json` relative to the repo root, and the hook writes incremental cache there.
> Only `graphify-auto/` (the `.md` export) belongs in Obsidian.

## Step 3 — Gitignore the engine folder + tune the corpus

`graphify-out/` is a regenerated build artifact — keep it out of git, but do **not** move it:

> **macOS / Linux**
```bash
cd <REPO>
grep -qxF 'graphify-out/' .gitignore 2>/dev/null || { echo '' >> .gitignore; echo 'graphify-out/' >> .gitignore; }
```

> **Windows (PowerShell)**
```powershell
if (-not (Select-String -Path .gitignore -Pattern 'graphify-out/' -Quiet 2>$null)) { Add-Content .gitignore "`ngraphify-out/" }
```

Then create a **`.graphifyignore`** at the repo root, **before the first build (Step 5)**, so the graph indexes
signal, not noise. Exclude generated, vendored, and bundled-asset files; **inspect the repo and adjust** — this
is per-project tuning:

```
# .graphifyignore
node_modules/
dist/
build/
public/            # large bundled / single-page UI assets
*.log
graphify-out/
# also list any generated source (e.g. a file written by a codegen/prestart step)
```

Without it, big UI bundles, build output, and downloaded assets dilute community detection and bloat the graph.

## Step 4 — Create the vault hub + folders

> **macOS / Linux (bash)**
```bash
mkdir -p <VAULT>/{specs,decisions,knowledge,reference,plans,investigations}
```

> **Windows (PowerShell)** — brace-expansion isn't supported; loop instead:
```powershell
'specs','decisions','knowledge','reference','plans','investigations' | ForEach-Object {
  New-Item -ItemType Directory -Force "<VAULT>\$_" | Out-Null
}
```

Create `<VAULT>/<PROJECT>.md` from the **Project Hub** template in `FORMAT.md`. Stamp **both** version fields
from [[VERSIONS]]: `format_version` (current FORMAT.md) and `setup_version` (this template's `doc_version` —
records which setup procedure wired the project, so a stale value later flags a re-wire). Add the `## Notes`
router grouped by type and `## Key Paths`. Then add a bullet to `<VAULT-ROOT>/Home.md`:

```markdown
- [[Projects/<PROJECT>/<PROJECT>|<PROJECT>]] — <one-line description>
```

## Step 5 — Build the graph

From the repo root:

```bash
cd <REPO>
/graphify
```

This runs detection → AST extraction (free, code) + semantic extraction (LLM subagents, docs) → clustering →
outputs in `graphify-out/` (`graph.json`, `GRAPH_REPORT.md`, `graph.html`).

> [!note] Use the active agent's subagents, not Gemini
> If `GEMINI_API_KEY`/`GOOGLE_API_KEY` is set graphify routes semantic extraction through Gemini. Unset it to
> use Claude Code or Codex subagents instead. The code (AST) pass is always local and 0-token regardless.

## Step 6 — Export to the vault + install & patch the hook

```bash
graphify export obsidian --dir <VAULT>/graphify-auto/
graphify hook install
```

`graphify hook install` installs **two** hooks: **post-commit** (rebuild on every commit) and **post-checkout**
(rebuild on branch switch) — both 0-token AST rebuilds. **But the stock hooks only rebuild `graph.json` — they do
not re-export to Obsidian.** Patch the post-commit one below. (Neither hook fires on `git pull` — see *Daily use*.)

> [!warning] Husky / a custom `core.hooksPath` (common on JS/TS repos)
> If the repo uses **Husky** (or otherwise sets `git config core.hooksPath`), git does **not** read
> `.git/hooks/` — hooks live in the pointed-at dir (Husky v9 → `.husky/`, shims in `.husky/_`). Consequences:
>
> 1. **`graphify hook install` may error** with *"hooks path from core.hooksPath looks like a Windows path"* —
>    recent graphify rejects a Windows-style `core.hooksPath`. This does **not** mean the hook is missing; check:
>    ```bash
>    git config --get core.hooksPath                 # e.g. .husky/_  → Husky is active
>    grep -l 'graphify-hook-start' .husky/post-commit .git/hooks/post-commit 2>/dev/null
>    ```
>    - If a file already contains `graphify-hook-start`, the hook is installed — skip install, patch that file.
>    - If not, install into the Husky dir manually: create `.husky/post-commit` and `.husky/post-checkout` with
>      graphify's rebuild block (copy from another wired repo, or run `graphify hook install` from a shell where
>      `core.hooksPath` is temporarily unset, then move the generated files into `.husky/`). **Do not** just
>      `git config --unset core.hooksPath` — that silently disables Husky's own hooks (lint-staged, commit-msg).
> 2. Graphify's block **coexists** with Husky's — append it as an extra hook file; don't overwrite
>    `.husky/pre-commit` (lint-staged).
>
> `.husky/` is usually gitignored and hooks are never version-controlled anyway, so re-apply after a fresh clone.

**Wire the Obsidian export into the hook (the missing piece).** Edit the post-commit hook —
`<REPO>/.git/hooks/post-commit`, **or `<REPO>/.husky/post-commit` if the repo uses Husky** (patch whichever file
actually contains the `graphify-hook-start` marker). Find the embedded `_src` Python block, locate the
`_rebuild_code(...)` call, and insert the export immediately after it:

```python
    _rebuild_code(_root, changed_paths=changed, force=_force)

    # Export updated graph to Obsidian vault.
    # Hardcode the DISCOVERED absolute vault path for THIS machine here (the <VAULT>/graphify-auto/ you
    # resolved in Step 0) — the hook runs detached and under GUI git clients, so it can't discover or read
    # env vars at run time. expanduser('~') resolves to the user home on every OS, so pick the expanduser
    # form whose tail matches where the vault actually lives:
    import subprocess as _sp
    _obsidian_dir = os.path.expanduser('~/Obsidian/Claude/Projects/<PROJECT>/graphify-auto/')      # macOS default
    # e.g. a Windows box with the vault on the Desktop:
    # _obsidian_dir = os.path.expanduser('~/Desktop/Claude/Projects/<PROJECT>/graphify-auto/')
    _r = _sp.run(
        [sys.executable, '-m', 'graphify', 'export', 'obsidian', '--dir', _obsidian_dir],
        cwd=str(_root), capture_output=True, text=True
    )
    if _r.returncode == 0:
        print(f'[graphify hook] Obsidian vault updated → {_obsidian_dir}')
    else:
        print(f'[graphify hook] Obsidian export warning: {_r.stderr.strip()}')
```

The export runs in the hook's detached background process, so commits still return immediately. Log:
`~/.cache/graphify-rebuild.log`.

> [!tip] Optional — also surface `GRAPH_REPORT.md` inside Obsidian
> The report (god nodes, surprising connections, suggested questions) reads well in Obsidian. To keep a copy in
> the vault on every commit, add this right after the Obsidian export block (use the RESOLVED absolute path):
> ```python
>     import shutil as _sh
>     _sh.copy(str(_root / 'graphify-out' / 'GRAPH_REPORT.md'),
>              os.path.expanduser('~/Obsidian/Claude/Projects/<PROJECT>/graphify-auto/GRAPH_REPORT.md'))
> ```

## Step 7 — Add the hub Graph section + a hook gotcha note

In `<VAULT>/<PROJECT>.md`, under `## Notes` add:

```markdown
### Graph (`graphify-auto/`)
<!-- @generated — regenerated by graphify on each commit via post-commit hook -->
Auto-generated knowledge graph nodes. Query via `graphify query "<question>"` rather than reading individual nodes.
<!-- /@generated -->
```

Create `<VAULT>/knowledge/git-hook-graphify.md` (tags `gotcha` + `tooling`) recording: the
post-commit/post-checkout hook keeps the graph + vault fresh on every commit (0-token AST rebuild + Obsidian
export); a `git pull` is **not** a commit, so run `graphify update .` to resync after pulling; and re-register
with `graphify hook install` after a fresh clone (hooks aren't version-controlled).

## Step 8 — Populate the vault from the codebase (the initial scan)

Read the codebase and fill the vault with what can be **derived from the code**. This is the one-time initial
population. (Capturing knowledge from a *work session* later is `/obsidian-audit` — a different job; don't
conflate them.)

**Re-scan mode:** if the project is already wired and you were asked only to refresh the derived notes after
the codebase changed a lot, skip Steps 1–7 and run just this step.

**First read** `<VAULT-ROOT>/FORMAT.md` for folder layout, frontmatter, and note structure, and
`<VAULT-ROOT>/VERSIONS.md` for the current `format_version` to stamp.

### What to scan

Read in this order:
1. Entry point(s) — `server.js`, `index.js`, `main.py`, `app.py`, or equivalent
2. Route / controller files
3. Service / domain logic files
4. Models and schemas
5. Utilities and middleware
6. Config, constants, and environment variable references (`.env.example`, `CLAUDE.md`)

Use `graphify query "<question>"` to locate files quickly — the graph is already built (Step 5).

### What to fill

| Target | Fill with |
|--------|-----------|
| `specs/` | How the main flow works as built — entry points, key pipelines, data flow. One note per major feature or flow. |
| `knowledge/` | API quirks, non-obvious patterns, gotchas visible in the code. Only if genuinely surprising — not obvious from reading the file. |
| `reference/` | External endpoints, base URLs, third-party service names, env var names for credentials. |
| Hub `<One-paragraph summary.>` | What the system does, derived from the entry point and services. |
| Hub `**Stack:**` | The actual stack found in `package.json`, `requirements.txt`, imports, or equivalent. |
| Hub `## Key Paths` table | Entry point, key services, key models — with their real file paths. |

**Cannot derive from code — skip entirely:** `decisions/` (the "why" behind a choice isn't in code — ask the
human later), `plans/` (future intent — only the human knows), `graphify-auto/` (machine-generated).

### Anti-hallucination rules (hard)

- Only write what you actually read in the files — no filling gaps, no "probably", no "likely"
- No reconstructed reasoning — if you see a decision in code but not the reason, don't invent a rationale
- No inferred API behavior — only what was directly observed
- No vague notes — skip anything that doesn't pass the three-test bar
- When uncertain, omit — a missing note is recoverable, a wrong note misleads every future agent

### Three-test bar

A note must pass ALL three before being written:
1. Read from actual files in this scan — not assumed or inferred
2. Would cost real effort to re-derive
3. A future agent would actually need it

### Note format

```markdown
---
tags:
  - <tag>   # spec | gotcha | pattern | api-quirk | reference
project: <PROJECT>
date: <YYYY-MM-DD>
format_version: "<current, from VERSIONS.md>"
---

# <Title>

See also: [[<PROJECT>]]

<The knowledge — concise. One concept.>
```

Link each new note from the hub under its matching type subsection, with a **high-signal hook** stating what
the note answers. If one area grows past ~5 related notes, group it under `specs/<topic>/` or
`decisions/<topic>/` with an index note (FORMAT.md → *Topic Grouping*).

### Confirm before saving

Before writing any file, present the full proposed list to the user:
- Each candidate note: slug, type folder, one-line summary of what it captures
- The proposed hub fills: exact text for summary, stack, Key Paths
- Anything deliberately skipped, and why

**Wait for explicit user approval before creating or editing any file.**

## Step 9 — Verify

> **macOS / Linux**
```bash
ls <VAULT>/                              # specs decisions knowledge reference plans investigations graphify-auto
ls <VAULT>/graphify-auto/ | head -3      # one .md per code entity
cat <REPO>/.graphifyignore               # corpus tuning present
if grep -q 'Knowledge Graph' ~/.claude/CLAUDE.md 2>/dev/null || grep -q 'Knowledge Graph' ~/.codex/AGENTS.md 2>/dev/null; then echo "global graph-first directive present ✓"; fi
# make a trivial commit, then (hook runs in the background — give it a few seconds):
tail ~/.cache/graphify-rebuild.log       # should show rebuild + "Obsidian vault updated"
```

> **Windows (PowerShell)**
```powershell
ls <VAULT>\
ls <VAULT>\graphify-auto\ | Select-Object -First 3
Get-Content <REPO>\.graphifyignore
$instructionFiles = "$HOME\.claude\CLAUDE.md", "$HOME\.codex\AGENTS.md" | Where-Object { Test-Path $_ }
if ($instructionFiles | Select-String -Pattern 'Knowledge Graph' -Quiet) { "global graph-first directive present ✓" }
# make a trivial commit, then:
Get-Content "$HOME\.cache\graphify-rebuild.log" -Tail 10   # should show rebuild + "Obsidian vault updated"
```

Expect both lines in the log:
```
[graphify hook] N file(s) changed - rebuilding graph...
[graphify hook] Obsidian vault updated → <VAULT>/graphify-auto/
```

> [!warning] If the log is empty or shows no "Obsidian vault updated" line
> 1. Check the hook is executable: `ls -l <REPO>/.git/hooks/post-commit` — should show `-rwxr-xr-x`.
> 2. Check graphify is on PATH inside the hook's environment: `which graphify`.
> 3. Confirm the export block from Step 6 is actually present in the hook file that fired.
> 4. Manual fallback: `cd <REPO> && graphify update . && graphify export obsidian --dir "<VAULT>/graphify-auto/"`.

Then report back to the user: each note created (path + one-line hook), hub fields filled, and anything skipped.

---

## Daily use after setup

- **Code-structure questions** ("what calls X?", "trace flow through Y"): `graphify query "<question>"` — never read `graphify-auto/` notes by hand.
- **Human WHY/decisions/plans**: vault recall protocol (hub → one note), via `/obsidian-audit` and `/obsidian-recall`.
- **After `git pull` (teammate's changes):** run `graphify update .` (0 tokens, AST-only) to resync the graph. The hook only fires on commit/checkout — a fast-forward pull updates code but not the graph, so it (and its vault export) stay behind until your next commit or a manual `update`.
- **After editing docs (`.md`):** run `/graphify` again (code is automatic; doc concepts aren't).
- **Never** edit `graphify-auto/` manually — it's overwritten every commit. Annotate only inside `<!-- @user -->…<!-- /@user -->` sentinels.
- **Adding a new note type / folder to the vault format**: invoke the `obsidian-format-update` skill — it lists every vault doc that must change, the version-bumping protocol, and what NOT to touch.
- **Bringing existing projects up to date after a format/setup bump**: type **`/obsidian-migrate-projects`** — it scans every project hub, skips those already current, and applies hub-only bumps for MINOR changes vs full note migration for MAJOR changes, updating both `format_version` + `setup_version` in one pass.
