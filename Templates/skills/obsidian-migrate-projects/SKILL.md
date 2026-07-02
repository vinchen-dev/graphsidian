---
name: obsidian-migrate-projects
description: Use when FORMAT.md or graphify-obsidian-setup.md has just been bumped and existing projects in Projects/ need migrating to the new version — covers which files to touch per change type, what to skip, and how to verify. Do NOT use for structural vault changes (use obsidian-format-update for that first).
---

# Obsidian — Migrate Existing Projects

After a vault format or setup version bump, apply the migration to every project in `Projects/`. The migration checklist in `versions/format.md` (or `versions/setup.md`) says what changed — this skill says how to apply it without creating noise.

**REQUIRED BACKGROUND:** Run `obsidian-format-update` first (structural vault docs). This skill handles the per-project follow-through.

## Critical Rules — Read Before Touching Anything

**MINOR (additive) FORMAT change — new folder/tag added:**
- ✅ Bump `format_version` on **hubs only** (`<project>.md`)
- ✅ Create the new `<folder>/` dir in each project
- ✅ Add the new `### <Type>` subsection to each hub's `## Notes`
- ❌ Do NOT bulk-bump `format_version` on individual notes — they already follow the format correctly; bumping them is noise until the note itself is next edited

**MAJOR FORMAT change — folder renamed, frontmatter restructured:**
- ✅ Bump `format_version` on **hubs AND every affected note**
- ✅ Apply structural changes to each affected note (rename, move, reformat)
- ✅ Create/remove folders as directed by the migration checklist

**Setup version bump (graphify-obsidian-setup.md):**
- ✅ Bump `setup_version` on **hubs only** — notes don't carry `setup_version`
- ✅ Apply any new wiring steps the changelog lists (re-run hook, new ignore file, etc.)
- ❌ Do NOT re-run `graphify-obsidian-init` on an existing project unless the changelog says to

**Both bumped at once (e.g., format 2.3.0 + setup 1.4.0):**
- Apply both sets of rules in one pass per hub — one edit touches both `format_version` and `setup_version`

## Steps

### 1. Read the migration checklists

```bash
# Get the target versions
grep "format_version\|doc_version" /Users/vince/Obsidian/Claude/VERSIONS.md
```

Open `versions/format.md` and `versions/setup.md`, read the top entry's **Migration** checklist. That list is the contract — do exactly what it says, no more.

### 2. Discover all projects

```bash
ls /Users/vince/Obsidian/Claude/Projects/
```

### 3. Per project: check current versions

Read the hub (`Projects/<project>/<project>.md`) frontmatter:
```yaml
format_version: "X.Y.Z"   # compare to FORMAT.md target
setup_version:  "X.Y.Z"   # compare to setup target
```

A hub already at target version is done — skip it.

### 4. Per project: apply the migration checklist

Work through the checklist items in order. For a MINOR FORMAT bump + setup bump the pattern is:

```
a. mkdir -p Projects/<project>/<new-folder>/          (if checklist requires it)
b. Add ### <Type> subsection to hub ## Notes          (if checklist requires it)
c. Edit hub frontmatter: bump format_version + setup_version in one edit
```

Do NOT edit notes unless the MAJOR migration checklist explicitly names them.

### 5. Verify

```bash
# All hubs at new format_version
grep -r "format_version" /Users/vince/Obsidian/Claude/Projects/*/\*.md | grep -v "2\.3\.0"
# (should return nothing if all hubs are updated; adjust version number)

# All hubs at new setup_version  
grep -r "setup_version" /Users/vince/Obsidian/Claude/Projects/*/\*.md | grep -v "1\.4\.0"

# New folders exist
ls Projects/*/investigations/ 2>/dev/null

# .md files unstaged (per vault git rules)
git status --short
```

## Common Mistakes

| Mistake | Why it's wrong | What to do instead |
|---------|---------------|-------------------|
| Bumping `format_version` on all 200+ notes for a MINOR change | Creates massive noisy diff; notes already comply | Bump hub only; notes pick up the new version when next edited |
| Forgetting `setup_version` on hubs | Hub will show stale setup, triggering false "needs re-wiring" audits | Always check both version fields on the hub |
| Re-running `graphify-obsidian-init` on existing projects | Overwrites hub content | Only run on new projects; apply discrete steps manually for existing ones |
| Creating the `<type>/` folder without adding the hub subsection | Hub has no `### <Type>` entry → recall protocol misses it | Always do both together |
| Skipping projects already at an intermediate version | If hub is at 2.2.0 and target is 2.3.0, it still needs the 2.3.0 migration | Check each hub's actual version; apply each skipped migration in order |
