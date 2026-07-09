---
name: agent-loop-lite
description: >-
  Lightweight Agent Loop for explicit $agent-loop-lite use: run small-to-medium
  tasks through a minimal auditable flow with task, plan, mandatory Subagent
  Check, execution evidence, manifest when useful, and a concise gate summary.
  Use for parameter sweeps, local experiments, bounded code changes, script
  runs, and tasks that need reproducible evidence without full DAG/scheduler
  orchestration. It may use a small subagent when explicitly requested or when
  there is clear independent evidence value, but it must not call
  subagent-plan-decomposer or create full-loop scheduler/reviewer ceremony.
---

# Agent Loop Lite

## Goal

Run a minimal auditable loop:

```text
goal -> lightweight plan -> Subagent Check -> execution -> evidence -> gate summary -> user judgment
```

Principles:

```text
Save evidence, not ceremony.
Always consider subagents; do not always use subagents.
Keep only what helps reproduction, acceptance, handoff, or risk control.
Do not create process history just to prove the process happened.
```

## Non-Goals

Do not turn this skill into full-loop orchestration.

By default, do not:

- call `$subagent-plan-decomposer`
- create `decomposed_plan.md`
- create `scheduler_state.md`
- create a full subagent DAG
- create reviewer-agent workflow
- create standalone join reports for small steps
- write historical reports for ordinary naming fixes, minor formatting, or scheduling actions

If these become necessary, stop and recommend switching to `$agent-loop-full`. Do not silently upgrade.

## Default Files

Default state files:

```text
loop_state/
  task.md
  plan.md
  gate_summary.md
```

Optional only when needed:

```text
loop_state/subagent_outputs/<short-name>.md
```

For experiments or repeated runs, create a run output directory:

```text
outputs/<run_id>/
  manifest.csv
  manifest.jsonl
  images / logs / result files
```

## Retention Test

Before writing any intermediate file, it must satisfy at least one condition:

```text
1. Reproduction: without it, a future run cannot be repeated or explained.
2. Acceptance: without it, PASS / FAIL / UNKNOWN / BLOCKED cannot be judged.
3. Handoff: the task may pause, span sessions, or require another agent/person.
4. Risk: the task touches deletion, deployment, privacy, permissions, production data, or model weights.
```

If none apply, summarize the point in `gate_summary.md` or the final reply instead of creating another file.

## Artifact Budget

Default budget:

```text
state files: <= 3
optional subagent output files: <= 2
pre-execution plan: <= 120 lines
final gate summary: <= 180 lines
```

If the budget is exceeded, explain why in `gate_summary.md`:

```text
Artifact budget exceeded because:
- ...
```

Do not exceed the budget just to look more rigorous.

## Step 1: Task

Create or update `loop_state/task.md`.

Keep it short:

```markdown
# Task

## User Request
Brief summary of the original request.

## Real Goal
What must actually be achieved.

## Scope
What this run will do.

## Non-Goals
What this run will not do.

## Acceptance
- ...

## Evidence
Files, logs, metrics, images, tests, or other artifacts that prove completion.

## Side Effects
Allowed write locations; forbidden writes; whether remote calls, deletion, overwrite, or upload are forbidden.

## Human-Only
Judgments that must remain with the user.
```

If the task can finish in one turn and does not need state recovery, keep `task.md` minimal. Do not expand it into a PRD.

## Step 2: Plan

Create or update `loop_state/plan.md`.

Only include information needed to execute:

```markdown
# Plan

## Steps
1. ...
2. ...
3. ...

## Fixed Variables
Conditions that must not change.

## Varying Variables
Conditions that will change.

## Outputs
Expected artifacts.

## Subagent Check
Use the required template below.

## Gate
How to judge completion, failure, unknowns, or blockers.
```

For experiment tasks, include:

- parameter matrix
- fixed variables
- varying variables
- output directory
- manifest fields
- human-only quality checks

Do not write a scheduler DAG.

## Step 3: Subagent Check

Subagent Check is mandatory. Do not omit it and do not write only "no subagent needed".

Write it in `loop_state/plan.md`:

