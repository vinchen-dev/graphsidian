---
name: obsidian-maintenance
description: Use ONLY when the user types /obsidian-maintenance — a per-project maintenance sweep that re-checks whether a project's specs/reference/knowledge notes still match the CURRENT code (graph-first via graphify, source-confirmed) and proposes fixes for drifted or obsolete notes. Runs on explicit /obsidian-maintenance only; intended weekly/monthly. Do NOT use for capturing new knowledge (obsidian-audit), recall/pre-debug lookup (obsidian-recall), or version migration (obsidian-migrate-projects).
---

# Obsidian Maintenance (Verify Notes Against Code)

Notes rot quietly. A `specs/` note describes behavior the code later changed; a `knowledge/` gotcha was fixed; a `reference/` env var was renamed. This skill re-checks **one project's** current-state notes against the live codebase and **proposes corrections** — so the vault stays trustworthy instead of drifting into lies.

Run it as a deliberate maintenance sweep (weekly or monthly), from **inside the project repo**. It never rewrites a note on its own — it proposes, you confirm.

## Locating the vault

Resolve the vault path at runtime — never from an env var or a remembered path (the vault can move). The vault root is the folder containing **both** `FORMAT.md` and a `Projects/` subfolder; identify it by that content, not by its name. Try in order, verifying `FORMAT.md` exists before using any result:

1. **Running Obsidian, CLI enabled** (most current — survives a moved vault):
   ```bash
   obsidian vault="Claude" eval code="app.vault.adapter.basePath"
   ```
   Strip the leading `=> `. If the vault isn't named `Claude`, adjust `vault=` or fall through.
2. **Common locations:** `~/Obsidian/Claude`, `~/Desktop/Claude`, `~/Documents/Claude` — first one holding `FORMAT.md` + `Projects/` wins.
3. **Bounded recursive search** under the user's home for a `FORMAT.md` sitting beside a `Projects/` folder.
4. **Still nothing — ask the user** for the vault-root path. Never guess, and never create a new vault.

The resolved absolute path is `<vault>` in everything below.

## Scope — what the sweep checks (and what it deliberately doesn't)

Only **current-state** notes, whose truth is a claim about the code *as it is now*:

- `specs/` — behavior as built
- `reference/` — env vars, endpoints, collections, config
- `knowledge/` — gotchas, patterns, API quirks, bug root causes

**Out of scope** — historical records, not current-code claims; checking them against code is a category error:

- `decisions/` — *why* we chose X (stays true even after the code changes)
- `plans/` — intent (active / done)
- `investigations/` — resolved symptom → root cause → fix (historical)

## Preconditions

The sweep checks against the **graph** and the **source**, so both must be present and current:

1. **You are inside the project's repo** — `cwd` maps to a `<vault>/Projects/<project>/<project>.md` hub the same way `obsidian-audit`/`obsidian-recall` derive it (`~/Desktop/Projects/finance-ai` → `finance-ai`). Per-project only; if `cwd` doesn't map to a hub, stop and say so.
2. **`graphify-out/` exists and is current.** The post-commit hook keeps it fresh; if the repo was just `git pull`ed or the graph looks stale, run `graphify update .` (0-token AST rebuild) first. **Never verify against a stale graph — it manufactures false verdicts.**

If the repo or graph isn't available on this machine, **stop** — don't guess a verdict from the note text alone.

## How the sweep works — route each claim by type

The graph is a code-**structure** map (entities + edges), **not a behavior/value oracle**. So route each claim to the tool that can actually settle it — graph-first, reading source only where the graph is blind, and never crawling the repo.

**1. Existence (any note type) — the graph settles it cheaply.**
For each code anchor a note names (`classifyReceipt`, `attachmentPath.js`, a route, a class):
`graphify explain "<anchor>"` — returns nothing → the entity is gone → **❌ obsolete**, without reading a line.

**2. Structure / flow claims (mostly `specs/`) — the graph settles it.**
"A calls B", "X feeds Y", pipeline / call-graph claims:
`graphify path "A" "B"`, `graphify explain "A"`, `graphify affected "X"` — a missing edge or a changed dependency set is a **⚠️ drifted** candidate.

