---
tags:
  - meta
  - versioning
updated: 2026-07-02
---

# Vault Document Versions

Single source of truth for the version and change history of the vault's **standardized documents**.
Each is versioned **independently** with semantic versioning. When you change one of these docs: bump its
version in its own frontmatter, update the [Registry](#registry) row, and add a changelog entry under its
section below.

> **Scope.** This tracks the standard *documents themselves*. It does **not** track per-project note
> `format_version` — that field on a note records which [[FORMAT]] version the note follows, and [[FORMAT]]
> remains the authority for note-migration mechanics.

## Registry

| Document | Version | Updated | Purpose |
|----------|---------|---------|---------|
| [[FORMAT]] | 2.3.0 | 2026-07-02 | Per-project note structure standard — folders, frontmatter, hub, recall protocol |
| [[graphify-obsidian-setup]] | 1.4.0 | 2026-07-02 | AI procedure to wire graphify + Obsidian onto a project |

> [[How to Setup]] is the human-facing mirror of [[graphify-obsidian-setup]] — keep the two in step. It
> carries no independent version; its frontmatter `mirrors_setup` records which setup `doc_version` it's in
> step with (currently `1.3.0`). If `mirrors_setup` lags the registry's setup version, the human guide is
> behind the AI template and needs reconciling.

## How projects reference these versions

Every **project hub** (`Projects/<project>/<project>.md`) stamps two fields in its frontmatter so you can tell
at a glance what's behind:

| Field | Points at | Bumped when | A stale value means |
|-------|-----------|-------------|---------------------|
| `format_version` | [[FORMAT]] version | the project's notes are migrated to a new FORMAT | notes need migrating ([[FORMAT]] → *Migrating*) |
| `setup_version` | [[graphify-obsidian-setup]] `doc_version` | the project's graphify/Obsidian wiring is rebuilt to a new setup | the wiring (hooks, export, layout) may need re-applying |

To audit the whole vault: compare each hub's two fields against the [Registry](#registry) above — any hub
behind on either number is a project to update. `setup_version` lives **only on the hub** (wiring is
per-project), while `format_version` lives on the hub *and* every note (structure is per-note).

## Semantic versioning

`MAJOR.MINOR.PATCH`, applied per document:
- **MAJOR** — breaking change. FORMAT: a layout or required-frontmatter change that forces migrating every
  project. Setup: a step change that invalidates existing wiring.
- **MINOR** — additive (new optional field, new note type/folder, new setup step). Existing work stays valid.
- **PATCH** — wording / clarification only. No migration.

---

## Changelogs

Per-document history lives in its own file (kept out of this registry so it stays lean):

- [[versions/format]] — version history + per-version migration checklists for [[FORMAT]].
- [[versions/setup]] — version history for [[graphify-obsidian-setup]].

When you change a doc: bump its version (in its frontmatter), update the [Registry](#registry) row above,
and add an entry to that doc's changelog file.
