---
tags:
  - meta
  - howto
format_version: "2.2.0"
mirrors_setup: "1.11.0"     # human mirror of graphify-obsidian-setup.md (this doc has no independent version; keep in step)
updated: 2026-09-30
---

# How to Setup — Graphify + Obsidian on a New Project

> [!tip] Two ways to run this
> - **Machine already set up?** Just open your agent in the new repo and type **`/obsidian-setup`** — the bundled skill runs everything below. This is the fast path for every project after your first.
> - **First project on this machine, or an agent without skills?** Paste [the prompt](#the-prompt) below — it bootstraps everything, including installing the `/obsidian-setup` skill for next time.

**One prompt. Give it to any agent (Claude Code, Codex, …) opened in the repo you want to wire.** The agent
reads the self-contained runbook [[graphify-obsidian-setup]] and does everything: installs the bundled vault
skills, adds the global directives, discovers the vault, builds the graph, wires the commit hook, creates the
project hub, and populates it. You don't run a pile of commands — you paste this and let the agent work.

> [!important] One-time prerequisites first
> The agent does **not** install Obsidian or Graphify. Do the manual software install once, per machine, from
> the vault's **`README.md`** (install Obsidian + CLI, install `uv` + `graphify`, run `graphify install`). If
> those aren't done, the agent will stop and point you back to the README.

---

## The prompt

Open **any** agent in the project repo (`claude`, `codex`, or another), then paste this **verbatim** — it works
no matter which agent you're using or where your vault lives, and you don't need to know the vault path:

```
Set up Graphify + Obsidian for THIS repository. Do all of it yourself — discover, install, scaffold, and
wire — don't hand commands back to me.

1. Find my Obsidian vault root: the folder that contains BOTH a `FORMAT.md` file and a `Projects/` subfolder
   (it may be named anything — identify it by that content, not by its name). Look in this order: (a) if
   Obsidian is running with its CLI, ask it for the vault path; (b) common spots for this OS —
   ~/Obsidian/Claude, ~/Desktop/Claude, ~/Documents/Claude; (c) a recursive filesystem search under my home
   for a `FORMAT.md` sitting next to a `Projects/` folder; (d) if all of those fail, ask me. Never create a
   new vault or guess a path.

2. Read `<vault-root>/Templates/graphify-obsidian-setup.md` and follow it end to end. It is self-contained
   and tells you exactly which skills + global directives to install and how to wire this project.

3. Adapt to whichever agent you are: install the skills and directives into YOUR own user-level locations
   (Claude Code → ~/.claude/…, Codex → ~/.codex + ~/.agents/skills, or another agent's equivalent). If you
   have no skill mechanism, tell me the exact files to add instead.

Project name: this repo's folder name unless I say otherwise. Prerequisites (Obsidian, uv, graphify,
`graphify install`) are installed per the README — if one is missing, stop and point me at the README. When
done, report what you created and anything you skipped.
```

The prompt is self-bootstrapping: the agent discovers the vault by **content** (`FORMAT.md` + `Projects/`), so
it works on any machine and any vault location, and it adapts to whichever agent runs it.

---

## What the agent will do (so you know what "done" looks like)

Driven entirely by [[graphify-obsidian-setup]]:

1. **Resolve the vault root** and verify the README prerequisites are installed.
2. **Install the six bundled vault skills** (`obsidian-setup`, `obsidian-audit`, `obsidian-recall`,
   `obsidian-format-update`, `obsidian-migrate-projects`, `obsidian-maintenance`) into the agent's user skill
   directory, and ensure the global graph-first + trigger directives are present. Idempotent —
   already-installed pieces are skipped.
   (After this, `/obsidian-setup` is available for your next project.)
3. **Wire the current project**: gitignore + `.graphifyignore`, build the graph (`/graphify`), install and patch
   the post-commit hook so it re-exports into Obsidian, create the project hub, and scan the codebase to
   populate `specs/` `knowledge/` `reference/` and fill the hub (it asks for approval before writing notes).
4. **Verify** the hook fires and the vault export lands, then report back.

---

## After setup — daily use

| You want… | Do this |
|-----------|---------|
| "What calls X? / trace flow through Y" | `graphify query "<question>"` (in the repo) |
| Keep the graph fresh (your own work) | nothing — every commit auto-updates it (zero tokens) |
| Resync after `git pull` (teammate's code) | `graphify update .` — free AST rebuild |
| Refresh after editing **docs** (`.md`) | run `/graphify` again (code is automatic; doc concepts aren't) |
| Persist a decision / gotcha / plan | `/obsidian-audit` |
| Recall before debugging / deciding | `/obsidian-recall` |
| Browse the graph | open `graphify-out/graph.html`, or `graphify-auto/` in Obsidian |
| Add a new note type / folder to the vault format | invoke the `obsidian-format-update` skill |
| Bring existing projects up to date after a format/setup version bump | `/obsidian-migrate-projects` |
| Check whether a project's notes are still true (weekly/monthly) | `/obsidian-maintenance` |
