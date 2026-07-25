---
name: imp
description: "Implement an active specs change from its tasks.md using the red-green execution workflow, respecting phase gates and recording assumptions. Invoke explicitly with `$imp` when the user asks to implement an active mapped change. Requires a compatible `tdd` skill or equivalent executor instructions."
---

Implement the active specs change by running the red-green executor over its `tasks.md`.

This is a **thin wrapper** — it locates the change and hands off to the
`tdd` skill. It does **not** restate the protocol.

## 1. Find the change

- If the user named a change, use `active/<name>/`.
- Else if exactly one folder exists under `active/`, use it.
- Else list the `active/` folders and ask which one.

Confirm it has `tasks.md` and `requirements/<cap>/<req>.md` files. If `tasks.md`
is missing, stop — there's nothing to execute (run `$map` first).

## 2. Pre-flight gate

Read `tasks.md` and grep for `<!--.*-->` notes plus the "Phase 0 — Gates"
section. For each Phase-0 gate, decide soft vs hard:

- **Hard (`STOP`-marked) and unresolved** → **HARD STOP.** Present it; do not
  implement past it.
- **Soft** (no `STOP` marker — carries a `Default if undecided:` line, or is
  already documented resolved) → **PROCEED** on that default/resolution; do
  **not** block. Record the assumption (gate id + the default taken + its basis)
  so it surfaces in the Report.
- A genuine open question in a `<!-- … -->` blocker note still **STOPS**.

Net effect: unattended runs proceed through settled/defaulted gates; only
`STOP`-marked gates (and live blocker notes) halt.

## 3. Run the protocol

Invoke the **`tdd`** skill against `active/<change>/`. Follow it exactly,
phase by phase: RED SWEEP (orchestrator writes the tests) → GREEN DISPATCH
(green-developer agents) → PHASE GATE → AMENDMENT LOOP as needed; UI in its
separate lane; `no-test (glue)` tasks done by their Verify step. Do not duplicate
or paraphrase those steps here — the skill is the source of truth.

Before green agents start, have each read the project's relevant specs (e.g.
`platform/specs/` code-style / testing guides) and any agent instructions in
CLAUDE.md.

## 4. Report

After the final phase gate and whole-change verification, summarise:
- Phases completed; tests written and now passing (file › title).
- `verified by:` links recorded into `requirements/`.
- Typecheck/tidy + Verify results.
- Any amendments made, and any AC still without a passing test (blocker).
- **Every Phase-0 gate and how it was handled** — `proceeded-on-default «X»`
  (gate id + default taken + basis) or `stopped-for-«Y»`.

## What NOT to do

- Don't restate the red-green protocol — invoke `tdd`.
- Don't implement past an unresolved `STOP` gate or a live blocker note (soft gates: proceed on default + log).
- Don't author tests inside a green agent, and don't let a green agent edit a test.
- Don't write AC IDs or any specs methodology into the work codebase — natural test names only.
