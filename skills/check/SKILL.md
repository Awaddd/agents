---
name: check
description: "Read-only acceptance-criteria gate for specs changes: validate requirement structure, run linked tests, detect missing coverage and orphans, and review fidelity. Invoke explicitly with `$check` when the user asks to check or verify an active change."
---

# $check — the AC check (read-only)

The real check: it both **RUNS the tests** and **REVIEWS quality**. Cross-checks every acceptance
criterion against the test that's supposed to prove it, then reads that test and the implementation to
judge whether the test really exercises the AC. **READ-ONLY: never merge, never move, never edit specs.**
This is the same check `$ok` runs as a hard gate — run it freely while working.

**Done bar: an AC is done = covered + green + faithful.** Covered = it has a `verified by:` test; green =
that test passes; faithful = the review confirms the test really exercises the AC (not weak/vacuous) and
the impl is justified by an AC. All three, or it's not done. (A `glue`-lane AC is the exception: it has no
runnable test — it is "done" when its Verify step is satisfied, confirmed by inspection, not a test run.)

**Honest limit.** Structural validation (Step 0) and the test run (Steps 1–3) are deterministic — format
either matches or it doesn't, the runner says pass/fail. The review part (Step 4) is judgment from
*reading* the test + impl; it is not deterministic and can miss things. The fast green loop while coding
is just the test runner, not `$check` — `$check` is the fuller gate.

**Severity.** Every finding gets one of four tiers. Only CRITICAL means "cannot ship"; the rest are for
triage (the user decides whether to fix or accept):

- **CRITICAL** — a structural failure (Step 0) OR an AC with no passing test (FAILING / NO TEST / broken
  `verified by:` link). These block `$ok`. (A `glue`-marked AC has **no runnable test by design** and is
  **not** CRITICAL — it is verified by inspection / its Verify step; see Step 0 rule 5 and Step 2.)
- **HIGH** — a weak/vacuous test, or a contradiction with another AC / the `SHALL` / `why.md`. The AC
  technically passes but the proof is unsound.
- **MEDIUM** — invented behavior (impl no AC justifies) or a missing AC (behavior implied but uncaptured).
- **LOW** — orphan tests, orphan tasks, and non-breaking format warnings.

## Scope

`$ARGUMENTS` selects what to check:
- **(default, no args)** — the active change. Resolve via `specs/active/` (the single change folder there;
  if more than one, ask which). Check every AC in `active/<change>/requirements/<cap>/<req>.md`.
- **`all`** — every requirement file under `specs/<cap>/<req>.md` (the whole durable spec).
- **a path** — check that one requirement file or capability folder.

## Step 0 — structural validation (deterministic)

Before running anything, validate the **shape** of every in-scope requirement file. This is the
deterministic half of the gate — no judgment, just format. A malformed spec file can't be trusted to
map ACs to tests, so structural defects are caught first and surfaced as **CRITICAL (structural)** ahead
of the run table.

For each in-scope requirement file, confirm:

1. **Requirement heading + normative statement** — a `### Requirement: <name>` heading is present, and
   its body contains a `SHALL` or `MUST` statement.
2. **At least one AC** — ≥1 line of the form `#### AC <cap>.<feature>.<n> — <name>: <EARS>`.
3. **AC ID shape + uniqueness** — each AC id matches `^[a-z0-9-]+\.[a-z0-9-]+\.\d+$` (e.g. `auth.2fa.1`;
   reject `auth.2fa`, `Auth.2FA.1`, `auth.2fa.1a`). Every AC id is **unique** across all in-scope ACs —
   a duplicate id is CRITICAL.
4. **EARS keyword** — the AC text carries a canonical EARS keyword: `WHEN`, `WHILE`, `IF … THEN`,
   `WHERE`, or a bare ubiquitous `THE SYSTEM SHALL`. An AC with no recognizable EARS form is a defect.
