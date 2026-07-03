---
tags:
  - meta
  - template
doc_version: "1.4.0"
aligns_with_format: "2.2.0"
updated: 2026-07-02
---

# Template: Wire Graphify + Obsidian on an Existing Project

Reusable setup guide for adding the two-layer knowledge system (Obsidian vault for human WHY + graphify graph for machine WHAT/HOW) to any existing codebase. See also: [[FORMAT]]. Registry: [[VERSIONS]] · history: [[versions/setup]].

> This document is versioned independently — its own `doc_version` (above), tracked in [[VERSIONS]].
> `aligns_with_format` records which [[FORMAT]] version its vault-layout references match.

Substitute throughout:
- `<PROJECT>` — project name (e.g. `finance-ai`)
- `<REPO>` — absolute path to the code (e.g. `~/Desktop/Projects/<PROJECT>`)
- `<VAULT>` — `~/Obsidian/Claude/Projects/<PROJECT>`

---

## What FORMAT.md does and does NOT cover

| Concern | Source of truth |
|---------|-----------------|
| Vault folder layout, note types, hub template | `FORMAT.md` |
| Running the extraction pipeline | `/graphify` skill |
| Installing the post-commit rebuild hook | `graphify hook install` |
| **Obsidian export wired into the hook** | **This template (Step 4) — not automatic** |

FORMAT.md alone is not enough: it describes the vault, not the graphify pipeline or the hook wiring. Follow the steps below.

---

## Quick start

### Skip map — what the scaffolder already did

If `graphify-obsidian-init` ran successfully in Phase 1 of [[How to Setup]], several steps below are already done. Use this table to know where to start:

| Step | Skip if scaffolder ran? | Why |
|---|---|---|
| Output Locations → `.graphifyignore` | **Skip** | Phase 1 Step 2 already created it |
| Step 1 — vault folders + hub | **Skip** | Script scaffolded folders and created the hub |
| Step 2 — build graph (`/graphify`) | **Run** | Always needed — first graph build |
| Step 3 — `graphify hook install` + export | **Skip** | Script installed and wired the hook |
| Step 4 — patch post-commit hook | **Skip** | Script patched it |
| Step 5 — hub Graph section + gotcha note | **Run** | Not done by script |
| Step 5b — `/obsidian-init` vault scan | **Run** | Not done by script |
| Step 6 — verify | **Run** (partial) | Skip the log check — Phase 2 Step 4 of [[How to Setup]] already covers it |

**If scaffolder ran:** start at **Step 2**, then jump to **Step 5 → 5b → Step 6** (skip log check).
**If scaffolder was skipped (Windows fallback or script unavailable):** follow all steps in order.

> The sections below are the reference that `graphify-obsidian-init` automates. Read them to understand the design, adapt for non-standard layouts, or when the script couldn't patch the hook.

---

## Output Locations — what goes where (read before setup)

Graphify produces two kinds of output. They live in **different places on purpose** — do not try to move everything into Obsidian.

| What | Purpose | Lives in |
|------|---------|----------|
| `graph.json` | Source of truth that `graphify query` reads | **Repo root** `graphify-out/` |
| `cache/`, dated run folders, `memory/` | Incremental-rebuild state (lets the hook skip re-scanning unchanged files) | **Repo root** `graphify-out/` |
| `manifest.json`, `cost.json` | Engine bookkeeping | **Repo root** `graphify-out/` |
| `GRAPH_REPORT.md`, `graph.html` | Human-readable report + interactive viz | Repo root (optionally copied to vault — see below) |
| `graphify-auto/` (one `.md` per code entity) | Browsable graph nodes for Obsidian | **`<VAULT>/graphify-auto/`** |

> [!warning] Do not move `graphify-out/` into the vault
> `graphify query` resolves `graph.json` relative to the repo root, and the hook writes incremental cache there. Only `graphify-auto/` (the `.md` export) belongs in Obsidian.