```markdown
## Subagent Check

### User Explicitly Requested Subagent
yes / no

### Decision
use / do-not-use

### Checklist
| Question | Answer | Evidence |
|---|---|---|
| Did the user explicitly request subagent? | yes/no | original request |
| Is there an independent read-only audit task? | yes/no | task boundary |
| Is there an independent evidence-generation task? | yes/no | expected artifact |
| Is there a high-risk judgment needing independent review? | yes/no | risk |
| Would sequential main-agent work pollute judgment or overload context? | yes/no | input size |
| Is there parallel work with non-overlapping write sets? | yes/no | read/write sets |
| Is there a shared bottleneck that makes subagent execution unhelpful? | yes/no | GPU / manifest / runner / single output dir |

### If Use
- Subagent:
- Purpose:
- Input:
- Output:
- Read set:
- Write set:
- Forbidden:
- Merge rule:
- Save output? yes/no, because ...

### If Do Not Use
- Reason:
- What would trigger reconsideration:
```

Decision rules:

```text
If the user explicitly requested subagent:
- Use it by default.
- Do not use it only when there is write-set conflict, shared bottleneck, no independent acceptance path, or increased risk.
- If not using it, write the block reason.

If the user did not explicitly request subagent:
- Use it only when there is clear independent evidence value.
- "More rigorous", "might be faster", or "task is complex" is not enough.
- If all execution shares the same GPU, runner, manifest, or output directory, do not split execution itself to subagents by default.
```

Good subagent uses:

- read-only audit of an independent directory or report
- independent manifest/image coverage check
- independent evidence generation with no write conflict
- objective artifact coverage before human review

Bad subagent uses:

- creating a worker for ordinary small steps
- pretending a single-GPU serial sweep can be parallelized
- creating a reviewer for naming or formatting issues
- letting a subagent write the same manifest, runner, or state file as the main agent
- using subagents because it looks more rigorous

Subagent results are summarized in `gate_summary.md` by default. Save to `loop_state/subagent_outputs/<short-name>.md` only when the output is itself decision evidence.

## Step 4: Execute

Execute according to `plan.md`.

Rules:

- Prefer the smallest sufficient change.
- Do not add new scope.
- Do not modify model weights unless explicitly requested.
- Do not delete existing files.
- Do not overwrite an existing output directory; create a new run id.
- Record failures in the manifest or `gate_summary.md`.
- Do not mark human-only judgment as PASS.
- If the task exceeds the Artifact Budget or needs full scheduling, stop and recommend `$agent-loop-full`.

For experiment tasks, write a manifest. Recommended fields:

```text
run_id
group
variant
fixed_config
changed_config
command_or_entrypoint
status
runtime
peak_memory_if_available
output_path
error_if_any
```

## Step 5: Gate Summary

Create `loop_state/gate_summary.md`.

This is the only final acceptance file in lite mode:

```markdown
# Gate Summary

## Result
accept / revise / block

## Completion
PASS:
- ...

FAIL:
- ...

UNKNOWN:
- ...

BLOCKED:
- ...

## Evidence
- output:
- manifest:
- logs:
- tests:

## Subagent Evidence
- not used, because ...
or
- <subagent-name>: <key conclusion and evidence path>

## What Changed
What was actually done.

## What Did Not Change
What was explicitly not changed.

## Human-Only
What the user must judge.

## Risks / Gaps
What this result still does not prove.

## Next Step
The shortest useful next action.
```

Do not write:

```text
basically done
mostly done
should be fine
```

Use explicit status:

```text
PASS / FAIL / UNKNOWN / BLOCKED
```

## Final Reply

Keep the final reply short and link the key artifacts:

```text
Result:
Artifacts:
Verification:
Subagent:
Unfinished / human-only:
Next:
```

Do not replay the whole process to the user.

## Switch to Full Loop

Do not switch automatically. Ask or recommend switching to `$agent-loop-full` when needed.

Signals:

- the user asks to loop until acceptance conditions are met
- multiple modules need parallel development with clear write-set isolation
- a high-risk change needs independent reviewer gate
- deletion, deployment, permissions, privacy, production data, or formal release is involved
- the task exceeds the lite Artifact Budget due to real complexity, not over-recording
- Subagent Check shows multiple subagents, join gates, or a scheduler are needed

Before switching, say:

```text
This task has exceeded lite-loop's evidence budget. Continuing in lite mode would lose important handoff or acceptance information; I recommend switching to $agent-loop-full.
```

## Quality Bar

- Save evidence, not ceremony.
- Planning serves execution, not performance.
- Always perform Subagent Check.
- Use subagents only with clear evidence value or explicit user request.
- Prefer manifests over process reports.
- Prefer one gate summary over multiple join/gate/decision files.
- Keep human-only judgment with the user.
- Do not interpret local success as product readiness.