5. **`verified by:` line present + well-formed** — every AC is followed by a `verified by:` line whose
   value is **one of**: the literal `TBD` (valid before `$imp` fills it); the shape `<path> › "<title>"`;
   or a **`glue` marker** — the value begins with `glue`, optionally followed by ` — <why it has no test>`
   (e.g. `verified by: glue — route registered with X permissions; regression covered by the suite`). The
   `glue` form is for glue-lane ACs (wiring / config / permission-registration / migration / css / docs)
   that are verified by inspection or their Verify step, **not** a runnable test. A value that is none of
   these (e.g. `verified by: login test`, missing the `›` / quotes) is a structural failure. Missing the
   line entirely is also a structural failure.

**Scope split — read this carefully:** Step 0 owns the *line shape only*. `verified by: TBD` is a
**valid shape** and passes Step 0. Whether the named test actually exists and passes is Steps 1–3' job —
so an AC sitting at `TBD` mid-work still shows up as **NO TEST** there (unchanged). Don't double-report:
a `TBD` AC is structurally fine (Step 0) and NO TEST (Step 2). A `glue` AC is likewise structurally fine
(Step 0) and gets status **glue** (Step 2) — no runnable test, not CRITICAL.

Any Step 0 failure = **CRITICAL (structural)**. Record the file, the AC id (if parseable), and which
rule failed. These appear in their own block in the report, before the AC status table.

## Step 1 — collect ACs

For each in-scope requirement file, parse every AC line:

```
#### AC <cap>.<feature>.<n> — <name>: WHEN … THE SYSTEM SHALL …
verified by: <test file> › "<test title>"
```

Record per AC: the **AC id**, its **`verified by:`** target (test file + exact test title), and whether
a `verified by:` line is present at all.

- AC with **no `verified by:` line** (or value `TBD`) → status **NO TEST** (nothing to run). It is CRITICAL.
- AC whose `verified by:` value begins with **`glue`** → status **glue** (no runnable test by design;
  verified by inspection / its Verify step). Not run in Step 2, **not** CRITICAL. Note it in the report so
  a human eyeballs the Verify step.

## Step 2 — run the verifying tests

**Skip `glue` ACs** — they carry no runnable test (status **glue**, set in Step 1); they are not run here
and are never CRITICAL. List them in the report so their Verify step gets a human eyeball.

Group the remaining ACs by test file so each file runs once. For each `verified by:` target:

