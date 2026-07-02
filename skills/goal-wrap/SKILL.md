---
name: goal-wrap
description: "Bundle current work into /goal-runnable prompt with tracked done-when criteria, architecture grounding via research swarm, and full autonomy by default (pass --gates to enable operator confirmation prompts)"
version: "1.1.0"
argument-hint: "[--gates] What should the next session accomplish?"
triggers:
  - command: /goal-wrap
  - goal-wrap
  - wrap goal
  - prepare handoff
---

# /goal-wrap — Grounded, tracked goal bundle

Pack current conversation state into a self-contained execution bundle:
1. **Research swarm first** — Ruflo haiku workers query repowise, gbrain, and git to ground the goal in real architecture
2. **Tracked DONE WHEN** — each criterion has a proof command; executor runs it and updates status; goal clears only when all show `[✓]`
3. **Full autonomy by default** — commits, push, merge, DB migrations, prod deploys, schema changes all pre-approved; pass `--gates` to revert to Claude Code's default ask-first behavior
4. **Ruflo swarm baked in** — goal instructs executor to spawn a swarm for parallel implementation, with declared file/path ownership so workers never collide
5. **Scoped and falsifiable** — a single-sentence GOAL statement plus a NON-GOALS block bound the work; unresolved ambiguity gets a logged `ASSUMPTION:`, never a silent guess

Use before: `/clear`, agent handoff, `/loop`, `/schedule`, switching machines.

## Inputs

`$ARGUMENTS` — task description + optional `--gates` flag.

**Parse flags:**
- `--gates` present → `AUTONOMY=supervised` (ask before irreversible/high-blast actions)
- `--gates` absent → `AUTONOMY=full` (all production actions pre-approved, no confirmation prompts)

## Autonomy Modes

| Mode | Trigger | Pre-approved actions |
|---|---|---|
| `full` (default) | no flag | commits, push, merge to main, DB migrations, prod deploys, schema changes, file deletes, branch ops, env changes |
| `supervised` | `--gates` | none — Claude Code default: ask before each irreversible action |

The AUTONOMY declaration appears in the goal prompt header. The executor reads and honors it with no further prompting.

## Ambiguity Policy

AUTONOMY governs confirmation before *irreversible* actions only. It does NOT
govern what the executor does when spec/plan is silent or unclear about a
non-irreversible implementation detail — that's a separate axis:

| Situation | full | supervised |
|---|---|---|
| Spec silent/unclear on implementation detail (not irreversible) | Pick the most conservative interpretation consistent with DONE WHEN + prior gbrain decisions; log one-line `ASSUMPTION:` in the checkpoint; proceed | Surface as a question before proceeding |
| Irreversible action pre-approved by AUTONOMY=full | No confirmation — but log a one-line rollback hint before acting (see Hard Rules) | N/A — asks first |

Never silently guess without a logged trail. An `ASSUMPTION:` line is
mandatory whenever AUTONOMY=full lets execution continue past an unclear
spec point.

## Workflow

### Phase 0: Architecture Research Swarm

Run before all other steps. Spawn Ruflo swarm (`hierarchical-mesh` topology) with 3 parallel haiku workers then synthesize.

**Spawn:**
```
mcp__ruflo__swarm_init(topology: "hierarchical-mesh")

Worker A (haiku, label: "arch-map"):
  → repowise get_overview()
  → repowise get_answer("key entry points, services, data flows, and hotspot files for this codebase")
  → Returns: architecture summary, hotspot file list, service topology

Worker B (haiku, label: "gbrain-recall"):
  → scripts/gbrain-openclaw.sh query "<task description from $ARGUMENTS>"
  → scripts/gbrain-openclaw.sh search "<key noun phrases from $ARGUMENTS>"
  → Returns: prior decisions, resolved blockers, known patterns, spec history

Worker C (haiku, label: "spec-scan"):
  → TaskList (open tasks count + ids)
  → find specs -maxdepth 3 -name 'spec.md' | xargs ls -t 2>/dev/null | head -3
  → git log -10 --oneline
  → git diff origin/main...HEAD --stat
  → Returns: open task count/ids, active spec list, recent change summary
```

