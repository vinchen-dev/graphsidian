---
tags:
  - meta
---

# Claude Code + Codex Obsidian Vault

An Obsidian vault that gives Claude Code and Codex persistent memory across sessions. Stores project knowledge — decisions, investigations, API quirks, and patterns — so agents recall context instead of re-deriving it, keeping token usage low. Includes Graphify knowledge graphs for semantic search and impact analysis.

Each project gets its own folder with typed notes (specs, decisions, knowledge, investigations). The active agent reads only what it needs per session. At the end of a session, `/obsidian-audit` captures what's worth keeping as atomic notes linked from the project hub.

> **New project, in a hurry?** If this machine is already set up, open your agent in the new repo and type **`/obsidian-setup`** — it discovers your vault and creates + populates the project's Obsidian folder for you, no commands to run by hand. First time on this machine? Do the [one-time install](#manual-install-one-time-per-machine) below, then paste the prompt from [Templates/How to Setup.md](Templates/How%20to%20Setup.md).

---

## Setup at a glance

There are two layers of setup, and the split matters:

1. **Manual install (this README)** — one time per machine. You install the software by hand: Obsidian, the vault, `uv` + Graphify, and `graphify install`. The agent never installs these.
2. **Per-project wiring (`Templates/How to Setup.md`)** — for each new project you give **one prompt** to Claude Code or Codex, and the agent does the rest (installs the bundled vault skills, discovers the vault, builds the graph, wires the commit hook, and populates the project). See [Templates/How to Setup.md](Templates/How%20to%20Setup.md).

The rest of this README covers layer 1.

---

## Manual install (one time per machine)

