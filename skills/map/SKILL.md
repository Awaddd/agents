---
name: map
description: "Create an AC-anchored implementation plan for a change under specs/active, including phases, gates, tests, implementation files, and verification. Invoke explicitly with `$map` when the user asks to plan or map an active spec."
---

Create an AC-anchored implementation plan for the active change.

Read `active/<change>/` (the change folder produced by `$spec`) and produce
`active/<change>/tasks.md` — a rich, ordered plan that `$imp` (the `tdd`
executor) can run without further context.

This command PLANS. It does not write code. Execution is `$imp`.

## Inputs — read these first

Inside `specs/active/<change>/`:
- **why.md** — intent, scope, and the prereqs/decisions/dependencies surfaced during `$spec`.
  These become **Phase-0 Gates**.
- **requirements/<cap>/<req>.md** — the new/modified requirement files. Each holds a `SHALL` and one or
  more ACs in the format:
  ```
  #### AC <cap>.<feature>.<n> — <name>: WHEN <trigger> THE SYSTEM SHALL <response>
  verified by: <test file path> › "<exact test title>"
  ```
- **removals.txt** (if present) — requirement files being deleted/renamed.
- **context.md** (project root of `specs/`) + **context.local.md** — stack, conventions, glossary.

If `<change>` isn't obvious from context, pick the single folder under `active/`; if there are several, ask.

## Process

### 1. Gather the ACs
List every AC across all `requirements/` requirement files. Each AC's `verified by:` line gives you the
**test file** and the **exact test title** (natural name — already methodology-free). These ACs are the
spine of the plan: every one must land in a phase table, and nothing else carries a test.

### 2. Classify each AC into a lane
Part of writing the plan — **automatic, and it does NOT pause or ask the user.** Tag every AC exactly one
lane in its table row (`logic` | `ui` | `glue`). Same carve-out as the `tdd` skill's UI-lane section
(keep the two in sync):
- **React component rendering/behaviour → `ui`.** The frontend-developer agent writes its own
  `smoke+visual (ui)` test; no orchestrator red-first, no markup/style assertions.
- **Pure logic, even when it lives in a UI feature** (URL builders, payload shapers, parsers, formatters,
  data transforms) → **EXTRACT it into a plain module and make it a SEPARATE `logic` task** (red-green).
- **Security boundary / non-trivial logic inside a component** (escaping, sanitizers, auth-gated render) →
  `logic` — test the logic, not the markup.
- **Wiring/config/migration/css/docs → `glue`** (no test; done by its Verify step).

### 4. Investigate the work repo
Explore the real code (agents in parallel where it helps):
- Which impl files must change, which must be created.
- Current behaviour, dependencies, edge cases, the real contract shapes for any mocked boundary.
- Whether each `verified by:` test file already exists (→ `edit`) or is new (→ `new`).
- **Frontend detection:** if a phase touches a Vite/React or `*-ui` service (e.g. staff-ui, `.tsx`,
  React components), default its ACs to `ui` AND actively hunt for extractable pure logic — URL builders,
  payload shapers, parsers, formatters, transforms — to split out into separate `logic` tasks (per step 2).
Never plan changes to files you haven't read.

### 5. Decide files (create vs edit)
map owns this. For every AC's test file and every impl file, decide `new` or `edit`. Record impl files
in a per-phase **Impl files** list. This is planning, not implementation.

### 6. Write the plan
Save to `specs/active/<change>/tasks.md` using the structure below. Group ACs into phases by dependency
order. Each phase is AC-anchored: an AC table, an Impl files list, tasks, a red-sweep line, Acceptance.

## Plan structure

