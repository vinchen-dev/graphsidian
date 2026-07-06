---
tags:
  - meta
format_version: "2.5.0"
updated: 2026-07-06
---

# Project Format — v2.5.0

The **single source of truth** for how every project is structured in this vault. The `/obsidian-audit` skill follows this file. Change the format here, bump the version, log it in the [Changelog](#changelog), then migrate existing projects (see [Migrating](#migrating)).

Every project note carries a `format_version` matching the version of this file it was last built/migrated against — so you can tell at a glance what needs updating.

## AI Agent Access (recall) — read this first

The whole layout exists to let an agent recall knowledge **without exploring** — one router read, one note read. Protocol:

1. **Check the hub exists:** `Projects/<project>/<project>.md`. If not, the project has no notes — stop, don't search.
2. **Read only the hub.** Its `## Notes` is a router: every note listed as `[[note]] — hook`, grouped by type. The hook says what each note answers — enough to choose without opening anything else.
3. **Open exactly one note** whose hook matches the task. For a multi-part topic, open its `…-00-index` (a sub-router), then the one part you need.
4. **Never** read a folder wholesale, never read `_Index_of_*` files (auto-generated noise), never grep the vault when the hub answers it.

This costs ~1 hub + 1 note per recall regardless of how big the project gets. The rules below (lean hub, high-signal hooks, atomic notes) are what keep that true.

## Folder Layout

```
Projects/<project>/
  <project>.md          # hub — the only entry point; overview + index by type
  specs/                # how features/system work AS BUILT (behavior reference)
    <slug>.md
    <topic>/            # multi-part feature → topic subfolder (see Multi-Part)
      <topic>-00-index.md
      <topic>-NN-<part>.md
  decisions/            # decisions, ADRs, architectural constraints, "why X"
    <slug>.md
  knowledge/            # reusable lessons: gotchas, patterns, api-quirks, bug root-causes
    <slug>.md
  reference/            # external lookup facts: endpoints, credentials, pricing, doc links
    <slug>.md
  plans/                # implementation plans, feature plans, roadmaps (human-authored)
    <slug>.md
  investigations/       # issue investigations: symptom → ruled-out → root cause → resolution
    <slug>.md
  graphify-auto/        # machine-generated graph nodes (one .md per code entity)
                        # rebuilt automatically by post-commit hook — never edit manually
```

One project = one folder. Inside it, notes are separated by **type folder** (below). Create a type folder only when it has a note — don't pre-create empty ones. Tiny projects may keep a few notes flat beside the hub until a type accumulates enough to warrant its folder.

## Note Types & Folders

Every atomic note belongs to exactly one type folder, chosen by **purpose**:

| Folder | Holds | Tag | Ask |
|--------|-------|-----|-----|
| `specs/` | How our feature/system behaves as built | `spec` | "How does X work?" |
| `decisions/` | Stakeholder rulings, architectural decisions/constraints, "why we chose X" (dated, sourced) | `decision` | "Why is it this way / what did we agree?" |
| `knowledge/` | Reusable lessons not tied to one feature | `gotcha` `pattern` `api-quirk` `bug` | "What did we learn / what's the trap?" |
| `reference/` | External facts you look up, not derive | `reference` | "What's the URL / key / price?" |
| `plans/` | Implementation plans, feature plans, roadmaps — what we're building and how we'll build it | `plan` | "What's the plan for X / what are we building?" |
| `investigations/` | Issue investigations — symptom, hypotheses ruled out, root cause, resolution | `investigation` | "Have we seen this issue before / how was it fixed?" |

The note's primary tag matches its folder's domain. When a note could fit two folders, pick by the **question it answers** (the "Ask" column).

## Machine-Generated Folder — `graphify-auto/`

`graphify-auto/` is **not a human type folder**. It is rebuilt automatically by the post-commit git hook (`graphify --update`). Do not create, rename, or edit notes here manually.

**Contents:** one `.md` per code entity (function, route, service, model). Notes contain wikilinks to related nodes and appear in Obsidian Graph View alongside human notes.

**Sentinel convention:** If you annotate a generated node, wrap your addition in `<!-- @user -->…<!-- /@user -->`. The `<!-- @generated -->…<!-- /@generated -->` block is overwritten on every rebuild; only `<!-- @user -->` blocks survive.

**Recall:** for code-structure questions ("what calls X?", "trace the flow through Y"), query the graph — `graphify query "<question>"` — rather than reading `graphify-auto/` notes individually. The vault recall protocol (hub → one note) does not apply to `graphify-auto/`.

## Topic Grouping (multi-part features)

When one area of `specs/` grows past ~5 notes, group them in a topic subfolder named for the feature (e.g. `specs/audit-report/`). This is the **only** place a second level of nesting is allowed: `specs/<topic>/`. Don't nest topics inside topics, and don't sub-folder `decisions/`, `knowledge/`, or `reference/` — those stay flat.

### Documenting a Multi-Part Feature

When capturing something made of many parts — report sections, API endpoints, pipeline stages, config flags, state-machine states — use **index + atomic parts**:

1. **One index note** holds the *overview* — the full list/order of parts and rules spanning all of them. It links to each part.
2. **One atomic note per part**, holding only that part's rules — so reading one part loads ~one part's tokens, not the whole feature.
3. **Cross-cutting concerns get their own note**, referenced by `[[wikilink]]` from each part that touches them — never duplicated inline.

Read the index first, then open only the part you need.

**Filename ordering** (so a human can browse the folder top-to-bottom):
- Ordered parts get a zero-padded numeric prefix: `<topic>-00-index.md`, then `<topic>-01-<part>.md … <topic>-NN-<part>.md` in order.
- Cross-cutting/supporting notes stay **unnumbered** (`<topic>-<name>.md`) so they group after the numbered parts.
- Result: `00 index → 01…NN parts in order → supporting notes`. If parts have no natural order, skip numbering.

> [!example] Pattern in practice
> `specs/audit-report/`: `audit-00-index` → `audit-01-risk … audit-13-unliquidated-advances` → unnumbered `audit-risk-floor`, `audit-report-modes` (shared rules). The completeness *decision* lives in `decisions/audit-completeness-fail-rule.md`, linked from the spec — not in the spec folder.

## Investigation Notes

Investigation notes are **permanent** — they are never deleted on resolution. Set `status: open` when the investigation is in progress and `status: resolved` (plus `resolved: <YYYY-MM-DD>`) when the root cause and fix are confirmed. Resolved investigations stay in `investigations/` so a recurrence can recall what happened.

**Titles and hub hooks** are phrased as the observable symptom (e.g., "Not allowed to split, check setting on GCash deposit"), not as a solution headline. This lets an agent match a new symptom to an existing investigation without knowing the root cause.

**Hub marker:** while an investigation is open, append `(open)` to its hub hook so it is visually distinct at a glance. Remove `(open)` — but keep the entry — when resolved.

**Named exception to the atomic rule:** an investigation note intentionally spans multiple sections (Symptom → Context → Ruled out → Root cause → Resolution). This is a deliberate chronological trail, not a topic sprawl — do not split it.

**Cross-link rule:** once a resolution yields a reusable lesson, extract that lesson into `knowledge/` or `decisions/` and link both ways — `[[knowledge/lesson]]` from the investigation and `See also: [[investigation-slug]]` from the knowledge note.

**Recall rule:** when a recurring issue is reported, go to the hub, scan `### Investigations` hooks for a matching symptom, and open the matched note before investigating from scratch.

### Canonical investigation note template

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

## Plan Notes

Plan notes carry a **lifecycle** so a reader can tell current intent from history. Frontmatter (extends the atomic note template):

```markdown
status: active                # active | done
completed:                    # <YYYY-MM-DD> when done
```

Set `status: active` when the plan is written and `status: done` (plus `completed: <YYYY-MM-DD>`) once it has been implemented. Done plans are **kept, not deleted** — an active plan answers "what are we building?"; a done plan answers "how was X built?".

**Hub marker:** append `(done)` to a completed plan's hub hook — keep the entry. Active plans stay unmarked, so the Plans subsection reads as current intent at a glance.

**Who flips it:** `/obsidian-audit` proposes the flip (in its confirm-before-saving step) when a session *explicitly* shows a captured plan was completed — implemented, shipped, or stated by the user. Completion is never inferred.

## Project Hub — `<project>/<project>.md`

```markdown
---
tags:
  - project
project: <project>
status: active            # active | paused | archived
repo: <absolute-path-to-code>
format_version: "2.2.0"   # which FORMAT.md (note structure) this project follows
setup_version: "1.3.0"    # which graphify-obsidian-setup.md the project's wiring was built against
---

# <project>

<One-paragraph summary.>

**Stack:** <tech · tech · tech>

## Notes

### Specs (`specs/`)
- [[<topic>-00-index]] — <hook>

### Decisions (`decisions/`)
- [[<slug>]] — <hook>

### Knowledge (`knowledge/`)
- [[<slug>]] — <hook>

### Reference (`reference/`)
- [[<slug>]] — <hook>

### Plans (`plans/`)
- [[<slug>]] — <hook>

### Investigations (`investigations/`)
- [[<slug>]] — <hook>

### Not covered yet
- [[<slug>]] — <deliberately deferred domains, listed so an agent knows their absence is intentional, not an oversight>

### Graph (`graphify-auto/`)
<!-- @generated — rebuilt by graphify post-commit hook -->
Auto-generated code graph. Query via `graphify query "<question>"` rather than reading individual nodes.
<!-- /@generated -->

## Key Paths

| Path | Purpose |
|------|---------|
| `<path>` | <purpose> |
```

Omit a Notes subsection that has no notes yet. `### Not covered yet` is **optional**: include it only when the project deliberately defers documenting known domains (it routes to a note listing them — the list itself is content and lives in the note, not the hub).

## Atomic Note — `<type-folder>/<slug>.md`

```markdown
---
tags:
  - <tag>                 # spec | decision | gotcha | pattern | api-quirk | bug | reference | plan | investigation
project: <project>
date: <YYYY-MM-DD>
format_version: "2.2.0"   # the current version of this file (keep in step with the hub template above)
---

# <Title>

See also: [[<project>]]

<The knowledge — concise. One concept. Use callouts for emphasis:>
> [!warning] Constraint
> <the rule that must not be broken>
```

## Rules

- **Atomic.** One concept per note. If a note grows two topics, split it. *Exception: investigation notes are intentionally multi-section (see [Investigation Notes](#investigation-notes)) — do not split them.*
- **Type-foldered.** Every note lives in the type folder matching its purpose (`specs/` `decisions/` `knowledge/` `reference/` `plans/` `investigations/`). Decisions never live inside a spec folder — link them instead. All type folders except `specs/` stay flat (no subfolders) — `investigations/` is flat like `decisions/`.
- **Hub is a router, kept lean (≤ ~400 words; large projects scale).** Baseline budget: ~400 words for a typical project. A project that routes many multi-part topics may add ~1 hub line per additional routed `…-00-index` (or topic subsection) beyond the baseline — more notes justify more *router lines*, never more content. The hard rule is size-independent: the hub holds hooks, never content. If a hub exceeds its budget, the excess belongs in a note (or a new `…-00-index`), not the hub.
- **High-signal hooks.** Every hub entry's hook states what the note *answers* (`[[note]] — <the question/fact it resolves>`), specific enough that an agent picks the right note without opening others. Vague hooks ("misc notes") defeat cheap recall.
- **Always linked.** Every note has a `See also: [[<project>]]` line and is listed under the matching hub subsection. No orphan notes.
- **Hub first.** The hub is the only entry point — read it, follow the single relevant `[[wikilink]]`. Never bulk-load a folder. Ignore `_Index_of_*` (auto-generated).
- **Capture, don't duplicate.** Don't store what's already in the repo's CLAUDE.md or git history.
- **Versioned.** Every note's `format_version` reflects the FORMAT it follows; bump on every format change.
  The **project hub** additionally carries `setup_version` — which [[graphify-obsidian-setup]] version the
  project's graphify/Obsidian wiring was built against. When either standard doc bumps (see [[VERSIONS]]),
  a hub whose `format_version`/`setup_version` is behind flags a project that needs migrating/re-wiring.

---

## Versioning

Semantic versioning `MAJOR.MINOR.PATCH`:
- **MAJOR** — breaking structural change (folder layout, required frontmatter renamed/removed). Requires migrating every project.
- **MINOR** — additive (new optional field, new note type/folder). Existing notes still valid; migrate when convenient.
- **PATCH** — wording/clarification only. No migration needed.

## Migrating

When this file's version changes:
1. Read the entry for the new version in **[[versions/format]]** — it lists the exact migration steps.
2. For each `Projects/<name>/` folder, find notes whose `format_version` is behind.
3. Apply the steps, then bump each migrated note's `format_version` to match.
4. A project is fully migrated when its hub and all notes match the current version.

## Changelog

### 2.5.0 — 2026-07-06
- **Plan notes gained a lifecycle:** frontmatter `status: active | done` plus `completed: <YYYY-MM-DD>` when done. Completed plans keep their hub entry with a `(done)` marker appended; active plans stay unmarked. `/obsidian-audit` proposes the flip on explicit session evidence only.
- Added the **Plan Notes** section documenting the lifecycle, hub marker, and flip rule.
- Additive — MINOR; existing plan notes gain `status:` when next touched, no immediate sweep.

### 2.4.0 — 2026-07-04
- **Hub budget now scales:** baseline ≤ ~400 words; hubs routing many multi-part topics may add ~1 router line per additional `…-00-index`/topic beyond the baseline. Hard rule unchanged: hooks, never content.
- Added optional **`### Not covered yet`** hub subsection — routes to a note listing deliberately deferred domains, so an agent knows absence is intentional.
- Additive/wording — MINOR; no structural change, migrate when convenient.

### 2.3.0 — 2026-07-02
- Added **`investigations/` human type folder** (tag: `investigation`): permanent issue-investigation notes recording symptom → hypotheses ruled out → root cause → resolution, so a recurring issue recalls how it was fixed. `status: open | resolved`; symptom-first titles and hub hooks; `(open)` marker for unresolved. Named exception to atomic rule. Distilled lessons still go to `knowledge/`/`decisions/` and cross-link.
- Added `### Investigations` subsection to Project Hub template.
- Additive — no structural change to existing type folders.

The full version history and per-version migration checklists live in **[[versions/format]]**.
The vault-wide version registry is **[[VERSIONS]]** (it tracks this file and [[graphify-obsidian-setup]]).
When you change the format here, bump `format_version` in the frontmatter, update the [[VERSIONS]] registry
row, then add the entry to [[versions/format]].
