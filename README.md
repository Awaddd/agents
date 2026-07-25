# agents

Reusable workflow skills for coding agents.

## Install

Place the repository in your coding agent's standard skills directory. It uses the conventional `skills/<name>/SKILL.md` layout, with no generated configuration or custom installer required.

## Included skills

`prompt` is independent. The remaining skills form a durable, acceptance-criteria-driven workflow:

```text
spec → map → imp (runs tdd) → ok (runs check)
```

- `spec` turns an idea into scoped requirements and acceptance criteria.
- `map` converts those requirements into a phased implementation plan.
- `imp` executes the plan through `tdd`'s red-green protocol.
- `check` verifies that each acceptance criterion has a passing, faithful proof.
- `ok` reconciles the verified result into the durable specification and archives the completed change.

`tdd` can be invoked directly when a compatible plan already exists. `check` is also useful during implementation, but `map`, `tdd`, `imp`, and `ok` depend on the artifacts created by the earlier stages.

## Why the workflow keeps durable artifacts

The workflow writes a small, committed `specs/` tree alongside the code:

```text
specs/
  context.md                 shared stack conventions and glossary
  active/<change>/           intent, requirements, and implementation plan in progress
  <capability>/              durable requirements that describe verified behaviour
  history/<date>-<change>/   completed change record
```

This gives a new agent or a resumed session the same decisions, scope, and acceptance criteria that guided the original work. `context.local.md` is reserved for local-only values and is ignored.

The practical benefits are:

- Requirements survive context resets and handoffs instead of living only in chat.
- Every acceptance criterion is linked to a real test once it is implemented.
- The `check` gate catches missing, weak, or stale proof before a change is finalised.
- The final specification records verified behaviour rather than an aspirational plan.
- Completed changes retain their intent and decisions without cluttering the working specification.