```markdown
# <change> — implementation plan

## Context
One paragraph from why.md: the problem and the scope of this change.

## Phase 0 — Gates
Decisions, approvals, and dependencies that block work. Each pulled from why.md.

Every gate is one of two kinds — mark it accordingly so `$imp` knows whether to halt:

- **Hard gate (`STOP`)** — genuinely needs the human: no safe default, because it's irreversible,
  destructive, business-logic, or proceeding-wrong is costly. Write it with a `STOP` marker:
  `- [ ] **G2 STOP — <gate>**`. `$imp` HARD-STOPS on an unresolved `STOP` gate and does not implement past it.
- **Soft gate (no marker)** — has a safe assumption to run on. It MUST carry a `Default if undecided:` line.
  `$imp` PROCEEDS on that default unattended and records the assumption; it does not block.

Don't mark a gate `STOP` just because it's undecided — `STOP` is for "a human must choose." If a safe
default exists, it's soft.

- [ ] **G1 <soft gate>** — <decision/approval/dependency needed>.
      **Blocks:** Phase N (or task N.M). **Default if undecided:** <the assumption we proceed on>.
- [ ] **G2 STOP — <hard gate>** — <decision needing human sign-off; no safe default>.
      **Blocks:** Phase N.

(No gates? Write "None." and move on.)

## Phase N — <goal>
Prereq: <Phase N-1 | none>. Parallel: <N.2 ∥ N.3 | sequential>.

| AC id          | Lane  | Test file                          | Test title (natural)                    | new\|edit |
|----------------|-------|------------------------------------|-----------------------------------------|-----------|
| <cap>.<feat>.1 | logic | path/to/foo.test.ts                | "returns X when Y"                      | new       |
| <cap>.<feat>.2 | logic | path/to/foo.test.ts                | "rejects Z with 401"                    | edit      |

**Lane** is one of (mirrors the `tdd` skill's lanes — keep them in sync):
- `logic` — orchestrator red-green (`tdd`'s `test:`): pure logic with a knowable right answer.
- `ui` — frontend-developer agent writes its OWN `smoke+visual (ui)` test (renders without throwing + one
  key role/text/`data-*`); correctness = render-and-look. **NOT orchestrator red-first.** No markup/style asserts.
- `glue` — no-test (`tdd`'s `no-test (glue)`): wiring/config/migration/css/docs; done by its Verify step.

**Impl files:** path/to/foo.ts (new), path/to/bar.ts (edit)

- [ ] **N.1 <task>** — **File:** path — **What:** specific change, tied to the AC(s) above.
      **Verify:**
        - automated: `pnpm test path/to/foo.test.ts` (the AC tests above go green)
        - manual: how to eyeball it (UI: render + look; else "n/a")
        - security-checks: auth/escaping/secret-handling to confirm, or "n/a"
- [ ] **N.2 <task>** — File / What / Verify (three-tier) …
      <!-- _deviation: if impl diverges from this task, add an italic note here when ticking — what changed + why_ -->
- [ ] **Red sweep confirmed** — orchestrator line: every test in THIS phase's table was written, run,
      and **failed on an ASSERTION** (not a compile/import error). Record each failure reason here.
      Flag any test that passed without implementation as **VACUOUS PASS — fix the test before green**.

**Acceptance:** [ ] every AC in this phase's table has a passing test · [ ] suite + tidy green · [ ] gates this phase depended on are resolved

## Verification (whole change)
- automated: full suite + types/tidy.
- manual: end-to-end checks across the change.
- security-checks: the cross-cutting security boundaries this change touches.
- coverage: every AC across all requirements maps to a phase row above (this is what `$ok` will gate on).
```

Checkboxes are execution state — `$imp` (and anyone resuming after a context reset) ticks them as
work lands. Single-phase changes are fine: Phase 0 (or "None.") then one phase.

A filled-out example of the structure above:

