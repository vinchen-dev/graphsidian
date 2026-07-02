---
name: obsidian-init
description: Use when the user types /obsidian-init after initial graphify+Obsidian setup — scans the codebase and populates specs/, knowledge/, reference/, and hub placeholders. One-time setup scan only. For capturing session knowledge use /obsidian-audit instead.
---

# Obsidian Init (Initial Vault Population from Codebase)

Scan the project codebase and populate the Obsidian vault with what can be derived from reading the code. This runs **once after initial setup** — not after every session (that's `/obsidian-audit`).

**FIRST: read `$CLAUDE_VAULT/FORMAT.md`** for folder layout, frontmatter, and note structure. Read `$CLAUDE_VAULT/VERSIONS.md` for current version numbers to stamp into frontmatter.

## What to scan

Read in this order:
1. Entry point(s) — `server.js`, `index.js`, `main.py`, `app.py`, or equivalent
2. Route / controller files
3. Service / domain logic files
4. Models and schemas
5. Utilities and middleware
6. Config, constants, and environment variable references (`.env.example`, `CLAUDE.md`)

Use `graphify query "<question>"` to locate files quickly if the graph is already built.

## What to fill

| Target | Fill with |
|--------|-----------|
| `specs/` | How the main flow works as built — entry points, key pipelines, data flow. One note per major feature or flow. |
| `knowledge/` | API quirks, non-obvious patterns, gotchas visible in the code. Only if genuinely surprising — not obvious from reading the file. |
| `reference/` | External endpoints, base URLs, third-party service names, env var names for credentials. |
| Hub `<One-paragraph summary.>` | What the system does, derived from reading entry point and services. |
| Hub `**Stack:**` | The actual tech stack found in `package.json`, `requirements.txt`, imports, or equivalent. |
| Hub `## Key Paths` table | Entry point, key services, key models — with their actual file paths. |

## What NOT to fill

| Folder | Reason |
|--------|--------|
| `decisions/` | The "why" is not in code — skip entirely, ask the human later |
| `plans/` | Future intent — only the human knows this |
| `graphify-auto/` | Machine-generated — never edit manually |

## Confirm before saving

Before writing any file, present the full proposed list to the user:
- Each candidate note: slug, type folder, one-line summary of what it captures
- Proposed hub field fills: show the exact text for summary, stack, key paths
- Anything deliberately skipped and why

**Wait for explicit user approval before creating or editing any file.**

## Anti-hallucination rules (hard)

- Only write what you actually read in the files — no filling gaps, no "probably", no "likely"
- No reconstructed reasoning — if you see a decision in code but not the reason, don't invent a rationale
- No inferred API behavior — only what was directly observed
- No vague notes — skip anything that doesn't pass the three-test bar
- When uncertain, omit — a missing note is recoverable, a wrong note misleads future agents

## Three-test bar

A note must pass ALL three before being written:
1. Read from actual files in this scan — not assumed or inferred
2. Would cost real effort to re-derive
3. A future agent would actually need it

## Note format

```markdown
---
tags:
  - <tag>   # spec | gotcha | pattern | api-quirk | reference
project: <project>
date: <YYYY-MM-DD>
format_version: "<current from FORMAT.md>"
---

# <Title>

See also: [[<project>]]

<The knowledge — concise. One concept.>
```

Link each new note from the hub under its matching type subsection.

## Report

After saving, tell the user: each note created (path + one-line hook), hub fields filled, and anything skipped.