1. Confirm the **test file exists**. Missing file → **NO TEST** (CRITICAL).
2. Run the suite filtered to that file (use the project's runner — vitest/jest/pytest/etc.; detect from
   the repo, don't assume). Filter by test title where the runner supports it.
3. Match the **exact test title** from `verified by:` against the run results:
   - title found AND passed → **passing**
   - title found AND failed → **FAILING** (CRITICAL)
   - title not found in results (renamed/deleted) → **NO TEST** (CRITICAL) — the link is broken; report the
     stale target so the user can re-point the `verified by:` line.

Do not edit anything to fix a broken link — only report it.

## Step 3 — orphans (reverse check)

Flag the other direction so nothing hides:
- **Orphan tests** — test titles in the in-scope test files that no AC's `verified by:` points at.
- **Orphan tasks** — rows in `active/<change>/tasks.md` whose AC id has no matching AC in the requirements.

Orphans are **LOW** — not blocking by themselves; list them for the user to judge.

## Step 4 — quality review (READ the test + the impl)

Steps 1–3 prove a test ran and passed; a green test can still be worthless. For each AC, **read its
verifying test AND the implementation it covers**, and flag (tier in brackets):

- **Weak / vacuous tests [HIGH]** — trivially true (asserts a constant, can't fail), **mocks away the
  unit under test** (so the real code never runs), or happy-path only when the AC implies
  error/empty/boundary cases.
- **Contradictions [HIGH]** — a test or impl that conflicts with another AC, the `SHALL`, or `why.md`.
- **Invented behavior [MEDIUM]** — implementation that does something **no AC justifies**. Either it
  needs an AC or it shouldn't be there.
- **Missing ACs [MEDIUM]** — behavior the ticket / `why.md` implies but **no AC captures** (the gap that
  lets behavior get invented in the first place).

For a `glue` AC, there is no test to read — instead confirm by inspection that its stated Verify basis
holds (e.g. the route really is registered with the claimed permissions, the config really is untouched).
Flag a glue claim that inspection contradicts as a **HIGH** (the glue assertion is unsound).

This is judgment, not a test result — present findings factually with their tier; whether to fix or
accept a HIGH/MEDIUM is the user's call. An AC is **faithful** only when its test survives this review.

## Step 5 — report

First, if Step 0 found any **structural** failures, print them in their own loud block before anything
else (these are CRITICAL — a malformed spec can't be trusted to map ACs to tests):

```
🚨🚨🚨 CRITICAL (structural) — these requirement files are malformed 🚨🚨🚨
  specs/auth/2fa-required.md   AC auth.2fa.2 — duplicate id (also on auth.2fa.1's sibling)
  specs/auth/2fa-required.md   AC auth.2fa.3 — verified by: malformed ("login test", expected `path › "title"`)
  specs/auth/login.md          requirement body has no SHALL/MUST statement
Fix the format before the run results below can be trusted.
🚨🚨🚨
```

Then the status table, every in-scope AC, one row each:

```
AC id            │ status
─────────────────┼─────────
auth.2fa.1       │ passing
auth.2fa.2       │ FAILING
auth.2fa.3       │ NO TEST
auth.login.1     │ passing
auth.login.2     │ glue
```

(`glue` in the table = no runnable test, verified by inspection / its Verify step — not a failure, not
CRITICAL; it does not appear in the NO-PASSING-TEST block below.)

Then, if any AC is FAILING or NO TEST, print the CRITICAL block — make it loud and unmissable:

```
🚨🚨🚨 CRITICAL — these ACs have NO PASSING TEST 🚨🚨🚨
  auth.2fa.2   FAILING   src/auth/__tests__/2fa.test.ts › "rejects an invalid TOTP with 401"
  auth.2fa.3   NO TEST   verifying test not found (title renamed or file missing)
These cannot ship. Either the test is broken, missing, or the verified-by link is stale.
🚨🚨🚨
```

If every AC is passing (or `glue`) and Step 0 was clean, say so plainly: `✅ All N ACs structurally valid;
every testable AC has a passing test (M glue ACs verified by inspection).`

Then the review findings from Step 4, **grouped by tier** (only the tiers that have findings) — these are
judgment calls, for the user to resolve or accept:

```
🔴 HIGH (passes, but the proof is unsound):
  weak test:     auth.2fa.2 — mocks the verifier, so the real TOTP check never runs
  contradiction: AC auth.login.1 SHALL issue a JWT, but the impl sets a session cookie

🟡 MEDIUM (behavior/AC mismatch):
  invented:      impl rate-limits login, but no AC covers rate-limiting
  missing AC:    why.md implies lockout after N attempts; no AC captures it
```

Then the LOW block — orphans and non-breaking warnings (only if present):

```
🔵 LOW (orphans / warnings — your call):
  orphan test:  src/auth/__tests__/2fa.test.ts › "locks account after 5 attempts"
  orphan task:  tasks.md row auth.2fa.4 — no matching AC in requirements
```

## Rules

- Read-only. No file writes, no merge, no archive, no git.
- One combined report: structural validation (Step 0), the run results (Steps 1–3), **and** the quality
  review (Step 4) together.
- Done = covered + green + faithful. Assign each finding a tier (CRITICAL / HIGH / MEDIUM / LOW) per the
  rubric in the header — but only **CRITICAL** is "cannot ship". HIGH/MEDIUM/LOW are triage: surface them
  and let the user decide whether to fix or accept.
- A `glue` AC has no runnable test by design (status **glue**) — it is never CRITICAL for lacking one; its
  soundness is judged by inspection in Step 4.
- Structural validation and the run are deterministic; the review is judgment from reading. Don't present
  a review finding as proven — present it (with its tier) for the user to fix or accept.
- A renamed test is the expected failure mode of the natural-name link — flag it loudly with the stale
  target, don't try to guess the new title.