```markdown
# add-2fa — implementation plan

## Context
<One paragraph from why.md: the problem and the scope of this change.>

## Phase 0 — Gates
Decisions, approvals, and dependencies that block work. Each pulled from why.md.
Resolve or accept the default before the phase it blocks starts.

- [ ] **G1 <decision>** — <what needs deciding/approving>.
      **Blocks:** Phase 1. **Default if undecided:** <the assumption we proceed on>.
- [ ] **G2 <dependency>** — <external thing this waits on, e.g. a library / API / migration>.
      **Blocks:** task 2.1. **Default if undecided:** <fallback approach>.

<!-- No gates? Replace this section's bullets with: None. -->

## Phase 1 — <goal>
Prereq: none. Parallel: sequential.

| AC id      | Lane  | Test file                       | Test title (natural)                    | new\|edit |
|------------|-------|---------------------------------|-----------------------------------------|-----------|
| auth.2fa.1 | logic | src/auth/__tests__/2fa.test.ts  | "returns a JWT when the TOTP is valid"  | new       |
| auth.2fa.2 | logic | src/auth/__tests__/2fa.test.ts  | "rejects an invalid TOTP with 401"      | new       |
| auth.2fa.3 | logic | src/auth/__tests__/2fa.test.ts  | "rejects an expired TOTP with 401"      | new       |

**Impl files:** src/auth/2fa.ts (new), src/auth/login.ts (edit)

- [ ] **1.1 Verify TOTP code** — **File:** src/auth/2fa.ts (new) — **What:** validate a submitted TOTP
      against the user's secret + window; satisfies auth.2fa.1–3.
      **Verify:**
        - automated: `pnpm test src/auth/__tests__/2fa.test.ts` (all 3 AC tests green)
        - manual: n/a
        - security-checks: constant-time compare; expired/replayed codes rejected; secret never logged
- [ ] **1.2 Wire 2FA into the login flow** — **File:** src/auth/login.ts (edit, ~L40) — **What:** after
      password verification, require TOTP when 2FA is enabled before issuing the JWT.
      **Verify:**
        - automated: existing login suite stays green
        - manual: log in with 2FA enabled end-to-end
        - security-checks: no JWT issued before TOTP passes
      <!-- _deviation: if impl diverges from this task, add an italic note here when ticking — what changed + why_ -->
- [ ] **Red sweep confirmed** — every test in this phase's table written, run, and **failed on an
      ASSERTION** (not compile/import). Record each failure reason here. Flag any **VACUOUS PASS**
      (went green with no implementation) and fix the test before green dispatch.

**Acceptance:** [ ] auth.2fa.1–3 have passing tests · [ ] suite + tidy green · [ ] G1, G2 resolved

## Phase 2 — 2FA enrolment screen (staff-ui)
Prereq: Phase 1. Parallel: 2.1 ∥ 2.2. (staff-ui = React/Vite → frontend lane; pure logic extracted out.)

| AC id      | Lane  | Test file                          | Test title (natural)               | new\|edit |
|------------|-------|------------------------------------|------------------------------------|-----------|
| auth.2fa.4 | logic | src/auth/otpauthUri.test.ts        | "builds an otpauth:// URI for the secret" | new   |
| auth.2fa.5 | ui    | (frontend agent writes its own)    | "Enrolment screen renders the QR"  | new       |

**Impl files:** src/auth/otpauthUri.ts (new), staff-ui/src/EnrolScreen.tsx (new)

- [ ] **2.1 Build the otpauth URI** — **File:** src/auth/otpauthUri.ts (new) — **What:** EXTRACTED pure
      logic from the enrolment screen — assemble the `otpauth://totp/...` URI the QR encodes; satisfies auth.2fa.4.
      **Verify:**
        - automated: `pnpm test src/auth/otpauthUri.test.ts` (AC test green)
        - manual: n/a
        - security-checks: secret never logged
- [ ] **2.2 Enrolment screen** `ui` — **File:** staff-ui/src/EnrolScreen.tsx (new) — **What:** render the
      QR (from the extracted URI builder) + verify-code field; satisfies auth.2fa.5. Frontend-developer agent
      writes its OWN `smoke+visual (ui)` test (renders + QR `role`/`data-*` present) — NOT orchestrator red-first.
      **Verify:**
        - automated: smoke test passes (frontend agent)
        - manual: render the screen and look — QR scans, field accepts input
        - security-checks: n/a
- [ ] **2.3 <glue task, no AC>** `glue` — **File:** path — **What:** migration / wiring / css / docs.
      **Verify:**
        - automated: n/a (no-test glue)
        - manual: <how to confirm this task alone>
        - security-checks: n/a
