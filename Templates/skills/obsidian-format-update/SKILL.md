---
name: obsidian-format-update
description: Use when adding, renaming, or removing a note type/folder from this Obsidian vault (FORMAT.md changes) — covers every file that must change, the version-bumping protocol, and the per-project migration steps. Do NOT use for one-off note creation or graphify wiring changes.
---

# Obsidian Vault Format Update

Adding a new note type touches more files than it looks like. Agents without this skill consistently miss the version changelogs, the init script, the setup docs, both skill copies, and per-project hub updates — while inventing changes to files that don't need touching (`_Index_of_*`, `.obsidian/` plugins, root-level directories).

## When to Use

- Adding a new `type-folder/` with a new tag to the vault
- Removing or renaming an existing type folder
- Changing the Project Hub template structure
- Any other FORMAT.md structural change

**Not for:** note creation, graphify wiring, `obsidian-audit` capture runs.

## Version Protocol

| Change | Bump | Which doc |
|--------|------|-----------|
| New/removed folder, required frontmatter change | MINOR | FORMAT.md |
| Init script now creates/removes a folder | MINOR | graphify-obsidian-setup.md |
| Wording/clarification only | PATCH | whichever doc changed |
| Breaking structural change | MAJOR | FORMAT.md |

Always bump FORMAT.md. Only bump setup version if `graphify-obsidian-init` changes.

## Complete File Checklist

Work top-to-bottom. Do not skip items.

### 1. FORMAT.md ← start here
- Bump `format_version` + `updated` in frontmatter; update `# Project Format — vX.Y.Z` title
- Folder Layout block: add/remove the folder entry
- Note Types & Folders table: add/remove row (`folder | purpose | tag | ask`)
- Add a `## <Type> Notes` section with: permanence rules, hub-hook conventions, canonical note template, cross-link rules
- Project Hub template: add/remove the `### <Type> (\`<folder>/\`)` subsection
- Rules section: update "Atomic." exception if needed; update "Type-foldered." folder list; update nesting rule
- Atomic note tag comment: add/remove the new tag
- Changelog section: add new version entry at the top

### 2. VERSIONS.md
- Registry table: bump FORMAT row (and setup row if init script changes)
- Bump frontmatter `updated`

### 3. versions/format.md
- Bump frontmatter `updated`
- Add changelog entry at the TOP:
  ```markdown
  ### X.Y.Z — YYYY-MM-DD
  - Added/removed `<folder>/` …
  - Additive / Breaking — …

  **Migration (X.Y.Z):**
  - [ ] <step>
  - [ ] Bump format_version to "X.Y.Z" on hubs/notes when convenient.
  ```

### 4. versions/setup.md (only if init script changed)
- Bump frontmatter `updated`
- Add changelog entry at the TOP (same pattern as above)

### 5. README.md
- Structure tree: add/remove the folder line
- "Notes separated by type" principle bullet: update folder list

### 6. Templates/bin/graphify-obsidian-init (only if folder is init-created)
- `mkdir -p "$VAULT"/{…,<new-folder>}` line
- Hub heredoc `## Notes`: add/remove the `### <Type>` subsection
- **Stage this file** (`git add Templates/bin/graphify-obsidian-init`) — it is a shell script, not a .md file

### 7. Templates/graphify-obsidian-setup.md (only if init script changed)
- Bump `doc_version` + `updated` in frontmatter
- Add `<folder>/` to any folder list

### 8. Templates/How to Setup.md (only if setup doc version changed)
- Bump `mirrors_setup` frontmatter to match setup doc version
- Add `<folder>/` to any folder list

### 9. Both obsidian-audit SKILL.md copies — apply identically
`~/.claude/skills/obsidian-audit/SKILL.md`
`Templates/skills/obsidian-audit/SKILL.md`

For each:
- Description: add trigger if the new type needs recall/capture behavior
- Two-Layer table vault row: add folder to list
- Recall Protocol: add step if recall behavior changes
- What to Save / Skip: update if capture rules differ for this type
- Workflow §3 type table: add row + canonical template + rules
- Workflow §4: add hub subsection guidance
- File Path Reference table: add row

After editing both: `diff` them — they must be identical.

### 10. ~/.claude/CLAUDE.md (only if a new recall trigger is needed)
- Add a `# <Section>` directive for the new behavior

### 11. Per-project folders and hubs
For each existing project in `Projects/`:
- `mkdir -p Projects/<project>/<new-folder>/`
- Add `### <Type> (\`<folder>/\`)` subsection to `## Notes` in `Projects/<project>/<project>.md` (in the same position as in the FORMAT hub template)

## Verification

```bash
# Every doc that lists type folders now includes the new one
grep -l "<new-folder>/" FORMAT.md README.md VERSIONS.md versions/format.md \
  Templates/skills/obsidian-audit/SKILL.md ~/.claude/skills/obsidian-audit/SKILL.md

# Version numbers landed
grep "X.Y.Z" FORMAT.md VERSIONS.md versions/format.md

# Init script (if changed)
grep "<new-folder>" Templates/bin/graphify-obsidian-init

# Skill copies are identical
diff ~/.claude/skills/obsidian-audit/SKILL.md Templates/skills/obsidian-audit/SKILL.md

# Per-project dirs exist
ls Projects/*/\<new-folder\>/

# Only shell script staged; .md files unstaged
git status --short
```

## What NOT to Touch

These files look relevant but are not:
- `_Index_of_*` files — auto-generated, never edit manually
- `.obsidian/` plugin configs — unrelated to note type taxonomy
- `graphify-auto/` — machine-generated, rebuilt by hook
- Root-level `<folder>/` — type folders live under `Projects/<project>/`, not the vault root
- `Templates/<folder>/` — there are no per-type template folders; the canonical template lives in the SKILL.md and FORMAT.md
