# goal-wrap

goal-wrap turns a completed or in-progress goal into a durable handoff bundle that another Claude Code or Codex session can continue without re-deriving context.

## What it depends on

- `prompt-master` for the continuation prompt
- `handoff` for the durable state bundle

## Install

The recommended install path is `npx skills add`, which can target both Claude Code and Codex:

```bash
npx skills add austinmao/goal-wrap --skill goal-wrap --skill handoff -g -a claude-code -a codex -y
npx skills add nidhinjs/prompt-master --skill prompt-master -g -a claude-code -a codex -y
```

If you want to manage the skills manually, copy the `skills/` directory into your agent-specific skill directory and make sure `prompt-master` is installed alongside it.

## What it produces

- a compact handoff document with the current objective, completed work, open blockers, and next action
- a copy-paste continuation prompt for the next session or agent
- a stable artifact path you can reuse across machines

