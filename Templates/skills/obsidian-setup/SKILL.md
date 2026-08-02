---
name: obsidian-setup
description: Use when the user types /obsidian-setup, or asks to set up / wire Graphify + Obsidian onto the current repository, or "create the Obsidian folder/vault for this project", or to re-scan an already-wired codebase to refresh its derived notes. Per-project setup entry point — discovers the vault, installs the bundled skills + global directives, builds the graph, installs the commit hook, creates the project's vault folder, and scans the codebase to populate specs/, knowledge/, reference/ and the hub. NOT for capturing session knowledge from a work conversation — that's /obsidian-audit.
---

# Obsidian Setup — wire Graphify + Obsidian onto the current repo

One-word entry point to the full setup runbook. Run this **in the repository you want to wire**. You (the
agent) do all the work — discover, install, scaffold, and wire — and never hand the commands back to the user.
Only stop to ask when this skill says to (vault not found, or a missing prerequisite).

This skill is a thin wrapper: the authoritative, self-contained procedure is
`<vault-root>/Templates/graphify-obsidian-setup.md`. This file just gets you to it correctly.

## 1. Find the vault root — discover by content, never assume a path

The vault root is the folder that contains **both** `FORMAT.md` and a `Projects/` subfolder — identify it by
that content, not by its name. Resolve it in this order:

1. **Running Obsidian, CLI enabled:** `obsidian vault="Claude" eval code="app.vault.adapter.basePath"` (strip
   the leading `=> `; verify the folder has `FORMAT.md`). If the vault isn't named `Claude`, adjust `vault=` or
   fall through.
2. **Common locations:** `~/Obsidian/Claude`, `~/Desktop/Claude`, `~/Documents/Claude`, or this OS's
   equivalent — the first that holds `FORMAT.md` + `Projects/`.
3. **Bounded recursive search** under the user's home for a `FORMAT.md` sitting next to a `Projects/` folder.
4. **Still not found:** ask the user for the vault-root path. Never create a new vault or guess.

## 2. Check prerequisites

The user installs the software by hand (vault `README.md`). Verify these, and if any is missing, **stop and
point them at the README** — do not install it yourself:

- Obsidian app + the vault synced locally (Step 1 found it → OK)
- `uv` + `graphify` on PATH (`graphify --help` succeeds)
- `graphify install` has been run (the `/graphify` skill is registered)

## 3. Run the runbook

Read `<vault-root>/Templates/graphify-obsidian-setup.md` and follow it **end to end** — it is self-contained
and covers everything: installing the bundled skills + global directives (adapt to whichever agent you are),
tuning the ignores, building the graph, installing and patching the commit hook, creating the project hub +
folders, scanning the codebase to populate the vault (Step 8), and verifying.

Project name = this repo's folder name unless the user says otherwise.

**Re-scan only:** if the project is already wired and the user just wants the derived notes refreshed after the
codebase changed, run **Step 8 alone** — don't redo the graph build, hook, or skill installs.

## 4. Report

When done, tell the user: the vault folder created (`<vault-root>/Projects/<project>/` with its subfolders +
hub), each note written, hub fields filled, hook status, and anything skipped.