- [ ] **Red sweep confirmed** — `logic` rows only (auth.2fa.4): failure reasons recorded; vacuous passes
      flagged. `ui`/`glue` rows are not swept here.

**Acceptance:** [ ] auth.2fa.4 (logic) has a passing test · [ ] auth.2fa.5 (ui) smoke + render-and-look done · [ ] suite + tidy green

## Verification (whole change)
- automated: full suite + types/tidy.
- manual: end-to-end checks across the change.
- security-checks: cross-cutting security boundaries this change touches.
- coverage: every AC in all requirements/ requirement files appears in a phase table above WITH a lane tag
  (this is what `$ok`'s coverage gate enforces).
```

## The four required patterns

1. **Phase-0 Gates.** Decisions/approvals/dependencies from why.md, each tagged with **what it blocks**.
   Mark a gate `STOP` only when a human must choose (no safe default — irreversible/destructive/business-logic);
   every soft (un-`STOP`ed) gate carries a **default-if-undecided** so `$imp` runs unattended and work isn't
   stranded. Surface scope/severity to the user; don't pre-decide it.
2. **Inline deviation notes.** When a ticked task diverged from plan, add an _italic note_ under its checkbox
   (what changed + why). Keeps the why-trail visible and feeds `$ok`'s reconcile step.
3. **Three-tier Verify.** Every task's **Verify:** has `automated:`, `manual:`, `security-checks:` — use
   "n/a" where a tier doesn't apply rather than dropping it.
4. **Vacuous-pass flagging.** The red-sweep line must call out any test that went green before implementation
   existed — an assertion that can't fail proves nothing. Flag it; the test gets fixed before green dispatch.

## AC-anchored rules

- **Every AC lands in exactly one phase table.** The table row is the single source linking AC → lane → test
  file → test title → new/edit. Don't restate the EARS text in the task; point at the AC id.
- **Every AC carries a lane** (`logic` | `ui` | `glue`) — see step 2. The lane tells `$imp`'s `tdd` executor
  which path the AC runs: `logic` = orchestrator red-green, `ui` = frontend-developer's own smoke+visual test
  (no red-first), `glue` = no test (done by Verify). An untagged AC silently defaults to red-green, which is
  wrong for UI — so no AC ships untagged.
- **Natural test names only** in tables and the repo — never write AC ids or "SHALL"/methodology into the work
  codebase. The AC id lives only in `tasks.md` and the durable spec.
- **UI is not red-green** — this is the `ui` lane, and it must match the `tdd` skill's "UI / visual work" section
  (the two cannot drift). For component/page/styling work: extract pure logic (parsers, transforms, URL builders,
  payload shapers, formatters, sanitizers) into a plain module and give THAT a `logic` AC + test; the component
  gets a `ui` AC whose test is a single `smoke+visual (ui)` check the frontend-developer agent writes itself,
  plus a `manual:` render-and-look Verify. Security boundaries / non-trivial logic in components (escaping,
  auth-gated render) are `logic` — test the logic, not the markup. Don't blanket-test markup.

## Execution is delegated — do NOT inline red-green here

The full red-green protocol (orchestrator-owned red sweep, green agents never touch tests, phase gates,
amendment loop) lives in the **`tdd`** skill and is run via **`$imp`**. This plan only needs
to be AC-anchored and ordered well enough for that executor — do not copy the protocol into tasks.md.

## Style rules

- **Concrete.** Real files, real functions, real line numbers (approximate). No "update the service layer".
- **Minimal.** Plan only the delta this change introduces — not the whole system.
- **Ordered.** Dependencies in order; independent work marked Parallel so `$imp` fans out.
- **No ceremony.** No time estimates, risk registers, success metrics, or definition-of-done boilerplate.
- **Don't write code** — describe the change. One exception: a red-sweep test may be sketched, since the test
  IS the spec.

## What NOT to do

- Don't author or edit requirements/ requirement files — that's `$spec`. map consumes them.
- Don't apply the merge into `specs/` — that's `$ok`.
- Don't leave any AC out of a phase table (`$ok`'s coverage gate will fail on it).
- Don't over-plan a task small enough to just do — say so.