**Synthesize (sonnet) → ARCH_CONTEXT block (max 500 chars):**
```
ARCH:
entry: <key files/routes>
services: <list>
spec: <active spec + SHA>
decisions: <1-2 relevant prior decisions>
hotspots: <repowise-flagged files>
tasks: <count> (<ids>)
open: <unresolved question the swarm couldn't answer — omit line if none>
```

If swarm fails to spawn: fall back to sequential repowise + gbrain queries. Log `SWARM_FAILED` in bundle output.

### Step 1: Gather context (no writes yet)

Detect:
- Primary spec: `find specs -maxdepth 2 -name 'spec.md' -printf '%T@ %p\n' 2>/dev/null | sort -rn | head -1 | cut -d' ' -f2-`
- Primary plan: same dir, `plan.md`
- Active branch: `git branch --show-current`
- Worktree: `git worktree list | grep "$(pwd)"`
- Recent commits: `git log -5 --oneline`
- Spike/research artifacts: `find docs/artifacts \( -name '*-spike-*.md' -o -name '*-research-*.md' -o -name '*-findings-*.md' \) -printf '%T@ %p\n' 2>/dev/null | sort -rn | head -5 | cut -d' ' -f2-`
- ADRs touched: `git diff origin/main...HEAD -- 'docs/adr/**' --name-only`
- Skills invoked this session: scan conversation for `Skill` tool calls
- Outstanding operator decisions: review last 10 `AskUserQuestion` answers
- Out-of-scope / non-goals: grep spec.md for "Out of Scope"/"Non-Goals" heading

If NO `spec.md` + `plan.md` found:
- ASK operator: "No spec/plan found. Anti-drift baseline cannot be established. (A) run `/office-hours` + `/speckit.specify` first, (B) point to existing reference doc, (C) override 'no baseline' — autonomous run has nothing to drift-check against."

### Step 1a: Derive single-sentence GOAL statement

From $ARGUMENTS + spec.md acceptance criteria, write ONE falsifiable
sentence naming the measurable end-state (e.g., "POST /api/x returns 200
with the new field for all existing callers"). Reject compound sentences
joined by "and" that hide two conditions — split the second condition into
a DONE WHEN item instead. Max 120 chars.

### Step 2: Capture anti-drift SHAs

```bash
SPEC_SHA=$(git log -1 --format=%h -- "$SPEC_PATH" 2>/dev/null || echo "uncommitted")
PLAN_SHA=$(git log -1 --format=%h -- "$PLAN_PATH" 2>/dev/null || echo "uncommitted")
# AUTO-COMMIT if uncommitted or dirty — on feature branch only, never ask
if [ "$SPEC_SHA" = "uncommitted" ] || [ "$PLAN_SHA" = "uncommitted" ] || ! git diff --quiet -- "$SPEC_PATH" "$PLAN_PATH"; then
  CURRENT_BRANCH=$(git branch --show-current)
  if [ "$CURRENT_BRANCH" = "main" ] || [ "$CURRENT_BRANCH" = "master" ]; then
    echo "STOP: on $CURRENT_BRANCH — branch first, then re-run /goal-wrap"
    exit 1
  fi
  git add "$(dirname "$SPEC_PATH")"
  git commit -q -m "docs(spec): commit spec/plan for goal-wrap anti-drift baseline"
  SPEC_SHA=$(git log -1 --format=%h -- "$SPEC_PATH")
  PLAN_SHA=$(git log -1 --format=%h -- "$PLAN_PATH")
fi
```

### Step 3: Build tracked DONE WHEN criteria

Extract acceptance criteria from `spec.md` (success_criteria, acceptance criteria, or DONE WHEN section).
Build a structured list — each item: `STATUS | DESCRIPTION | proof: BASH_COMMAND`.

**Proof command rules:**
- Must exit 0 when the criterion is proven true
- Must be runnable non-interactively from repo root
- Must NOT be "I verified X" — always a real command

**Status states (executor updates these):**
- `[ ] pending` → `[→] in_progress` → `[✓] verified` (proof ran + exit 0 + output captured)
- `[✗] blocked` — proof failed or prerequisite missing

**Template:**
```
DONE WHEN:
[ ] pending | <measurable outcome 1 from spec> | proof: <bash cmd>
[ ] pending | <measurable outcome 2 from spec> | proof: <bash cmd>
[ ] pending | All e2e tests pass | proof: <e2e test command for this repo>
[ ] pending | All unit + integration tests pass | proof: <unit test command for this repo>
[ ] pending | No spec/plan drift | proof: git diff <SPEC_SHA>..HEAD -- <SPEC_PATH> <PLAN_PATH> | wc -l | grep -qx 0
[ ] pending | No open in-progress tasks | proof: TaskList count in_progress == 0
[ ] pending | No existing test files deleted | proof: git diff <SPEC_SHA>..HEAD --diff-filter=D --name-only -- '**/*.test.*' '**/*.spec.*' | wc -l | grep -qx 0
```

Loosened assertions (not deletions) aren't mechanically provable here — the qualitative "do not edit tests to force a pass" constraint is fed to prompt-master (Step 4) and enforced downstream by `campaign-tdd`/`tdd-guide`/`codex-gate`, not re-verified by goal-wrap itself.

Goal clears ONLY when ALL items show `[✓]`. Executor must run proof commands; never self-report done without exit 0.

### Step 3a: Build NON-GOALS block

Extract explicit out-of-scope/non-goals text from spec.md if present. If
absent, infer conservative boundaries from the spec's stated scope (e.g.,
"does not include <adjacent area>") and mark the block `INFERRED` so the
operator can correct it. Max 150 chars, 2-3 bullet items.

