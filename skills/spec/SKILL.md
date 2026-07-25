---
name: spec
description: "Start the specs workflow from a feature idea or Jira/Linear ticket, interrogate gaps, lock EARS acceptance criteria, and write the active change artifact. Invoke explicitly with `$spec` when the user asks to specify, scope, or acceptance-test a change."
---

Grill a freeform seed into a locked set of acceptance criteria, then write the change artifact.

`$spec <seed>` — the seed is EITHER a freeform idea (e.g. `$spec daily digest`) OR a ticket reference (a Jira key like `ABC-123`, or a Linear id/URL).

This is the front door of the **specs** workflow. It runs a short, story/feature-focused grill (bounded and AC-first), then emits the change folder under `active/` that `$map` will plan from. It does NOT plan tasks or write code.

## Seed type — freeform idea vs ticket (one system, two inputs)

Before scaffolding/grilling, classify the seed:

- **Ticket reference** — looks like a Jira key (`ABC-123`), a Linear id, or a Linear URL. The ticket is the **already-designed seed**: fetch it and treat its body as the design.
  - **Jira** → fetch via the atlassian MCP (`getJiraIssue`).
  - **Linear** → fetch via the linear MCP.
  - From the fetched ticket, extract the intent, the stated requirements, and any acceptance criteria already written. Then **grill the GAPS the ticket leaves unspecified** — be MORE probing here, not less. Undesigned gaps (unstated edge cases, error/empty/boundary behavior, anything the ACs imply but don't pin down) are exactly where behavior gets invented and the ticket's real ACs get broken. Don't re-litigate what the ticket already decided; spend every question on what it left open.
  - **Degrade gracefully** if the relevant MCP isn't available or the fetch fails: tell the user you couldn't fetch the ticket, and fall back to the freeform grill using whatever the user pasted.
- **Freeform idea** (no ticket, or no MCP available) — the normal freeform grill below: sharpen the seed itself into a `SHALL`, then lock its ACs.

Either way the grill stays **≤5 questions, via `Codex question interface`, each with a recommended answer**, and emits the same change artifact. The only difference is where the design starts: a ticket seed starts pre-designed (grill the gaps), a freeform seed starts blank (grill the whole thing).

## Preflight — scaffold on first run

1. Resolve the repo root (the directory containing `.git`; else the current working directory).
2. **If `specs/` doesn't exist there, scaffold it first** (see "Scaffold the specs/ folder" below), then continue. If `specs/` already exists, skip scaffolding and proceed.
3. Read `specs/context.md` (stack + conventions + glossary). Read `specs/context.local.md` if present (gitignored; never echo its values).
4. Scan `specs/` to learn existing **capabilities** (the folders — ignore reserved names: `active`, `history`, `context.md`, `context.local.md`, `.gitignore`) and skim existing requirement files in the capability the seed likely touches. Reuse existing capability/term names; don't invent parallel ones.
5. Derive a kebab `<change-name>` from the seed (e.g. `daily-digest`). If `active/<change-name>/` already exists, ask whether to resume it or pick a new name.

## Scaffold the specs/ folder (first run only)

Run once per repo, at the repo root. Detect the project stack and write a pre-filled `specs/` skeleton: the durable source of truth (`context.md`), the gitignored secrets stub (`context.local.md`), the `.gitignore`, and the `active/` + `history/` working folders. Never clobber an existing `specs/`.

### Detect the stack

Read whatever manifest(s) exist and infer language, framework, and test runner. Read the files; don't ask the user — but never GUESS a framework you can't see in a manifest.

**Find the manifests first (handle monorepos).** Look at the repo root, and if the root has no usable manifest (e.g. a pnpm workspace with code under `services/*`), scan subdirectories before concluding anything:
- `*/package.json`, `services/*/package.json`, `packages/*/package.json`, `apps/*/package.json`
- read `pnpm-workspace.yaml` (and any `package.json` `workspaces` globs) and follow its globs to the real manifests
- the equivalent for non-JS stacks (e.g. nested `pyproject.toml`, `go.mod`, `Cargo.toml`)

Infer language/framework/test-runner from the manifests you actually found — not from the root's absence of one. Cover at least:

- **`package.json`** → JS/TS. Language = TS if `typescript` is a dep or `tsconfig.json` exists, else JS. Framework from deps (`next`, `react`, `vue`, `svelte`, `express`, `fastify`, `nest`…). Test runner from deps/scripts (`vitest`, `jest`, `mocha`, `playwright`). Note the package manager from the lockfile (`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `package-lock.json` → npm, `bun.lockb` → bun). Capture the test script (e.g. `pnpm test`).
- **`pyproject.toml` / `setup.cfg` / `requirements.txt`** → Python. Framework (`django`, `flask`, `fastapi`). Test runner (`pytest`, `unittest`).
- **`go.mod`** → Go. Test runner = `go test`.
- **`Cargo.toml`** → Rust. Test runner = `cargo test`.
- **`Gemfile`** → Ruby (`rspec`, `minitest`). **`pom.xml` / `build.gradle`** → Java/Kotlin (`junit`). **`composer.json`** → PHP (`phpunit`).

If multiple stacks are present (monorepo), record each on its own line.

**Never guess a framework.** If the stack still can't be determined confidently after scanning subdirs — no manifests found anywhere, or the manifests are genuinely mixed/conflicting — write **`Stack: TBD — confirm`** for the unknown rows and ask the user to confirm. Do NOT default to a specific framework (no Hono/Express/Next/etc. unless a manifest actually names it).

### Write the skeleton

Create exactly this layout under the repo root. Use `.gitkeep` to keep the empty working folders trackable.

```
specs/
  context.md           # COMMITTED — stack + conventions + glossary + ABSTRACT infra refs
  context.local.md     # GITIGNORED — the actual secret/infra values
  .gitignore           # ignores context.local.md
  active/.gitkeep      # in-flight changes land here ($spec)
  history/.gitkeep     # landed changes archived here ($ok)
```

**`specs/context.md`** — use the context template below. Fill the **Stack** section from detection (language, framework, test runner, package manager, test command). Leave **Conventions** lightly seeded from what you observed (e.g. test file location pattern) or `TBD`. Leave **Glossary** with one or two placeholder rows. Leave **Infra references** empty except the template's note. Never write a secret or infra value here.

```markdown
<!--
Committed context template — specs/context.md
The durable, COMMITTED reference for the specs workflow: stack + conventions + glossary + ABSTRACT infra refs.
Written when $spec scaffolds (Stack auto-filled from the project's manifests); kept current by $spec (glossary).
SAFE TO COMMIT. Secrets and infra VALUES never go here — see "Infra references" below; values live in the
gitignored context.local.md. Replace every <placeholder>; delete rows that don't apply.
-->

# context.md

The shared context for this repo's **specs** workflow. Read before planning or building.

## Stack

| Aspect | Value |
|---|---|
| Language | <e.g. TypeScript> |
| Framework | <e.g. Next.js / FastAPI / none> |
| Test runner | <e.g. Vitest / pytest / go test> |
| Test command | <e.g. `pnpm test`> |
| Package manager / build | <e.g. pnpm / uv / cargo> |

<!-- Monorepo: one row block per package/service. -->

## Conventions

File layout, naming, and patterns the spec and the implementation follow.

- **Test location** — <e.g. `src/**/__tests__/*.test.ts` colocated; or `tests/`>
- **Source layout** — <where app code lives, e.g. `src/`>
- **Patterns** — <house patterns worth pinning, e.g. "thin route handlers, logic in services">
- <add as they emerge>

## Glossary

Domain terms with one canonical meaning. $spec updates this when a term gets sharpened.

| Term | Meaning |
|---|---|
| <Term> | <one-line canonical definition> |
| <Term> | <one-line canonical definition> |

## Infra references

Name the infra things the system depends on — **NAMES ONLY, never values.** Each name's real value
(host, IP, domain, connection string, key) lives in `context.local.md`, which is gitignored.

> NEVER write a secret, credential, host, IP, or domain into this file. It is committed and pushed.
> This file names the variable (e.g. `GATEWAY_HOST`); `context.local.md` holds what it equals.

| Variable name | What it is (abstract) |
|---|---|
| `GATEWAY_HOST` | <e.g. the API gateway the app calls — value in context.local.md> |
| `DB_URL` | <e.g. primary database connection — value in context.local.md> |

<!-- Leave the table empty if the project has no external infra refs yet. -->
```

**`specs/context.local.md`** — a short gitignored stub holding the real values that `context.md` only names:

```markdown
# context.local.md — LOCAL ONLY, gitignored. Never commit this file.

Holds the real values for the infra references named (abstractly) in `context.md`.
Each entry: the variable NAME from context.md = its value.

<!-- Example (delete and replace):
GATEWAY_HOST = <the actual host>
-->
```

**`specs/.gitignore`** — one line:

```
context.local.md
```

**Working folders** — create `specs/active/.gitkeep` and `specs/history/.gitkeep`.

Scaffold rules:
- Don't overwrite or merge into an existing `specs/`.
- Don't ask the user about the stack — detect it (scan subdirs for monorepos). Never guess a framework: if it genuinely can't be determined, write `TBD — confirm` and ask the user to confirm.
- Don't write any secret, credential, host, IP, or domain into `context.md` or any committed file — only into `context.local.md` (which is gitignored).
- Don't use the reserved names (`active`, `history`, `context.md`, `context.local.md`, `.gitignore`) for anything other than their defined purpose.

## The grill

Story/feature-focused, NOT an exhaustive design tree. The goal is to **surface and lock the acceptance criteria** — the user's core pain is that ACs get lost mid-build. Crystallize each AC by stress-testing concrete scenarios.

For a **ticket seed**, the ticket already supplies the intent and its stated ACs — don't re-ask those. Aim every question at the **gaps the ticket left unspecified** (the undesigned edges where behavior gets invented). For a **freeform seed**, grill from blank as below.

**Hard rules:**

- **FIXED MAX 5 questions.** Fewer for small features — a tiny tweak may need 1–2. Scale depth to task size; never pad to hit 5.
- **Every question goes through the `Codex question interface` tool** (the UI tool), one focused subject at a time. No free-text interrogation in the chat body.
- **Each question carries a RECOMMENDED option** (mark it, and say why in one line).
- **Explore the codebase to answer what you can** instead of asking. Only spend a question on something the code/context can't settle. A question you could have answered by reading a file is a wasted question.

**What to spend questions on (priority order):**

1. **The core behavior** — what must the system DO. Sharpen the seed into a `SHALL` statement.
2. **The acceptance criteria** — the concrete, testable scenarios that must hold. For each candidate AC, stress-test with a scenario: "WHEN <trigger>, what's the right response — A or B?" Edge cases, failure modes, boundaries. This is where most questions go.
3. **Scope boundaries** — what's explicitly OUT (so it lands in why.md's "Out").
4. **Prerequisites / decisions** — anything that must be settled or approved before `$map` can plan (becomes a Phase-0 gate).

**During the grill:**

- Challenge the seed against the glossary. If the user's word conflicts with `context.md`, surface it in the question's framing and propose the canonical term.
- When a term gets sharpened, note it — you'll write it to `context.md` at the end.
- Keep each AC down to one trigger → one response. If a scenario splits into two outcomes, that's two ACs.
- Stop early once the behavior + ACs are unambiguous. Don't manufacture questions.

## EARS notation for ACs

Each AC is one canonical EARS sentence. Use the pattern that fits the trigger:

- **Event:** `WHEN <trigger> THE SYSTEM SHALL <response>`
- **State-driven:** `WHILE <state> THE SYSTEM SHALL <response>`
- **Conditional:** `IF <condition> THEN THE SYSTEM SHALL <response>`
- **Feature/option:** `WHERE <feature is present> THE SYSTEM SHALL <response>`
- **Ubiquitous:** `THE SYSTEM SHALL <response>` (always-true invariant)

One trigger, one response per AC. No "and also" — split it.

## AC IDs

Each AC gets a stable ID: `<capability>.<feature>.<n>` — e.g. `digest.schedule.1`, `auth.2fa.3`.

- `<capability>` = the spec folder (reuse an existing one if the change fits; else propose a new one, kebab).
- `<feature>` = a short slug for the requirement/feature within the capability.
- `<n>` = sequential within that feature, starting at 1.

## Output

Write into `active/<change-name>/`:

### 1. `why.md`

Use the why template below. Fill:
- **Intent** — what this change is for, in prose (the sharpened seed + why now).
- **Scope — In / Out** — what's covered; what's explicitly excluded (from the scope question).
- **Phase-0 gates** — the prerequisites, decisions, and approvals that must clear before `$map` plans: open decisions surfaced in the grill, external dependencies, anything needing user sign-off. Each gate is one checkable line.

```markdown
<!--
why.md template — active/<change-name>/why.md
The change's intent + scope + the gates $map must clear before planning.
Prose, not strict edit files. The actual spec edits live alongside in requirements/.
-->

# <change-name>

## Intent

<What this change is for and why now — the sharpened seed in prose. The WHY behind the SHALL statements.>

## Scope

**In**
- <what this change covers>

**Out**
- <what is explicitly excluded — keeps the change bounded>

## Phase-0 gates

Prerequisites, decisions, and approvals that MUST clear before `$map` plans. Each is one checkable line.

- [ ] **Decision:** <open question settled in the grill, or still needing sign-off>
- [ ] **Dependency:** <external thing this relies on — must exist/be available first>
- [ ] **Approval:** <anything needing explicit user go-ahead>

<!-- Drop any subsection with no entries. If there are genuinely no gates, say "None." -->
```

### 2. `requirements/<cap>/<req>.md` — one file per new/changed requirement

Mirror the `specs/` layout: `active/<change-name>/requirements/<capability>/<requirement>.md`. Use the requirement template below. Each file:
- A `### Requirement:` heading + one `SHALL` statement.
- The ACs in EARS with their IDs.
- `verified by: TBD` on every AC — the test link is filled in later, once tests exist. Do NOT invent test paths now.
- Optional `Why` / `Dependencies` / `Contracts` sections only where they earn their place.

```markdown
<!--
Requirement file template — specs/<capability>/<requirement>.md
The filename IS the requirement's identity (folder per capability, file per requirement).
ACs are canonical EARS. AC ID = <capability>.<feature>.<n>.
`verified by:` is the ONLY place the AC↔test link lives — natural test names only, no methodology in the work repo.
At $spec time `verified by:` is `TBD`; it's filled once the test exists.
-->

### Requirement: <name>

The system SHALL <behavior — the WHAT, one sentence>.

#### AC <capability>.<feature>.1 — <short name>: WHEN <trigger> THE SYSTEM SHALL <response>
verified by: TBD

#### AC <capability>.<feature>.2 — <short name>: IF <condition> THEN THE SYSTEM SHALL <response>
verified by: TBD

<!--
EARS patterns — pick the one that fits each AC (one trigger → one response; split "and also"):
  WHEN <trigger> THE SYSTEM SHALL <response>          (event)
  WHILE <state> THE SYSTEM SHALL <response>           (state-driven)
  IF <condition> THEN THE SYSTEM SHALL <response>     (conditional / unwanted behavior)
  WHERE <feature present> THE SYSTEM SHALL <response> (optional feature)
  THE SYSTEM SHALL <response>                         (ubiquitous / invariant)

Once a test exists, replace TBD with: file path › "exact test title", e.g.
  verified by: src/auth/__tests__/2fa.test.ts › "returns a JWT when the TOTP is valid"
-->

<!-- Optional sections below — include only where they earn their place. Delete if unused. -->

#### Why
<rationale / trade-off — ADR-flavored, brief>

#### Dependencies
<other requirements or capabilities this relies on>

#### Contracts
<pre/postconditions, invariants — where relevant>
```

If the change MODIFIES an existing requirement, copy the current file from `specs/<cap>/<req>.md` into `requirements/` and edit the whole file (filename = identity; overwrite-in-full is the model). For a rare REMOVE/RENAME, add `active/<change-name>/removals.txt` naming the requirement file(s) to delete.

### 3. `specs/context.md` glossary update

If the grill sharpened any term, append/update it in `specs/context.md`. Glossary only — vocabulary, not implementation decisions, not the spec itself.

## Wrap-up

Report back concisely:
- The `<change-name>` and the capabilities/requirements touched.
- The locked ACs (IDs + one-line each) — so the user can eyeball that nothing's missing.
- The Phase-0 gates `$map` will need cleared.
- Next step: `$map`.

## What NOT to do

- Don't exceed 5 questions, and don't pad to reach 5.
- Don't ask anything the codebase or `context.md` already answers.
- Don't ask questions in the chat body — only via `Codex question interface`.
- Don't write tasks, an implementation plan, or code — that's `$map` / `$imp`.
- Don't fill in `verified by:` with guessed test paths — leave `TBD`.
- Don't put AC IDs, EARS, or any of our methodology into the work codebase (it only lives in `specs/`).
- Don't write secrets or infra identifiers into committed files — name the variable; the value lives in `context.local.md`.