Platform notes: **macOS/Linux** run the commands in a terminal; **Windows** use PowerShell 7+ (Graphify's git hooks run fine under Git for Windows).

### 1. Install Obsidian + the vault

- Install the Obsidian app: https://obsidian.md
- Clone (or sync) this vault to a local folder, then **Open folder as vault** in Obsidian:
  ```bash
  git clone https://github.com/vincesn/claude-obsidian-vault ~/Obsidian/Claude
  ```
- **Enable the Obsidian CLI** (Settings → General) so agents can resolve the vault path live. Claude Code and Codex read/write the vault over the **filesystem** — no Obsidian plugin required; the CLI is only used to discover where the vault lives.

The **vault root** is whatever folder ends up holding `FORMAT.md` + `Projects/` (e.g. `~/Obsidian/Claude`, or a Windows box's `C:\Users\<you>\Desktop\Claude`). The per-project agent prompt discovers this automatically.

### 2. Install an agent — Claude Code, Codex, or both

Every `/graphify`, `/obsidian-audit`, and skill invocation happens inside one of these sessions.

```bash
npm install -g @anthropic-ai/claude-code   # Claude Code — or https://claude.ai/code
claude --version

npm install -g @openai/codex               # Codex — or https://developers.openai.com/codex/cli/
codex --version
```

Install both if you want the same vault + Graphify workflow available from either agent.

### 3. Install `uv` + Graphify

`uv` installs and manages Graphify. The PyPI package is **`graphifyy`** (double-y); the CLI is **`graphify`**.

```bash
# macOS
brew install uv
# Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

uv --version
uv tool install graphifyy
graphify --help          # confirm it's on PATH
```

If `graphify` isn't found, restart your terminal. Manual PATH fix — macOS/Linux: ensure `~/.local/bin` is on `$PATH`; Windows: ensure `%APPDATA%\uv\bin` is on your `Path`. Upgrade later with `uv tool upgrade graphifyy`.

### 4. Register Graphify with your agent

Run Graphify's own installer once — this registers the **`/graphify`** skill and graph-first behaviour for your agent:

```bash
graphify install
```

> The `/graphify` skill comes from Graphify itself (this step), **not** from the vault. The vault bundles six skills of its own (the `obsidian-setup` entry point + five other `obsidian-*` skills, listed under [Skills](#skills)), which the per-project setup installs for you.

### 5. (Config) Extraction backend — optional

Graphify's semantic pass (docs/PDFs) uses **subagents from the active Claude Code or Codex session by default**. If `GEMINI_API_KEY` / `GOOGLE_API_KEY` is set, it routes through Gemini instead; **unset it** to use the active coding agent (or `uv tool install 'graphifyy[gemini]'` if you deliberately want Gemini). The code (AST) pass is always local and 0-token regardless. No other configuration is required.

**Manual install done.** From here, adding a project is a single prompt — see [Templates/How to Setup.md](Templates/How%20to%20Setup.md).

---

## Daily use

### Capture knowledge after a session

At the end of any Claude Code or Codex session:

```
/obsidian-audit
```

The agent synthesizes what's worth keeping into atomic notes and links them from the project hub. Use `/obsidian-recall` before debugging or deciding, to check what's already been figured out.

### Use Graphify for codebase queries

If a project has a `graphify-out/` directory, the agent consults the knowledge graph before reading files — for architecture questions, impact analysis, and locating where something happens (`graphify query "<question>"`).

---

## Skills

Six skills are **bundled with this vault** (a seventh, `/graphify`, comes from `graphify install` — see manual install Step 4). The bundled six live in `Templates/skills/` and are copied into `~/.claude/skills/` (Claude Code) or `~/.agents/skills/` (Codex) **automatically the first time you set up a project** — you don't install them by hand.

| Skill | Trigger | What it does |
|---|---|---|
| `obsidian-setup` | `/obsidian-setup` | Wires Graphify + Obsidian onto the current repo **and** scans the codebase to populate `specs/`, `knowledge/`, `reference/` + the hub — the one-word entry point to per-project setup. Re-run to refresh derived notes after big code changes. |
| `obsidian-audit` | `/obsidian-audit` | Persists session knowledge as atomic notes in the vault |
| `obsidian-recall` | `/obsidian-recall` | Recall / investigation lookup before debugging or deciding |
| `obsidian-format-update` | invoke by name | Guides `FORMAT.md` changes — every file to update, version-bumping protocol, per-project migration |
| `obsidian-migrate-projects` | `/obsidian-migrate-projects` | Scans all projects and brings any that are behind up to date after a `FORMAT.md` or setup version bump |
| `obsidian-maintenance` | `/obsidian-maintenance` | Per-project weekly/monthly sweep — re-checks whether `specs`/`reference`/`knowledge` notes still match the current code (graph-first) and proposes fixes for drifted or obsolete notes |

## Vault Structure

```
~/Obsidian/Claude/
├── Home.md                      # map of all projects
├── README.md                    # this file
├── FORMAT.md                    # versioned structure standard
└── Projects/
    └── <project>/
        ├── <project>.md         # hub: overview + index by type
        ├── specs/               # how features/systems work as built
        │   └── <topic>/         # optional: group a feature past ~5 notes
        ├── decisions/           # decisions, ADRs, constraints, "why X"
        │   └── <topic>/         # optional: group related ADRs past ~5 notes
        ├── knowledge/           # gotchas, patterns, API quirks, bug root-causes
        ├── reference/           # endpoints, pricing, doc links
        ├── plans/               # PRDs, implementation plans, roadmaps
        │   └── <topic>/         # optional: group a multi-phase plan past ~5 notes
        └── investigations/      # issue trails: symptom → root cause → resolution
```

Only `specs/`, `decisions/`, and `plans/` may nest one `<topic>/` level — it collapses many hub router lines into one index link, keeping recall at ~1 hub read + 1 note read.

## Note Types

| Type | Purpose |
|---|---|
| `specs/` | How a feature or system works as built |
| `decisions/` | Why a choice was made; ADRs and constraints |
| `knowledge/` | Gotchas, patterns, API quirks, bug root-causes |
| `reference/` | Endpoints, credentials, pricing, links |
| `plans/` | PRDs, implementation plans, feature plans, roadmaps (`active` → `done`) |
| `investigations/` | Symptom → root cause → resolution trails |

## How Recall Works

Claude finds the right note in two steps — no folder exploration:

1. Read the project hub (`Projects/<name>/<name>.md`) — it indexes all notes by type with a one-line hook each.
2. Open the one note the hook points to.

This is designed so an agent uses ~1 hub read + 1 note read per lookup.
