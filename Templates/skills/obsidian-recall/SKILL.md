---
name: obsidian-recall
description: Use when about to answer a question, make a decision, or debug something in a project that has a vault hub — before re-deriving from scratch, check whether it was already decided, investigated, or documented. Triggers on "why did we choose X", "have we seen this bug/error before", "what's the plan for Y", "check the vault", or any recurring-issue report. Also handles explicit /obsidian-recall invocation. Do NOT use this to write or capture new knowledge — that's obsidian-audit.
---

# Obsidian Recall (Read from Second Brain)

Look up durable knowledge already captured in the Obsidian second-brain vault instead of re-deriving it, re-debugging it, or asking the user to repeat themselves.

## Locating the vault

Resolve the vault path at runtime from the running Obsidian instance — never from an env var or a remembered path (the vault can move; the CLI always knows where it is):

```bash
obsidian vault="Claude" eval code="app.vault.adapter.basePath"
```

The returned absolute path is `<vault>` in everything below. If the command fails, Obsidian isn't running — tell the user to open Obsidian and stop; do not guess the path.

## Core Principle

The vault's whole layout — hub + atomic notes with high-signal hooks — exists so recall costs **one hub read + one note read**, regardless of project size. Never explore the vault like a filesystem. If the hub doesn't answer it, the vault doesn't have it — say so and move on.

## Protocol

1. **Identify the project.** Derive from cwd the same way `obsidian-audit` does (`~/Desktop/Projects/finance-ai` → `finance-ai`). Check `<vault>/Projects/<project>/<project>.md` exists.
   - If cwd doesn't map to a project folder, or the question names a different project than cwd, scan the project list in `<vault>/Home.md` to find the right one before giving up.
   - If no hub exists for the project at all: stop, tell the user there are no vault notes for it yet. Don't search further.
2. **Read only the hub.** Its `## Notes` section is a router — every note is listed as `[[note]] — hook`, grouped by type (Specs / Decisions / Knowledge / Reference / Plans / Investigations). The hook states what the note answers.
3. **Open exactly one note** whose hook matches the question. For a multi-part topic, open its `…-00-index` (a sub-router) first, then the one part actually needed.
4. **Never** read a type folder wholesale, never open `_Index_of_*` files (auto-generated noise, not content), never grep the vault when the hub already answers it.

**Recurring issue / bug report:** go straight to `### Investigations` in the hub, scan hooks for a matching symptom, open the matched note before investigating from scratch. Resolved investigations (`status: resolved`) still carry the root cause and fix — they aren't deleted, so a recurrence recalls what happened last time.

**Code-structure questions** ("what calls X", "trace the flow through Y") are out of scope here — query the graph instead: `graphify query "<question>"`. The hub→note protocol doesn't apply to `graphify-auto/` nodes.

## When the hub doesn't have it

Report plainly: "No vault note covers this" (or "closest is `[[note]]` but it's about Z, not this"). Do not pad the answer with reconstructed or inferred content to seem more helpful — a wrong recall is worse than no recall.

## Quick Reference

| Situation | Action |
|---|---|
| "Why did we pick X over Y?" | Hub → `### Decisions` → matching hook |
| "Have we hit this error before?" | Hub → `### Investigations` → matching symptom |
| "What's the plan for Z?" | Hub → `### Plans` → matching hook |
| "How does feature F behave?" | Hub → `### Specs` → matching hook (or `…-00-index` if multi-part) |
| "What calls/uses X in the code?" | Not this skill — `graphify query` instead |
| No hub for this project | Stop, report no notes exist — don't search elsewhere |
