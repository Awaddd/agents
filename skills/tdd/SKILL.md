---
name: tdd
description: "The standalone red-green executor for the specs workflow. Runs a phase-by-phase TDD loop over an AC-anchored tasks.md: the orchestrator writes failing tests, implementation agents make them pass, each phase is gated, and wrong tests trigger an amendment loop. UI is a separate non-red-green lane. Invoke explicitly with `$tdd` or through `$imp` when a tasks.md exists."
---

# tdd — the shared red-green executor

Execute an AC-anchored plan one **phase** at a time. The orchestrator owns the
tests; green agents own the implementation; the two are never the same actor.

**Inputs:** a change folder `active/<change>/` containing `tasks.md` (AC-anchored
tables: `AC id │ test file │ test title │ new|edit`) and the requirement files
under `requirements/<cap>/<req>.md` (each AC's `WHEN … SHALL …` and its
`verified by:` line). The requirement files are the spec; `tasks.md` is the
throwaway map.

**Hard invariants (apply to every phase):**
- The **orchestrator** writes and runs tests. Test-authoring is **never delegated**.
- **Green agents never modify test files.** A test that looks wrong is a STOP-and-report blocker, not an edit.
- An import/compile failure is **not a valid red** — fix the stub, not the test.
- Tests in the work repo get **natural names only** — no AC IDs, no "phase", no
  `verified by`, none of this methodology leaks into the work codebase. The
  AC↔test link lives only in the spec's `verified by:` line.

Run the phases **in order**. Within a phase, run the steps below in order.

---

## Step 0 — Phase-0 gates & blockers (before any phase)

Read `tasks.md` and the spec-edit requirement files. Grep the plan for
`<!--.*-->` notes and the "Phase 0 — Gates" section. For each Phase-0 gate:

- **Hard (`STOP`-marked) and unresolved** → **STOP** and present it. Do not execute past it.
- **Soft** (no `STOP` marker — has a `Default if undecided:` line, or is documented resolved)
  → **PROCEED** on the default/resolution; do **not** block. Record the assumption
  (gate id + the default taken + its basis) so it surfaces in the run.
- A genuine open question in a `<!-- … -->` blocker note still **STOPS**.

Only `STOP`-marked gates and live blocker notes halt; otherwise proceed to Phase 1.

---

## RED SWEEP — orchestrator, never delegated

For **this phase only** (never sweep ahead into later phases — their interfaces
haven't survived implementation yet):

1. **Write every `test:`-tagged test for this phase**, from the AC's
   `WHEN … SHALL …` / scenario. One test per AC row in the phase's table, with
   the natural title from the `test title` column.
   - **Example tests are the default.** Where an AC is a natural *invariant*
     ("never issues a JWT before TOTP passes", "at most one direction green at a
     time"), you MAY write it as a **property test** (fast-check / the project's
     PBT lib) instead — it asserts the invariant across generated inputs rather
     than one case. Still a **natural test title**, still linked by `verified
     by:` — no IDs, no methodology in the work repo. Don't force PBT where an
     example test is clearer.
2. **Mock shapes from the real contract.** Derive every mock from the actual
   backend response/type — read the real shape, don't invent it. Do this once,
   centrally, at the top of the sweep.
3. **Write minimal stubs** — correct signatures, return/throw garbage — so each
   test **fails on an ASSERTION**, not on a missing import or type error. If a
   test red-fails on compile, fix the stub; never weaken the test.
4. **Run the tests.** Record each failure's reason (the assertion that fired)
   against the red-sweep line in `tasks.md`.
5. **Flag vacuous passes.** Any test that passes against a garbage stub is
   testing nothing — flag it loudly and fix the test before dispatch. A green
   red sweep is a defect.

Done-condition of the sweep: every phase test exists, ran, and **failed on an
assertion**, with the reason recorded.

---

## GREEN DISPATCH — spawn green-developer agent(s)

Spawn one `green-developer` agent per task or parallel group. Each agent prompt
includes **exactly**:
- The tests it owns (file + titles) — already written and red.
- The implementation files it may touch (from the phase's `Impl files` / `new|edit`).
- The rule, verbatim: **"Never modify test files. If a test looks wrong, STOP
  and report back — do not work around it."**

Agents implement until their tests pass. They run their own tests but do **not**
tick acceptance — that's the gate's job. Parallelise independent groups; run
dependent ones in order.

---

## PHASE GATE — orchestrator

After the phase's green agents report done:

1. Rerun the **full suite** for the change + typecheck/tidy (project's lint/format/types).
2. Confirm every phase AC's test now passes.
3. **Record the link back into the spec:** for each AC satisfied this phase,
   write/confirm its `verified by: <test file> › "<test title>"` line in
   `active/<change>/requirements/<cap>/<req>.md`. (Natural title only — the link
   lives here, never in the test.)
4. Tick the phase's acceptance in `tasks.md`.
5. Proceed to the next phase's red sweep.

Do not advance to the next phase until the gate is green.

---

## AMENDMENT LOOP — wrong test

When a green agent escalates "this test looks wrong":

1. **Orchestrator** (not the agent) amends the test.
2. Re-confirm red — the amended test must fail on an assertion again.
3. Re-dispatch the affected green agent(s).
4. Add a **one-line note** in `tasks.md` per amendment, so test churn stays visible.

Never let a green agent silence or work around a test. All test changes flow
through the orchestrator.

---

## UI / visual work — a SEPARATE lane, NOT red-green

Red-first is for logic with a knowable right answer. UI is creative — a test
written before the component constrains the design, and markup/style assertions
are brittle churn that test the wrong thing. So for component / page / styling tasks:

- **Extract the logic, red-green the logic.** Pull pure functions — parsers,
  data transforms, tree/diff/format builders, calculations,
  **sanitizers/escapers** — out of the component into a plain module (wherever
  the project keeps non-component code: `lib/`, `utils/`, a hook, a sibling
  `.ts`) and run them through the full RED SWEEP / GREEN DISPATCH above (`test:`).
  That's where the bugs hide.
- **The component gets ONE cheap `smoke+visual (ui)` test, written by the
  `frontend-developer` agent** — not the orchestrator, not red-first. It asserts
  only: renders without throwing, and one key `role`/text/`data-*` is present.
  **No markup snapshots, no class/style assertions.**
- **Real UI correctness = render it and look** — the screenshot/visual Verify
  step on the task, not unit assertions.
- **Always-test exceptions** (these have a right answer, so they DO get
  red-green): security boundaries (sanitization, escaping, auth-gated rendering)
  and any non-trivial logic that happens to live in a component — test the
  boundary/logic, still never the markup.

A typical UI feature splits into a few `test:` logic-module tasks (red-green) plus
several `smoke+visual (ui)` component tasks (frontend-developer lane).

---

## no-test (glue) tasks

Tasks tagged `no-test (glue)` (migrations, wiring, css, docs) **skip the sweep**.
Their done-condition is the task's **Verify** step — run it and confirm. No red,
no green agent required (the orchestrator or a green agent may just do them).

---

## Final verification (whole change)

After the last phase gate: run the full suite + typecheck/tidy once more, run any
whole-change Verify steps, and confirm **every AC in the requirements has a passing
`verified by:` link**. Report any AC without a passing test as a blocker — it
cannot be considered done.
