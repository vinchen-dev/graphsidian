---
name: obsidian-audit
description: Use when the user types /obsidian-audit to persist what's worth keeping from the conversation (decisions, gotchas, API quirks, patterns) as atomic notes in the Obsidian second-brain vault at $CLAUDE_VAULT/. ALSO use when recalling a project's stored knowledge from that vault — it defines the minimal-token recall protocol (read the project hub, open one note). Capture runs only on explicit /obsidian-audit invocation. ALSO use when the user asks to investigate/debug an issue or asks why something is failing: before investigating, read the project hub and check `investigations/` and `knowledge/` for a matching symptom.
---

# Obsidian Audit (Capture to Second Brain)

Persist durable knowledge from this session into the second-brain vault at `$CLAUDE_VAULT/`.

**FIRST: read `$CLAUDE_VAULT/FORMAT.md`** — it is the single source of truth for folder layout, frontmatter, and note structure. The templates below are a summary; FORMAT.md wins on any conflict. For the **current version numbers** to stamp into frontmatter, read `$CLAUDE_VAULT/VERSIONS.md` (the registry tracking both `format_version` ← FORMAT.md and the hub's `setup_version` ← graphify-obsidian-setup.md). Use `obsidian:obsidian-markdown` for note syntax and `obsidian:obsidian-cli` for vault operations.

## Two-Layer Knowledge System

This vault is one layer of a two-layer system:

| Layer | What it holds | How to query |
|-------|--------------|-------------|
| **Vault** (`specs/`, `decisions/`, `knowledge/`, `reference/`, `plans/`, `investigations/`) | Human-curated WHY, constraints, patterns, external facts, implementation plans | Hub → one note (recall protocol below) |
| **Graph** (`graphify-auto/`, `graphify-out/graph.json`) | Machine-extracted code structure: functions, routes, call edges, communities | `graphify query "<question>"` |

Never duplicate graph content into vault notes. Never use vault recall for code-structure questions.

## Core Principle

The vault is **on-demand**: small atomic notes, one concept each, linked from a project hub. Capture only what is worth recalling later and would otherwise be re-derived. When in doubt, leave it out. Every note created carries the `format_version` from FORMAT.md.

**Capture for cheap recall.** Notes are read back by an agent following the recall protocol (below): read the hub, open one note. So every note must be (a) in the right type folder, (b) atomic, and (c) listed in the hub with a **high-signal hook that states what the note answers** — specific enough to pick without opening anything else. Keep the hub a lean router (≤ ~400 words); put content in notes, never in the hub.

## Recall Protocol (how this data is read back — agents, follow this)

To recall project knowledge without burning tokens on exploration:
1. Check `$CLAUDE_VAULT/Projects/<project>/<project>.md` exists. If not, there are no notes — stop.
2. Read **only** the hub. Its `## Notes` router (grouped by type, each `[[note]] — hook`) tells you which single note to open.
3. Open **one** note matching the task. For a multi-part topic, open its `…-00-index`, then the one part.
4. Never read a folder wholesale, never read `_Index_of_*`, never grep the vault when the hub answers it.

For investigate/debug/error tasks, scan hub `### Investigations` and `### Knowledge` hooks for a matching symptom before starting fresh.

**Code-structure questions** ("what calls X?", "trace flow through Y?", "what services touch Z?"): skip the vault entirely — run `graphify query "<question>"` against `graphify-out/graph.json`. The vault holds WHY; the graph holds WHAT and HOW.

## What to Save vs Skip

**The bar is high.** Default to skipping. Only save a note if it passes ALL three tests:
1. **It was explicitly discussed in this session** — not assumed, inferred, or reconstructed. If you didn't read it or the user didn't say it, don't write it.
2. **It would cost real effort to re-derive** — not obvious from reading the code, CLAUDE.md, or git history.
3. **A future agent would actually need it** — it changes a decision, prevents a mistake, or answers a non-obvious "why".

If a candidate note fails any one of these, skip it.

**Save** (each becomes its own atomic note, or extends an existing one):
- A decision + its explicit reasoning from this session ("chose X over Y because…") → `decisions/`
- An API quirk or undocumented behavior actually encountered → `knowledge/`
- A reusable pattern that concretely emerged, with evidence → `knowledge/`
- A bug's root cause identified this session (the *why*, not the fix — fix is in git) → `knowledge/`
- A non-obvious constraint explicitly stated ("never do X because…") → `decisions/` or `knowledge/`
- How a feature/system behaves as built, when that behavior is surprising or non-obvious → `specs/`
- A finalized plan or roadmap — what was decided and how → `plans/`
- An issue investigated this session — symptom, ruled-out hypotheses, root cause, resolution (or current state if open) → `investigations/`

**Skip:**
- Anything not explicitly stated in this session — no filling gaps, no reasonable assumptions, no "probably" or "likely"
- Obvious facts any developer would infer from reading the code
- In-progress / half-finished state — but a finalized plan/roadmap document is *not* "half-finished"; it belongs in `plans/` (exception: an unresolved investigation is savable as `status: open`)
- Anything already in the repo's CLAUDE.md or git history
- Mechanical steps documented elsewhere
- Ephemeral conversation detail (debugging attempts, intermediate outputs, throwaway commands)
- Code structure graphify already captures (function signatures, call graphs, imports, route handlers) — query `graphify-out/graph.json` instead
- Anything representable as a `graphify-auto/` node — the graph is the source of truth for code shape
- Generic software engineering knowledge not specific to this project

## Anti-Hallucination Rules

These are hard rules. Violating them corrupts the vault.

- **Only write what was said.** Every sentence in a note must trace back to something explicitly discussed in the session. No filling gaps with plausible-sounding detail.
- **No reconstructed reasoning.** If the user stated a decision but not the reason, write the decision only — do not invent a rationale.
- **No inferred API behavior.** If an API quirk wasn't directly observed or stated, don't write it as fact.
- **No vague notes.** A note like "LLM analysis may have edge cases" adds no value and pollutes the vault. Skip it.
- **When uncertain, omit.** A missing note is recoverable. A wrong note misleads future agents.

## Workflow

### 1. Identify the project
Derive from the working directory (`~/Desktop/Projects/finance-ai` → `finance-ai`). Check whether `$CLAUDE_VAULT/Projects/<project>/` exists.

### 2. New project? Create the hub
If the folder doesn't exist, create `Projects/<project>/<project>.md`:
```markdown
---
tags:
  - project
project: <project>
status: active
repo: <working-directory>
format_version: "<current FORMAT.md version, from VERSIONS.md>"
setup_version: "<current graphify-obsidian-setup.md version, from VERSIONS.md>"
---

# <project>

<One-paragraph summary. Stack line.>

## Notes

### Specs (`specs/`)
### Decisions (`decisions/`)
### Knowledge (`knowledge/`)
### Reference (`reference/`)
### Plans (`plans/`)

## Key Paths

| Path | Purpose |
|------|---------|
```
(Omit a Notes subsection until it has a note. If the project uses graphify, a `### Graph (`graphify-auto/`)` section is added by the setup template — see FORMAT.md.) Then add a bullet to the Projects list in `$CLAUDE_VAULT/Home.md`:
```markdown
- [[Projects/<project>/<project>|<project>]] — <one-line description>
```

### 3. Extract & write atomic notes
One concept per note. **Pick the type folder by purpose** (see FORMAT.md → Note Types & Folders):

| Folder | Holds | Tag |
|--------|-------|-----|
| `specs/` | how a feature/system behaves as built | `spec` |
| `decisions/` | stakeholder rulings, architectural decisions/constraints, "why X" | `decision` |
| `knowledge/` | reusable lessons | `gotcha` `pattern` `api-quirk` `bug` |
| `reference/` | external lookup facts (endpoints, creds, pricing, links) | `reference` |
| `plans/` | implementation plans, feature plans, roadmaps | `plan` |
| `investigations/` | Issue investigations | `investigation` |

If it's a **multi-part feature** (report sections, API endpoints, pipeline stages), follow FORMAT.md's *Documenting a Multi-Part Feature* pattern under `specs/<topic>/`: numbered index + one atomic note per part + shared cross-cutting notes — don't write one big note. Decisions never go in a spec folder — put them in `decisions/` and link.

Write `Projects/<project>/<folder>/<slug>.md` (slug = 2–4 kebab words):
```markdown
---
tags:
  - <tag>         # spec | decision | gotcha | pattern | api-quirk | bug | reference | plan
project: <project>
date: <YYYY-MM-DD>
format_version: "<current from FORMAT.md>"
---

# <Title>

See also: [[<project>]]

<The knowledge itself — concise. Use callouts for emphasis:>
> [!warning] Constraint
> <the rule that must not be broken>
```
If a note on the topic already exists, extend it rather than duplicating.

**Investigation notes** use this canonical template:
```markdown
---
tags:
  - investigation
project: <project>
date: <YYYY-MM-DD>            # started
status: open                  # open | resolved
resolved:                     # <YYYY-MM-DD> when resolved
format_version: "2.3.0"
---

# <Symptom as title — e.g. "Not allow to split, check setting" on GCash deposit>

See also: [[<project>]]

## Symptom
<What was observed — exact error message, behavior, where it appeared.>

## Context
<Environment, entities involved (transaction/card/setting), how to reproduce.>

## Ruled out
- <Hypothesis> — <why ruled out (evidence)>

## Root cause
<The actual cause — filled when found.>

## Resolution
<What fixed it, step by step — the "do this if it happens again.">

## Links
<Optional: [[distilled-knowledge-note]] — lesson extracted from this investigation.>
```

Rules for investigation notes:
- Extend an existing open investigation note rather than creating a duplicate.
- On resolution: set `status: resolved` + `resolved:` date, fill the Resolution section, cross-link any distilled note.

### 4. Link from the hub
Add each new note under the matching **type subsection** of `## Notes` in `Projects/<project>/<project>.md` (Specs / Decisions / Knowledge / Reference / Plans / Investigations). For a multi-part topic, link only its index note:
```markdown
- [[<slug>]] — <one-line hook>
```

Investigations are listed under `### Investigations (\`investigations/\`)` in the hub. The hook is the observable symptom. Mark open investigations with `(open)`; remove the marker on resolution.

Example:
```markdown
### Investigations (`investigations/`)
- [[split-not-allowed-error]] — GCash deposit shows "not allowed to split" error (open)
```

### 5. Confirm before saving
Before writing any note, present the full proposed list to the user:
- Each candidate note: slug, type folder, and one-line summary of what it captures
- Anything you're deliberately skipping and why

**Wait for explicit user approval before creating or editing any file.** Do not write speculatively and report after — the user decides what gets saved.

### 6. Report
After saving, tell the user: each note created/updated (path + one-line hook), and anything skipped.

## File Path Reference

| Purpose | Path |
|---------|------|
| Vault map | `$CLAUDE_VAULT/Home.md` |
| Usage manual | `$CLAUDE_VAULT/README.md` |
| Format spec | `$CLAUDE_VAULT/FORMAT.md` |
| Project hub | `$CLAUDE_VAULT/Projects/<project>/<project>.md` |
| Spec note | `$CLAUDE_VAULT/Projects/<project>/specs/<slug>.md` (multi-part: `specs/<topic>/`) |
| Decision note | `$CLAUDE_VAULT/Projects/<project>/decisions/<slug>.md` |
| Knowledge note | `$CLAUDE_VAULT/Projects/<project>/knowledge/<slug>.md` |
| Reference note | `$CLAUDE_VAULT/Projects/<project>/reference/<slug>.md` |
| Plan note          | `$CLAUDE_VAULT/Projects/<project>/plans/<slug>.md` |
| Investigation note | `$CLAUDE_VAULT/Projects/<project>/investigations/<slug>.md` |
| Graph nodes (auto) | `$CLAUDE_VAULT/Projects/<project>/graphify-auto/` |
