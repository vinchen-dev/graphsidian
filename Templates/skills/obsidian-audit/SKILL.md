---
name: obsidian-audit
description: Use ONLY when the user types /obsidian-audit — persists what's worth keeping from the conversation (decisions, gotchas, API quirks, patterns) as atomic notes in the Obsidian second-brain vault (path resolved via the obsidian CLI). Capture runs only on explicit /obsidian-audit invocation. Do NOT load this skill for recall or pre-debug lookup — use obsidian-recall for that.
---

# Obsidian Audit (Capture to Second Brain)

Persist durable knowledge from this session into the Obsidian second-brain vault.

**Locating the vault:** resolve the path at runtime from the running Obsidian instance — never from an env var or a remembered path:

```bash
obsidian vault="Claude" eval code="app.vault.adapter.basePath"
```

The returned absolute path is `<vault>` in everything below. If the command fails, Obsidian isn't running — tell the user to open Obsidian and stop; do not guess the path.

**FIRST: read `<vault>/VERSIONS.md`** — it holds the current version numbers to stamp into frontmatter (`format_version` ← FORMAT.md, and the hub's `setup_version` ← graphify-obsidian-setup.md). For a routine capture, do **not** read FORMAT.md — the folder table and templates embedded below are kept in sync with it and are sufficient. Read `<vault>/FORMAT.md` (source of truth; wins on any conflict) only when: (1) creating a new project hub, (2) using the multi-part spec pattern (`specs/<topic>/`) for the first time in a project, or (3) anything conflicts with these embedded rules or you are otherwise uncertain how to structure a note.

## Core Principle

The vault is **on-demand**: small atomic notes, one concept each, linked from a project hub. Capture only what is worth recalling later and would otherwise be re-derived. When in doubt, leave it out. Every note created carries the `format_version` from FORMAT.md.

**Capture for cheap recall.** Notes are read back via the recall protocol (Vault Recall in `~/.claude/CLAUDE.md`; canonical spec: FORMAT.md → *AI Agent Access*): read the hub, open one note. So every note must be (a) in the right type folder, (b) atomic, and (c) listed in the hub with a **high-signal hook that states what the note answers** — specific enough to pick without opening anything else. Keep the hub a lean router (≤ ~400 words); put content in notes, never in the hub.

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
Derive from the working directory (`~/Desktop/Projects/finance-ai` → `finance-ai`). Check whether `<vault>/Projects/<project>/` exists.

### 2. New project? Create the hub
If the folder doesn't exist, read `<vault>/FORMAT.md` and create `Projects/<project>/<project>.md` from its **Project Hub** template (this is trigger (1) from the note at the top — the hub template is not embedded here; FORMAT.md is the source of truth). Then add a bullet to the Projects list in `<vault>/Home.md`:
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
  - <tag>         # spec | decision | gotcha | pattern | api-quirk | bug | reference | plan | investigation
project: <project>
date: <YYYY-MM-DD>
format_version: "<current FORMAT.md version, from VERSIONS.md>"
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
format_version: "<current FORMAT.md version, from VERSIONS.md>"
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
| Vault map | `<vault>/Home.md` |
| Format spec | `<vault>/FORMAT.md` |
| Project hub | `<vault>/Projects/<project>/<project>.md` |
| Spec note | `<vault>/Projects/<project>/specs/<slug>.md` (multi-part: `specs/<topic>/`) |
| Decision note | `<vault>/Projects/<project>/decisions/<slug>.md` |
| Knowledge note | `<vault>/Projects/<project>/knowledge/<slug>.md` |
| Reference note | `<vault>/Projects/<project>/reference/<slug>.md` |
| Plan note          | `<vault>/Projects/<project>/plans/<slug>.md` |
| Investigation note | `<vault>/Projects/<project>/investigations/<slug>.md` |