### Step 4: Invoke /prompt-master for goal body

Call `Skill(prompt-master, args=<below spec>)`:

```
Draft a /goal-runnable prompt in the 1900-2250 char range for autonomous execution. Context:
- Repo: <repo-name>
- Branch: <branch>
- AUTONOMY: <full|supervised>
- Spec baseline: <SPEC_PATH> @ <SPEC_SHA>
- Plan baseline: <PLAN_PATH> @ <PLAN_SHA>
- Objective: <$ARGUMENTS task description>
- Architecture context: <ARCH_CONTEXT from Phase 0>
- Hard constraints:
  * AUTONOMY=<mode>: [full = all prod/DB/merge/deploy pre-approved, no confirmation prompts] [supervised = ask before irreversible actions]
  * Spawn Ruflo swarm (mcp__ruflo__swarm_init, hierarchical-mesh) for all parallel work (3+ tasks)
  * haiku workers for mechanical tasks, sonnet for orchestration, opus for architecture/security decisions
  * Use repowise (get_context, get_answer, get_risk) for codebase questions — do NOT re-search from scratch
  * Use gbrain (`scripts/gbrain-openclaw.sh query/search`) for prior decisions — do NOT re-derive
  * Full test suite (unit + integration + e2e) MUST pass before marking any DONE WHEN [✓]
  * Run proof command from DONE WHEN; capture exit code + output before updating status
  * Anti-drift: every action traces to spec @ <SPEC_SHA>. Drift = STOP + emit blocker checkpoint
  * Ambiguity in spec/plan (non-irreversible details): pick conservative interpretation consistent with DONE WHEN + prior gbrain decisions; log one-line ASSUMPTION in checkpoint; never silently guess
  * Before any irreversible action (commit/push/merge/migration/deploy/delete): log a one-line rollback hint (revert SHA placeholder, down-migration path, rollback command) BEFORE acting — this replaces asking under AUTONOMY=full, not the safety trail
  * Do NOT modify or delete existing tests to force a pass. A failing test is a blocker — flag it, do not loosen assertions or delete it
  * Parallel implementation workers MUST declare file/path ownership before dispatch; no overlapping claims. For spec-level parallelism, route to separate git worktrees per CLAUDE.md Worktree Convention — do not reinvent isolation logic
  * NON-GOALS (see below) are out of scope — do not silently expand into them
  * Checkpoint after each DONE WHEN status change: {changed_item, new_status, proof_output, blockers, next_action, assumptions_logged, rollback_hints_logged}

Output ONLY the prompt body (no /goal wrapper, no markdown fence). Target 1900-2250 chars.
```

Verify returned body is in the 1900-2250 char range. Compress (max 2 retries) if over.

### Step 5: Assemble /goal body

