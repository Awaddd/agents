---
name: prompt
description: Sharpens a vague or undisciplined prompt into a tight, deliberate one. Use whenever the user asks to improve a prompt, fix a prompt that isn't working, write a better instruction for Codex or another model, or rewrite an instruction so it is clearer and more predictable. Invoke explicitly with `$prompt`; also trigger on "help me prompt", "this prompt isn't working", "improve my prompt", "make this clearer", "tighten this prompt", or a rough instruction the user wants sharpened.
---

## What this skill does

Take a raw or undisciplined prompt and return a sharpened version — plus a short explanation of what changed and why.

The goal is not to make the prompt longer. It's to make it more deliberate: one job, clear output shape, instruction before material.

## Five principles to apply

1. **Instruction first** — the task goes at the top, before any context or source material
2. **State task + output format** — what to do, and exactly what to return (bullets, code block, table, single sentence, etc.)
3. **Separate instructions from material** — don't mix "here's what I want" with "here's the data"; use a clear break (`---`, `<content>`, or a labeled section)
4. **Add an example when the task is style-sensitive or ambiguous** — one concise example beats a paragraph of description
5. **Ask for what you want, not what you don't** — positive framing is more reliable than a list of things to avoid

## Additional rules for agent tasks and handoffs

When sharpening a prompt for an agent, make these explicit when they apply:

- **Responsibility** — the outcome the agent owns
- **Frozen decisions** — choices already made that the agent must preserve
- **Remaining discretion** — choices the agent may make independently
- **Scope** — included work and meaningful exclusions
- **Acceptance** — the observable conditions for completion

If evaluation is part of implementation, instruct the agent to fix failures and recheck the result; do not leave the task at critique or diagnosis alone.

For a prompt-file handoff, include this instruction verbatim: “Read and execute this file as the task instructions.”

## Output format

Return exactly three sections, nothing more:

**Diagnosis** — one or two sentences. What's undisciplined about the original: too vague, no output shape, multiple jobs mixed together, instruction buried in context.

**Sharpened prompt** — the rewrite. Keep it short. If the original was bloated, cut it.

**What changed** — a tight list of the specific moves made (e.g. "moved instruction to top", "added output format", "split into two separate prompts", "removed negative framing").

## Example

**Original:** "Help me with this codebase and be concise."

**Diagnosis:** No task specified, no output format, "be concise" is a style note not a job.

**Sharpened prompt:**
```
Review this function and identify the bug. Explain the cause in 3 bullets, then give a minimal patch in a code block.

[paste function here]
```

**What changed:**
- Moved concrete task to the top
- Added output format (3 bullets + code block)
- Removed "be concise" — the format constraint makes brevity implicit
- Separated instruction from where the material goes

## When to ask before rewriting

If the original prompt is so vague you can't tell what the job is — ask one question: "What should the output look like?" Don't guess at intent.

If the task is clear but the output format is missing, infer a reasonable one and note it in "what changed."
