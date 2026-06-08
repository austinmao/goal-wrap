---
name: goal-wrap
description: "Wrap the current goal into a portable handoff for Claude Code or Codex"
version: "1.0.0"
triggers:
  - command: /goal-wrap
  - goal-wrap
  - wrap goal
  - prepare handoff
---

# goal-wrap

Use this when a task should stop as a durable handoff rather than continue in-place.

## Dependencies

- `handoff` skill
- `prompt-master` skill

## Workflow

1. Read the current objective, branch, spec, plan, and task state.
2. Invoke the `handoff` flow to write the durable state bundle.
3. Use `prompt-master` to generate the continuation prompt for the next session or agent.
4. Return a copy-paste-ready handoff block that includes:
   - the handoff file path
   - the current objective
   - the next step
   - the target host if known (`Claude Code` or `Codex`)

## Output contract

- Keep the prompt concise.
- Prefer file paths over re-explaining work already captured in the handoff.
- If the goal is already complete, say so and emit an archive/close-out handoff instead of a continuation prompt.

