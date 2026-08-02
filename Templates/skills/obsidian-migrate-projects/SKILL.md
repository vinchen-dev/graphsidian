---
name: obsidian-migrate-projects
description: Use when the user types /obsidian-migrate-projects, or when FORMAT.md or graphify-obsidian-setup.md has just been bumped and existing projects in Projects/ need bringing up to date. Scans every project hub, compares its format_version / setup_version to the registry, and applies the migration checklist to any that are behind — covering which files to touch per change type, what to skip, and how to verify. Do NOT use for structural vault changes (use obsidian-format-update for that first).
---

# Obsidian — Update / Migrate Existing Projects (`/obsidian-migrate-projects`)

Bring existing projects up to date after a vault **format** or **setup** version bump. Scan every project in
`Projects/`, compare each hub's `format_version` / `setup_version` against the registry in `VERSIONS.md`, and
apply the migration checklist to any hub that is behind. A hub already at target is done — **skip it**. If the
user is in one project's repo and only wants that one updated, target just its hub.

The migration checklist in `versions/format.md` (or `versions/setup.md`) says WHAT changed — this skill says HOW
to apply it without creating noise.

**REQUIRED BACKGROUND:** if the format/setup docs haven't been bumped yet, run `obsidian-format-update` first
(it changes the structural vault docs). This skill is the per-project follow-through.

## Step 0 — Locate the vault (discover by content, never a hardcoded path)

Every command below runs relative to the **vault root** — the folder that contains **both** `FORMAT.md` and a
`Projects/` subfolder. Resolve it once, then substitute it for `<vault>` everywhere:

1. **Running Obsidian, CLI enabled:** `obsidian vault="Claude" eval code="app.vault.adapter.basePath"` (strip
   the leading `=> `; verify the folder has `FORMAT.md`). If the vault isn't named `Claude`, adjust `vault=` or
   fall through.
2. **Else probe / search:** common spots (`~/Obsidian/Claude`, `~/Desktop/Claude`, `~/Documents/Claude`), or a
   bounded recursive search under the user's home for a `FORMAT.md` sitting next to a `Projects/` folder.
3. **Else ask the user.** Never guess or create a new vault.

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
- ❌ Do NOT re-run full setup (`/obsidian-setup`) on an existing project unless the changelog says to — apply the discrete migration steps instead

**Both bumped at once (e.g., format 2.3.0 + setup 1.4.0):**
- Apply both sets of rules in one pass per hub — one edit touches both `format_version` and `setup_version`

## Steps

### 1. Read the migration checklists

Get the target versions from the registry (the `## Registry` table lists the FORMAT + setup targets):

> **bash**
```bash
grep -E "FORMAT|graphify-obsidian-setup" "<vault>/VERSIONS.md"
```
> **PowerShell**
```powershell
Select-String -Path "<vault>\VERSIONS.md" -Pattern 'FORMAT|graphify-obsidian-setup'
```

Open `<vault>/versions/format.md` and `<vault>/versions/setup.md`, read the **top entry's Migration checklist**.
That list is the contract — do exactly what it says, no more.

### 2. Discover all projects

> **bash**
```bash
ls "<vault>/Projects/"
```
> **PowerShell**
```powershell
Get-ChildItem "<vault>\Projects" -Directory | Select-Object -ExpandProperty Name
```

### 3. Per project: check current versions

Read the hub (`<vault>/Projects/<project>/<project>.md`) frontmatter and compare to the targets from Step 1:
```yaml
format_version: "X.Y.Z"   # compare to FORMAT.md target
setup_version:  "X.Y.Z"   # compare to setup target
```

A hub already at both target versions is done — **skip it**. A hub at an intermediate version still needs every
migration between its version and the target, applied in order.

### 4. Per project: apply the migration checklist

Work through the checklist items in order. For a MINOR FORMAT bump + setup bump the pattern is:

```
a. create Projects/<project>/<new-folder>/          (if the checklist requires it)
b. Add ### <Type> subsection to hub ## Notes         (if the checklist requires it)
c. Edit hub frontmatter: bump format_version + setup_version in one edit
```

> **bash** — create a required new folder
```bash
mkdir -p "<vault>/Projects/<project>/<new-folder>/"
```
> **PowerShell**
```powershell
New-Item -ItemType Directory -Force "<vault>\Projects\<project>\<new-folder>" | Out-Null
```

Do NOT edit notes unless the MAJOR migration checklist explicitly names them.

### 5. Verify

Substitute the real target numbers for `X.Y.Z` / `A.B.C`:

> **bash**
```bash
# Hubs NOT yet at target format_version (should print nothing when all are migrated)
grep -rL 'format_version: "X.Y.Z"' "<vault>"/Projects/*/*.md
# Hubs NOT yet at target setup_version
grep -rL 'setup_version: "A.B.C"' "<vault>"/Projects/*/*.md
# New folders exist (adjust folder name)
ls -d "<vault>"/Projects/*/<new-folder>/ 2>/dev/null
```
> **PowerShell**
```powershell
# Hubs still behind on format_version (should list none)
Get-ChildItem "<vault>\Projects\*\*.md" | Where-Object { -not (Select-String -Path $_ -Pattern 'format_version: "X.Y.Z"' -Quiet) }
# New folders exist (adjust folder name)
Get-ChildItem "<vault>\Projects\*\<new-folder>" -Directory -ErrorAction SilentlyContinue
```

Then follow the vault's git rule — leave `.md` changes **staged** for the user (`git add`), don't commit.

## Common Mistakes

| Mistake | Why it's wrong | What to do instead |
|---------|---------------|-------------------|
| Bumping `format_version` on all notes for a MINOR change | Creates a massive noisy diff; notes already comply | Bump hub only; notes pick up the new version when next edited |
| Forgetting `setup_version` on hubs | Hub shows stale setup, triggering false "needs re-wiring" audits | Always check both version fields on the hub |
| Re-running full setup (`/obsidian-setup`) on existing projects | Recreates the hub from template → overwrites its content | Only run full setup on new projects; apply discrete migration steps to existing ones |
| Creating the `<type>/` folder without adding the hub subsection | Hub has no `### <Type>` entry → recall protocol misses it | Always do both together |
| Skipping a project at an intermediate version | A hub at 2.2.0 with target 2.3.0 still needs the 2.3.0 migration | Check each hub's actual version; apply each skipped migration in order |
| Using a hardcoded vault path | Breaks on machines where the vault lives elsewhere | Resolve `<vault>` by content (Step 0) before running any command |
