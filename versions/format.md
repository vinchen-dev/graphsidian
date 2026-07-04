---
tags:
  - meta
  - changelog
tracks: FORMAT
updated: 2026-07-04
---

# FORMAT.md — changelog

Version history and per-version migration checklists for [[FORMAT]]. Registry: [[VERSIONS]].
[[FORMAT]]'s *Migrating* section points here.

### 2.4.0 — 2026-07-04
- **Hub budget scales for large projects:** baseline ≤ ~400 words; +~1 router line per additional routed `…-00-index`/topic beyond the baseline. Hard rule unchanged — the hub holds hooks, never content.
- Added optional `### Not covered yet` hub subsection (between Investigations and Graph): routes to a note listing deliberately deferred domains.
- Additive/wording — no structural change to folders or required frontmatter.

**Migration (2.4.0):**
- [ ] Nothing mandatory. Hubs already using `### Not covered yet` (PMS2.0_PH) are now standard-compliant as-is.
- [ ] Bump `format_version` to `"2.4.0"` on hubs/notes when convenient (obsidian-migrate-projects handles this later).

### 2.3.0 — 2026-07-02
- Added **`investigations/` human type folder** (tag: `investigation`): permanent issue-investigation notes recording symptom → hypotheses ruled out → root cause → resolution, so a recurring issue recalls how it was fixed. `status: open | resolved`; symptom-first titles and hub hooks; `(open)` marker for unresolved. Named exception to atomic rule. Distilled lessons still go to `knowledge/`/`decisions/` and cross-link.
- Added `### Investigations` subsection to Project Hub template.
- Additive — no structural change to existing type folders.

**Migration (2.3.0):**
- [ ] Nothing mandatory. Create `investigations/` when a project's first investigation lands.
- [ ] Bump `format_version` to `"2.3.0"` on hubs/notes when convenient.

### 2.2.0 — 2026-06-24
- Added **`plans/` human type folder** (tag: `plan`): implementation plans, feature plans, roadmaps. Distinct from `specs/` (as-built behavior) and `decisions/` (why we chose X).
- Added **`graphify-auto/` machine-generated folder**: post-commit auto-rebuilt code graph, sentinel convention (`<!-- @generated -->`/`<!-- @user -->`), and recall guidance (use `graphify query` — not vault recall — for code-structure questions).
- Added `### Plans` and `### Graph` sections to the Project Hub template.
- Additive — no structural change to existing type folders.

**Migration (2.2.0):**
- [ ] Add `plans/` folder to any project with implementation plans to store.
- [ ] Add `### Plans` subsection to hub `## Notes` (omit if empty).
- [ ] Add `### Graph` with `<!-- @generated -->` sentinel to hub if project uses graphify.
- [ ] Bump `format_version` to `"2.2.0"` on hub and all notes.

### 2.1.0 — 2026-06-22
- Added the **AI Agent Access (recall) protocol** — the minimal, no-exploration read path (1 hub + 1 note) an agent follows.
- Added rules that keep recall cheap: **hub is a lean router (≤ ~400 words)**, **high-signal hooks** (each says what its note answers), ignore `_Index_of_*`.
- Additive — no structural change.

**Migration (2.1.0):**
- [ ] Trim any hub over ~400 words (move content into notes). Sharpen vague note hooks to say what each note answers.
- [ ] Bump `format_version` to `"2.1.0"`.

### 2.0.0 — 2026-06-22
- **Type folders.** Notes are now separated by purpose into `specs/`, `decisions/`, `knowledge/`, `reference/` (was: flat notes + ad-hoc topic folders). Added the Note Types & Folders taxonomy.
- Multi-part topics now live under `specs/<topic>/`. `decisions/`, `knowledge/`, `reference/` stay flat.
- Tag set aligned to folders: `spec`, `decision`, `gotcha`, `pattern`, `api-quirk`, `bug`, `reference`.
- Hub `## Notes` is now grouped by the four type subsections.

**Migration (2.0.0):**
- [ ] Create the type folders that apply; move each note into the folder matching its purpose.
- [ ] Move multi-part topic folders under `specs/` (e.g. `audit-report/` → `specs/audit-report/`).
- [ ] Retag notes to the aligned tag set (e.g. a config/endpoint note tagged `api-quirk` → `reference`).
- [ ] Regroup the hub `## Notes` into Specs / Decisions / Knowledge / Reference subsections.
- [ ] Wikilinks resolve by basename, so moves don't break links — only update links when renaming.
- [ ] Bump `format_version` to `"2.0.0"` on every migrated note + the hub.

### 1.4.0 — 2026-06-22
- Added **Decisions vs Spec**: stakeholder/business decisions live in a project `decisions/` folder (tagged `decision`), separate from feature spec folders, and are linked from the spec note they affect.

### 1.3.0 — 2026-06-22
- Added **filename ordering** for multi-part topics: zero-padded numeric prefixes (`00-index`, `01..NN`); cross-cutting notes stay unnumbered and group at the end.

### 1.2.0 — 2026-06-22
- Added the **Documenting a Multi-Part Feature** pattern (index + atomic parts + shared cross-cutting notes).

### 1.1.0 — 2026-06-22
- Added **Topic Grouping**: a project area with many notes can live in a one-level subfolder with its own index note.

### 1.0.0 — 2026-06-22
- Initial format: `Projects/<project>/` with a hub + flat atomic notes; frontmatter + `See also:` link rule; semantic versioning + migration process.

<!--
TEMPLATE for future entries:

### X.Y.Z — YYYY-MM-DD
- <what changed>

**Migration (X.Y.Z):**
- [ ] <step per affected note type>
- [ ] Bump format_version to "X.Y.Z" on migrated notes + the Registry row in VERSIONS.md
-->