```
GOAL: <single falsifiable sentence — the one measurable objective; never two ANDed conditions>

PROJECT: <repo>
BRANCH: <branch>  AUTONOMY: <full|supervised>
SPEC: <SPEC_PATH> @ <SPEC_SHA>  [anti-drift baseline]
PLAN: <PLAN_PATH> @ <PLAN_SHA>  [anti-drift baseline]

<ARCH_CONTEXT — max 500 chars, may include `open:` line>

NON-GOALS (out of scope — do not expand into these without a new /goal-wrap):
<NON-GOALS items — max 150 chars, marked INFERRED if not extracted from spec>

<prompt-master output — max 2250 chars>

HANDOFF DOC: <path from Step 6>

DONE WHEN (executor: run proof cmd, capture exit+output, then update status):
<DONE WHEN items from Step 3>

SWARM: mcp__ruflo__swarm_init(topology: hierarchical-mesh) → haiku workers for parallel tasks (each MUST declare file/path ownership before dispatch; no overlapping claims — route spec-level parallelism to separate git worktrees per CLAUDE.md Worktree Convention) → sonnet synthesis
TOOLS: repowise get_context/get_answer/get_risk for codebase; gbrain query/search for decisions; TaskList for task state
```

Verify total ≤4000 chars: `printf '%s' "$GOAL_BODY" | wc -c`

Final form: `/goal "<GOAL_BODY>"` ready to paste.

### Step 6: Invoke /handoff

Call `Skill(handoff, args=<$ARGUMENTS>)`. /handoff writes to OS temp dir per its spec. Capture path. Inject into GOAL_BODY at `HANDOFF DOC:` line.

Read handoff doc and verify it references (does not duplicate): spec+plan by SHA, ADRs by path, commits by sha+subject, spike findings by path+1-line summary, task count+ids, outstanding decisions 1-line each.

If handoff inlines large content from referenceable artifacts: flag as bug to operator.

### Step 7: E2E framework check

```bash
HAS_E2E=0
{ [ -f playwright.config.ts ] || [ -f playwright.config.js ] || \
  [ -f cypress.config.ts ] || [ -f cypress.config.js ] || \
  grep -q '"test:e2e"' package.json 2>/dev/null || \
  [ -d web/e2e ] || [ -d tests/e2e ] || [ -d e2e ]; } && HAS_E2E=1
```

If absent: ASK operator: "(A) add playwright setup as part of goal work (adds scope), (B) replace e2e gate with manual operator verification in DONE WHEN, (C) abort bundle until e2e committed."

### Step 8: Surface bundle

```
=== /goal-wrap bundle ready ===
GOAL: <one-liner>
AUTONOMY: <full — all prod/DB/merge/deploy pre-approved | supervised — ask before irreversible>
AMBIGUITY POLICY: <full — assumptions logged, no asks | supervised — ambiguity surfaced as questions>
RESEARCH SWARM: ✓ <N> workers completed (open questions: <N>) | ✗ SWARM_FAILED — fell back to sequential

GOAL PROMPT (<NNNN>/4000 chars):
──────────────────────────────────────
/goal "<GOAL_BODY>"
──────────────────────────────────────

HANDOFF DOC: <temp_path>

ANTI-DRIFT BASELINE:
  spec: <SPEC_PATH> @ <SPEC_SHA>
  plan: <PLAN_PATH> @ <PLAN_SHA>

NON-GOALS: <list, marked INFERRED if not extracted from spec>

DONE WHEN (<N> items, all pending):
  [ ] <criterion 1>  (proof: <cmd>)
  [ ] <criterion 2>  (proof: <cmd>)
  [ ] All e2e tests pass  (proof: <cmd>)
  [ ] All unit+integration tests pass  (proof: <cmd>)
  [ ] No spec/plan drift  (proof: git diff ...)
  [ ] No open in-progress tasks  (proof: TaskList ...)

ARCHITECTURE GROUNDING:
  <ARCH_CONTEXT brief — entry, services, hotspots, prior decisions recalled>

REFERENCES (in handoff doc):
  ADRs: <N> | Commits: <N> | Spikes: <N> | Tasks: <N> | Decisions: <N>

E2E STATUS: <ok | gap-flagged>

NEXT:
  Paste /goal in fresh session → executor spawns swarm, runs proof cmds, updates DONE WHEN
  OR: /loop 30m /goal "..." for autonomous run
  OR: /schedule one-shot at <time>
```