**3. Behavior / value / threshold claims (most `knowledge/`) — the graph CANNOT settle it; read source.**
"OCR confidence 70", "truncates at 8000 chars", "returns 400 on `temperature`", "truthy-wins merge drops `is_receipt:false`". The graph can't see a constant or a branch. Use `graphify query "<the claim>"` (or `explain`) only to **locate** the implementing code cheaply, then **read the pinpointed `file:line`** and compare the note's stated value/behavior against what the code actually does now.

**4. String-literal `reference/` claims — don't assume the graph indexes them.**
Env var names (`MONGODB_URI`), collection names (`transactions`), endpoints: verify by searching source for the literal (`grep`/read), since these are often not graph nodes.

**5. External-system claims — not code-checkable.**
Lark error codes, bank UI behavior, third-party gateway quirks have no local code to check them against. Verdict **🔗 not code-checkable** → route to the user. Never fabricate a pass/fail.

> Read raw source **only after** the graph (or grep) has pinpointed the exact location. The graph does the cheap wide filtering; source-reading is the final confirmation on a suspect — not the first move.

## Verdicts

Every in-scope note lands exactly one:

- ✅ **current** — claims match the code. No action.
- ⚠️ **drifted** — note says Y, code now does Z. Actionable.
- ❌ **obsolete** — describes code/behavior that no longer exists. Actionable.
- 🔗 **not code-checkable** — external system, routed to the user. No auto-action.

## Report + action

1. **Emit an ephemeral report** in the conversation (nothing is persisted between runs — this sweep is stateless):
   - a summary table — `note · folder · verdict`
   - then a short detail block for every ⚠️/❌ with the **exact evidence**: the note's claim vs the contradicting `file:line`.
2. **For each ⚠️/❌, propose the specific fix** — a precise edit for drift, a *rewrite-or-remove* choice for obsolete — and **write only on the user's confirm** (same contract as `obsidian-audit`). Removals are **always only proposed**, never executed unprompted.
3. **On a confirmed correction**, also bump that note's `format_version` to the current value in `<vault>/VERSIONS.md` — the note is being touched anyway (opportunistic migration). If the edit changes what the note answers, update its hub hook to match.
4. ✅ and 🔗 need no write.

## Anti-hallucination rules (hard)

A wrong "fix" corrupts the vault worse than a missed drift. These are non-negotiable:

- **Never report ⚠️ drift without a concrete contradicting `file:line`.** A hunch is not a verdict.
- **When the check is genuinely uncertain** — the anchor is ambiguous, the graph is blind, and the source is unclear — return **🔗 not code-checkable**, never a guessed ✅ or ⚠️.
- **Never propose replacement text you can't tie to a specific line of current source.** No plausible-sounding rewrites.
- **Never delete or overwrite a note on your own** — propose, and let the user decide.

## Cadence

This is a deliberate maintenance sweep, run **manually, per project, weekly or monthly**. It is not auto-invoked and does not schedule itself. If you want a nudge, a calendar/`launchd` reminder that just says "run `/obsidian-maintenance` in `<repo>`" is enough — keep the intelligent, confirm-driven part interactive.

## Quick Reference

| Claim in a note | Tool that settles it | Verdict trigger |
|---|---|---|
| "`X` exists / is called" | `graphify explain "X"` | empty → ❌ obsolete |
| "A calls / feeds B", flow | `graphify path`/`explain`/`affected` | missing edge → ⚠️ drifted |
| "threshold 70 / 8000 chars / returns 400" | `graphify query` to locate → **read `file:line`** | value differs → ⚠️ drifted |
| env var / collection / endpoint literal | `grep`/read source for the literal | absent/renamed → ⚠️/❌ |
| Lark code, bank UI, 3rd-party quirk | none — external | → 🔗 not code-checkable |

## What NOT to do

- Don't check `decisions/`, `plans/`, or `investigations/` — historical, out of scope.
- Don't run across all projects — per-project only; the repo you're in is the source of truth.
- Don't verify against a stale graph — refresh with `graphify update .` first.
- Don't rewrite or delete notes unprompted — propose + confirm every time.
- Don't read `_Index_of_*` files or `graphify-auto/` nodes as note content.
- Don't fabricate a verdict for anything you couldn't actually check — say 🔗 instead.