**Gitignore the engine folder** (it's a regenerated build artifact; the finance-ai target does this) — but do **not** move it:

```
# .gitignore
graphify-out/
```

**Tune the corpus with a `.graphifyignore`** (repo root) so the graph indexes signal, not noise — create it
**before the first build** (Step 2). Exclude generated, vendored, and bundled-asset files; adjust per repo:

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

Without it, big UI bundles, build output, and downloaded assets dilute community detection and bloat the
graph. This is per-project tuning — inspect the repo and add what's noise for *that* codebase.

**Optional — also surface `GRAPH_REPORT.md` inside Obsidian.** The report (god nodes, surprising connections, suggested questions) reads well in Obsidian. To keep a copy in the vault on every commit, add this to the hook in Step 4, right after the Obsidian export block:

```python
    import shutil as _sh
    _sh.copy(str(_root / 'graphify-out' / 'GRAPH_REPORT.md'),
             os.path.expanduser('~/Obsidian/Claude/Projects/<PROJECT>/graphify-auto/GRAPH_REPORT.md'))
```

---

## Step 1 — Create the vault hub

```bash
mkdir -p <VAULT>/{specs,decisions,knowledge,reference,plans,investigations}
```

Create `<VAULT>/<PROJECT>.md` from the **Project Hub** template in `FORMAT.md`. Stamp **both** version fields
from [[VERSIONS]]: `format_version` (current FORMAT.md) and `setup_version` (this template's `doc_version` —
records which setup procedure wired the project, so a stale value later flags a re-wire). Then add the
`## Notes` router grouped by type and `## Key Paths`. Add a bullet to `~/Obsidian/Claude/Home.md`:

```markdown
- [[Projects/<PROJECT>/<PROJECT>|<PROJECT>]] — <one-line description>
```

## Step 2 — Build the graph

From the repo root:

```bash
cd <REPO>
/graphify
```

This runs detection → AST extraction (free, code) + semantic extraction (LLM subagents, docs) → clustering → outputs in `graphify-out/` (`graph.json`, `GRAPH_REPORT.md`, `graph.html`).

> [!note] Use Claude subagents, not Gemini
> If `GEMINI_API_KEY`/`GOOGLE_API_KEY` is set graphify will route semantic extraction through Gemini. Unset it to use Claude subagents instead.

## Step 3 — Export to the vault + install the hook

```bash
graphify export obsidian --dir <VAULT>/graphify-auto/
graphify hook install
```

`graphify hook install` installs **two** hooks: **post-commit** (rebuild on every commit) and **post-checkout**
(rebuild on branch switch) — both 0-token AST rebuilds. **But the stock hooks only rebuild `graph.json` — they
do not re-export to Obsidian.** Fix the post-commit one in Step 4. (Note: neither hook fires on `git pull` — see
*Daily use* for the `graphify update .` resync.)

## Step 4 — Wire the Obsidian export into the hook (the missing piece)

Edit `<REPO>/.git/hooks/post-commit`. Find the embedded `_src` Python block, locate the `_rebuild_code(...)` call, and insert the export immediately after it:

```python
    _rebuild_code(_root, changed_paths=changed, force=_force)

    # Export updated graph to Obsidian vault
    import subprocess as _sp
    _obsidian_dir = os.path.expanduser('~/Obsidian/Claude/Projects/<PROJECT>/graphify-auto/')
    _r = _sp.run(
        [sys.executable, '-m', 'graphify', 'export', 'obsidian', '--dir', _obsidian_dir],
        cwd=str(_root), capture_output=True, text=True
    )
    if _r.returncode == 0:
        print(f'[graphify hook] Obsidian vault updated → {_obsidian_dir}')
    else:
        print(f'[graphify hook] Obsidian export warning: {_r.stderr.strip()}')
```

The export runs in the hook's detached background process, so commits still return immediately. Log: `~/.cache/graphify-rebuild.log`.

> **Graph-first behaviour is global — no per-project `CLAUDE.md` step.** It lives once in the user's global
> `~/.claude/CLAUDE.md` as a conditional directive that fires only when a repo has a `graphify-out/` (so it
> auto-applies to this project and every other graphify repo, with nothing to paste here). That global
> directive is the **durable** mechanism — it works even where `graphify claude install`'s PreToolUse hook
> silently no-ops (newer Claude Code lacks distinct `Grep`/`Glob` tools). It's a one-time-per-machine setup;
> see [[How to Setup]] → *Machine setup* for the exact block to put in `~/.claude/CLAUDE.md` on a new PC.

## Step 5 — Add the hub Graph section + a hook gotcha note

In `<VAULT>/<PROJECT>.md`, under `## Notes` add:

```markdown
### Graph (`graphify-auto/`)
<!-- @generated — regenerated by graphify on each commit via post-commit hook -->
Auto-generated knowledge graph nodes. Query via `graphify query "<question>"` rather than reading individual nodes.
<!-- /@generated -->
```

Create `<VAULT>/knowledge/git-hook-graphify.md` (tag `gotcha` + `tooling`) recording: the post-commit/post-checkout hook keeps the graph + vault fresh on every commit (0-token AST rebuild + Obsidian export); a `git pull` is **not** a commit, so run `graphify update .` to resync after pulling; and re-register with `graphify hook install` after a fresh clone (hooks aren't version-controlled).

## Step 5b — Populate the vault from the codebase

In the Claude Code session, type:

```
/obsidian-init
```

Claude will scan entry points, services, routes, models, utilities, and config, then present the full candidate note list for `specs/`, `knowledge/`, and `reference/` for approval before writing anything. It also fills the hub's placeholder fields (summary, stack, Key Paths table).

> [!note] This is not `/obsidian-audit`
> `/obsidian-audit` is for capturing knowledge from a work session. `/obsidian-init` is for the one-time initial vault population from code. Use each only in its intended context.

| Folder       | What Claude can derive from code                                          |
| ------------ | ------------------------------------------------------------------------- |
| `specs/`     | How the main flow works as built — entry points, key pipelines, data flow |
| `knowledge/` | API quirks, non-obvious patterns, gotchas visible in the code             |
| `reference/` | External endpoints, base URLs, third-party services used                  |

Also fills the hub's placeholder fields:
- `<One-paragraph summary.>` — what the system does
- `**Stack:** <tech · tech · tech>` — the actual stack
- `## Key Paths` table — entry points, key services, key models

**Cannot derive from code — skip:**

| Folder | Why |
|--------|-----|
| `decisions/` | The "why" behind choices isn't in code — needs the human to explain |
| `plans/` | Future intent — only the human knows this |

## Step 6 — Verify

> **macOS / Linux**
```bash
ls <VAULT>/                              # specs decisions knowledge reference plans investigations graphify-auto
ls <VAULT>/graphify-auto/ | head -3      # one .md per code entity
cat <REPO>/.graphifyignore               # corpus tuning present
grep -q 'Knowledge Graph' ~/.claude/CLAUDE.md && echo "global graph-first directive present ✓"
# make a trivial commit, then:
tail ~/.cache/graphify-rebuild.log       # should show rebuild + "Obsidian vault updated"
```

> **Windows (PowerShell)**
```powershell
ls $env:CLAUDE_VAULT\Projects\<PROJECT>\                    # specs decisions knowledge reference plans investigations graphify-auto
ls $env:CLAUDE_VAULT\Projects\<PROJECT>\graphify-auto\ | Select-Object -First 3
Get-Content <REPO>\.graphifyignore
if (Select-String -Path "$HOME\.claude\CLAUDE.md" -Pattern 'Knowledge Graph' -Quiet) { "global graph-first directive present ✓" }
# make a trivial commit, then:
Get-Content "$HOME\.cache\graphify-rebuild.log" -Tail 10   # should show rebuild + "Obsidian vault updated"
```

---

## Daily use after setup

- **Code-structure questions** ("what calls X?", "trace flow through Y"): `graphify query "<question>"` — never read `graphify-auto/` notes by hand.
- **Human WHY/decisions/plans**: vault recall protocol (hub → one note), via `/obsidian-audit` (see [[skills/obsidian-audit/SKILL|obsidian-audit skill]]).
- **After `git pull` (teammate's changes):** run `graphify update .` (0 tokens, AST-only) to resync the graph. The hook only fires on commit/checkout — a fast-forward pull updates code but not the graph, so the graph (and its vault export) stay behind until your next commit or a manual `update`.
- **Never** edit `graphify-auto/` manually — it's overwritten every commit. Annotate only inside `<!-- @user -->…<!-- /@user -->` sentinels.
- **Adding a new note type / folder to the vault format**: invoke the `obsidian-format-update` skill — it lists every vault doc that must change, the version-bumping protocol, and what NOT to touch.
- **Migrating existing projects after a format/setup bump**: invoke the `obsidian-migrate-projects` skill — covers hub-only bumps for MINOR changes vs full note migration for MAJOR changes, and both `format_version` + `setup_version` in one pass.