## Hard Rules

- Goal body ≤4000 chars. Budgets: GOAL 120, HEADER (PROJECT/BRANCH/AUTONOMY/SPEC/PLAN) 220, ARCH_CONTEXT 500, NON-GOALS 150, DONE WHEN 420, prompt body 2250, META (HANDOFF/SWARM/TOOLS) 200. (sum 3860, ~140 buffer for section delimiters/whitespace)
- DONE WHEN items MUST have proof commands. No proof = flag item; do not emit goal until fixed.
- AUTONOMY=full is default. Executor honors it — no confirmation for commits, push, merge, DB, deploys. `--gates` is the only opt-in to prompts.
- Research swarm MUST run (Phase 0) before prompt-master. Architecture grounding is not optional.
- Anti-drift SHA = committed state. AUTO-COMMIT on feature branch, never ask. STOP only on main/master.
- NEVER include secrets. Redact API keys, passwords, PII in both goal and handoff doc.
- ALWAYS invoke /handoff (Step 6) and /prompt-master (Step 4). Do NOT hand-craft or inline their outputs.
- Goal references handoff path. Executor fetches full context from handoff, not from goal body.
- GOAL statement must be a single, falsifiable, binary condition — never two ANDed outcomes in one sentence.
- NON-GOALS block is mandatory in every goal body. If spec.md has no explicit out-of-scope section, infer conservative boundaries and mark INFERRED.
- Ambiguity about implementation detail (not irreversible actions) is governed independently of AUTONOMY: full mode logs an ASSUMPTION line and proceeds; supervised mode surfaces it as a question. Never silently guess without a logged trail.
- Parallel implementation workers spawned by the executor MUST declare file/path ownership before dispatch; no two workers may claim overlapping paths. Route spec-level parallelism to separate git worktrees per CLAUDE.md Worktree Convention.
- DONE WHEN and prompt-master body MUST forbid modifying/deleting existing tests to force a pass; a failing test is a blocker, not something to edit into passing.
- Before every AUTONOMY=full irreversible action, executor logs a one-line rollback hint BEFORE acting — logging replaces asking, it does not replace the safety trail.

## Failure Modes

| Failure | Response |
|---|---|
| No spec/plan | BLOCK + ask (A/B/C) |
| Phase 0 swarm fails | Sequential fallback; log `SWARM_FAILED` in bundle |
| /prompt-master outside 1900-2250 char range (2 retries) | Surface with WARNING; operator decides |
| DONE WHEN item missing proof cmd | Flag each; block goal emission until fixed |
| /handoff unavailable | Inline minimal handoff to OS temp dir + warn |
| E2E framework absent | ASK (A/B/C) |
| On main/master with uncommitted spec | STOP; tell operator to branch first |
| Secrets in goal or handoff | REDACT; log count |
| No out-of-scope section in spec.md | Infer conservative NON-GOALS from stated scope; mark INFERRED |
| GOAL statement bundles >1 verifiable condition | Split into GOAL + secondary DONE WHEN item; re-derive single-sentence form |
| Parallel workers claim overlapping file paths | Reassign/serialize before dispatch; route to separate git worktrees per CLAUDE.md Worktree Convention |
| DONE WHEN item would require editing/deleting existing tests to pass | Flag as blocker; do not emit as valid proof path |
| Irreversible action logged without rollback hint | Treat as Hard Rule violation; surface in bundle as COMPLIANCE GAP |

## What this skill does NOT do

- Does NOT execute the goal (handoff to fresh session or runner)
- Does NOT run proof commands (executor does, inside the goal loop)
- Does NOT modify spec/plan (read-only)
- Does NOT push (operator action)
- Does NOT trigger /loop or /schedule (suggests only)

## Related skills

- `/prompt-master` — Step 4 dependency
- `/handoff` — Step 6 dependency
- `/swarm` — embedded in goal for executor
- `/loop` / `/schedule` — downstream execution
- `/office-hours` / `/speckit.specify` — upstream (create spec if missing)
