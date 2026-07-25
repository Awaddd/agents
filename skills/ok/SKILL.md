---
name: ok
description: "Finalize an active specs change only after the acceptance gate is clean; reconcile requirements, merge them into durable specs, archive the change, and never commit or push. Invoke explicitly with `$ok` when the user asks to land or finalize a change."
---

# $ok — finalize a change

Lands the active change into the durable spec. The spec records **VERIFIED REALITY** — reconciled here
against the passing tests so the living spec can't drift. The merge is **file operations** (`cp` + `rm`),
not section parsing.

**Hard rule: this command NEVER commits and NEVER pushes.** It ends by printing the changed paths and a
suggested commit message, then stops. The user commits via their own git flow.

## Resolve the active change

`$ARGUMENTS` may name the change; otherwise use the single folder under `specs/active/`. If there are
multiple, ask which. Call it `<change>` with folder `specs/active/<change>/`, containing `why.md`,
`requirements/<cap>/<req>.md`, `removals.txt` (optional), `tasks.md`.

---

## Step 1 — run `$check` (HARD GATE)

Run **`$check`** over this change's ACs. `$check` does all parts of the gate: it **validates structure**
(Step 0), **runs** each AC's `verified by:` test (passing / FAILING / NO TEST + orphans), **and reviews
quality** (weak/vacuous tests, invented behavior, missing ACs, contradictions). Findings come back tiered
CRITICAL / HIGH / MEDIUM / LOW.

**Any CRITICAL → STOP, hard.** CRITICAL = a structural failure (malformed requirement file) OR an AC with
no passing test (FAILING / NO TEST / broken link). A CRITICAL **cannot be accepted away** — it must be
fixed. **Any unresolved HIGH / MEDIUM / LOW finding → also STOP** until the user resolves or explicitly
accepts it. On any stop, print `$check`'s output and do **nothing else** — no reconcile, no merge, no move,
no commit suggestion.

> 🚨 Cannot finalize: `$check` is not clean (N CRITICAL — structural and/or no-passing-test — and/or
> unresolved HIGH/MEDIUM/LOW findings). Fix every CRITICAL; resolve or accept each lower-tier finding; then
> run `$ok` again.

Only when `$check` is clean — **zero CRITICAL**, and every HIGH/MEDIUM/LOW finding resolved or explicitly
accepted by the user — continue to Step 2.

---

## Step 2 — reconcile requirements to what actually shipped

The implementation may have drifted from what the edit files assumed. Bring the edit files in line with
**verified reality** before they become the source of truth:

For each `requirements/<cap>/<req>.md`:
- Confirm the `SHALL` statement and every AC still describe what the passing tests actually prove. If the
  shipped behavior differs (extra case, changed response, renamed test), **update the edit file** to match
  reality — the spec must describe the system as built, not as planned.
- Confirm every AC has a real `verified by:` line pointing at the actual test file + the exact test title
  that just passed. Fill in or correct any placeholder/stale links.
- If reconciliation requires a **behavior decision** (the test proves something different from the AC's
  intent and it's unclear which is right) — STOP and ask the user. Do not silently rewrite intent.

Edits in this step are to the `requirements/` files only (still in `active/`), never to `specs/` directly.

After editing, re-run `$check` to confirm it stays clean (links you just changed must still resolve to
passing tests).

---

## Step 3 — apply the merge (FILE OPERATIONS only)

Deterministic. No markdown surgery.

1. **Copy** each `specs/active/<change>/requirements/<cap>/<req>.md` → `specs/<cap>/<req>.md`, creating
   `specs/<cap>/` directories as needed. ADD = new file appears; MODIFY = whole file overwritten.
   Then **stamp the provenance backlink on the LIVING copy** (the one just written into `specs/`): for
   each `### Requirement: <name>` header, write/overwrite a `last-changed-by: <change-name>` line
   **immediately under the header**, where `<change-name>` is the `active/<change>` folder name being
   finalized this run. Plain line, same convention as `verified by:`. Applies to **both ADDED and MODIFIED**
   requirements — every requirement merged this run gets the current change stamped, overwriting any prior
   `last-changed-by:` line. **Do NOT stamp the `history/` copy** — its folder name already identifies the
   change; only the living `specs/` copy carries the line. The stamp is an extra metadata line under the
   header — it does **not** break `$check`'s structural validation (the requirement still has its `SHALL`,
   ≥1 AC, and `verified by:` lines).
2. **Removals** — if `removals.txt` exists, for each requirement path listed, delete the corresponding
   `specs/<cap>/<req>.md` (REMOVE/RENAME). Skip blank lines and `#` comments. If a listed file doesn't
   exist, note it and continue (don't fail the whole merge).

Track every path created / overwritten / deleted under `specs/` for the report.

---

## Step 4 — archive the change

Move the whole change folder out of `active/` into `history/`:

```
specs/active/<change>/  →  specs/history/<YYYY-MM-DD>-<change>/
```

Use today's date (`YYYY-MM-DD`). Create `specs/history/` if absent. This preserves the full why-trail
(why.md, tasks.md, the edit files) as the permanent record of the change.

---

## Step 5 — report & STOP (no git)

Do **not** run any git command. Print:

1. **Changed paths**, grouped:
   ```
   specs/   (source of truth)
     + specs/auth/2fa-required.md            (added)
     ~ specs/auth/login-issues-jwt.md        (modified)
     - specs/auth/legacy-session.md          (removed)
   history/
     + history/2026-06-26-add-2fa/           (archived from active/)
   ```
2. A **suggested commit message** — concise, no co-author line, no description body:
   ```
   suggested commit:
     specs: land add-2fa
   ```
3. The reminder:
   > Review the changed paths, then commit yourself. I have not run git. Per your rules: branch first if on
   > main, `git add <specific files>` (never `-A`/`.`), and commit only when you're ready.

Stop here.

---

## Rules

- **Never** `git add`, `git commit`, `git push`, or any git write. Reporting only.
- Order is fixed: gate (`$check`: validate + run + review) → reconcile → merge → archive → report. A
  non-clean `$check` (any CRITICAL — structural or no-passing-test — or any unresolved HIGH/MEDIUM/LOW
  finding) aborts everything after it.
- The file-ops merge is dumb on purpose — all judgment lives in `$check` (the run + review gate) and the
  reconcile (Step 2); the deterministic copy is what writes the source of truth.
- Surface what changed factually; the user decides whether to commit, amend the message, or hold.
