# agents

Reusable workflow skills for coding agents.

## Install

Place the repository in your coding agent's standard skills directory. It uses the conventional `skills/<name>/SKILL.md` layout, with no generated configuration or custom installer required.

## Included skills

- `spec` — turn a feature idea into a scoped change with acceptance criteria.
- `map` — create an implementation plan for an active change.
- `tdd` — execute a phase-by-phase red-green workflow.
- `imp` — implement an active mapped change.
- `check` — validate requirements, coverage, and fidelity before finalising.
- `ok` — finalise a clean, accepted change.
- `prompt` — sharpen a task prompt for a coding agent.

The workflow skills are designed to work together, but each remains independently usable.
