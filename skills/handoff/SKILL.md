---
name: handoff
description: "Capture the current goal state into a durable handoff document"
version: "1.0.0"
triggers:
  - command: /handoff
  - handoff
  - prepare handoff
---

# Handoff

Produce a compact, durable state bundle that another session can resume later.

## Workflow

1. Capture the current branch, worktree, spec, plan, and task state.
2. Record completed work, known blockers, and the next action.
3. Save the handoff to a stable file path.
4. Keep the document short enough to read in one pass.

## Handoff shape

- Objective
- Current branch/worktree
- Completed work
- Open blockers
- Files to inspect next
- Exact next command or prompt

