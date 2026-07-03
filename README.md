---
tags:
  - meta
---

# Claude Obsidian Vault

An Obsidian vault that gives Claude Code agents persistent memory across sessions. Stores project knowledge — decisions, investigations, API quirks, and patterns — so agents recall context instead of re-deriving it, keeping token usage low. Includes Graphify knowledge graphs for semantic search and impact analysis.

Each project gets its own folder with typed notes (specs, decisions, knowledge, investigations). Claude reads only what it needs per session. At the end of a session, `/obsidian-audit` captures what's worth keeping as atomic notes linked from the project hub.

## Getting Started

### 1. Clone the vault

```bash
git clone https://github.com/vincesn/claude-obsidian-vault ~/Obsidian/Claude
```

### 2. Point Claude Code at it

Add this to your `~/.claude/CLAUDE.md`:

```markdown
# Vault Recall
When debugging or investigating an issue, check the project hub at
`~/Obsidian/Claude/Projects/<project>/<project>.md` before investigating from scratch.
```

### 3. Add a project

Create a folder under `Projects/` following the structure in `FORMAT.md`:

```
Projects/
└── my-project/
    ├── my-project.md        # hub: overview + note index
    ├── specs/
    ├── decisions/
    ├── knowledge/
    ├── reference/
    └── investigations/
```

### 4. Capture knowledge after a session

At the end of any Claude Code session, run:

```
/obsidian-audit
```

Claude will synthesize what's worth keeping into atomic notes and link them from the project hub.

### 5. Use Graphify for codebase queries

If a project has a `graphify-out/` directory, Claude will consult the knowledge graph before reading files — for architecture questions, impact analysis, and locating where something happens.

## Skills

Five slash commands are bundled with this vault. Install them once via `Templates/How to Setup.md` → Machine setup Step 9; they live in `~/.claude/skills/`.

| Skill | Trigger | What it does |
|---|---|---|
| `graphify` | `/graphify` | Builds or rebuilds the semantic knowledge graph for the current repo |
| `obsidian-audit` | `/obsidian-audit` | Persists session knowledge as atomic notes in the vault; also handles recall and investigation lookup before debugging |
| `obsidian-init` | `/obsidian-init` | One-time codebase scan after initial setup — populates `specs/`, `knowledge/`, `reference/`, and fills hub placeholders |
| `obsidian-format-update` | invoke by name | Guides `FORMAT.md` changes — every file to update, version-bumping protocol, per-project migration |
| `obsidian-migrate-projects` | invoke by name | Migrates existing projects after a `FORMAT.md` or setup version bump |

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
        ├── decisions/           # decisions, ADRs, constraints, "why X"
        ├── knowledge/           # gotchas, patterns, API quirks, bug root-causes
        ├── reference/           # endpoints, pricing, doc links
        └── investigations/      # issue trails: symptom → root cause → resolution
```

## Note Types

| Type | Purpose |
|---|---|
| `specs/` | How a feature or system works as built |
| `decisions/` | Why a choice was made; ADRs and constraints |
| `knowledge/` | Gotchas, patterns, API quirks, bug root-causes |
| `reference/` | Endpoints, credentials, pricing, links |
| `investigations/` | Symptom → root cause → resolution trails |

## How Recall Works

Claude finds the right note in two steps — no folder exploration:

1. Read the project hub (`Projects/<name>/<name>.md`) — it indexes all notes by type with a one-line hook each.
2. Open the one note the hook points to.

This is designed so an agent uses ~1 hub read + 1 note read per lookup.

