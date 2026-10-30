---
description: Approve a pending approval gate and advance the pipeline to the next stage. Use this when the pipeline is paused at an "awaiting_approval" gate.
argument-hint: [notes]
allowed-tools: [Read, Write, AskUserQuestion]
---

# Agent OS Approve

You are approving a pending pipeline gate and advancing to the next stage.

For a view of the pending gate with the approver verdict, the history, and actions to approve, request changes or see the details, use `/agent-os:review`.

## Step 1: Read Current State

Read `.agent-os/state.json`.

If `stage_status` is NOT `"awaiting_approval"`:
```
There is no pending approval.
Current stage: [current_stage] — [stage_status]
```
Stop.

Read `pending_approval` from state.json. Identify:
- `gate`: which gate is pending
- `context`: what is being approved
- `next_stage`: what stage to advance to

## Step 2: Confirm Approval

Show the user what they are approving:

```
─────────────────────────────────────────
Approving: [gate name]
Stage:     [current_stage] → [next_stage]

[context from pending_approval]
─────────────────────────────────────────
```

If `pending_approval.approver_verdict` is not null, read the task file and add the approver verdict block (Presenting the Approver Verdict in the `approval-gates` skill) for `stages.[current_stage].review`. The approver's verdict is informative: this command approves even with `REQUEST_CHANGES`.

If arguments were provided (`$ARGUMENTS`), record them as approval notes.

## Step 3: Record Approval

Read `.agent-os/tasks/[active_task_id].json`.

Follow **How to Record an Approval** in the `approval-gates` skill with `approved_by = "human"`: the stage record gets `status = "done"`, `approved_at = now`, `approved_by = "human"`; state.json adds `current_stage` to `stages_completed`, sets `current_stage = [next_stage from pending_approval]`, `stage_status = "not_started"`, `pending_approval = null` and `last_updated = now`. `/agent-os:review` uses the same steps.

## Step 4: Advance Pipeline

Tell the user which stage is next and how to run it:

```
─────────────────────────────────────────
Approved. [current_stage] is complete.

Next: Stage [N] — [next_stage name]

Run `/agent-os:start` to continue the full pipeline,
or `/agent-os:[next_stage_command]` to run just this stage.
─────────────────────────────────────────
```

The commands mapping:
- `planning`    → `/agent-os:plan`
- `design`      → `/agent-os:design`
- `development` → `/agent-os:develop`
- `qa`          → `/agent-os:qa`
- `done`        → task is complete
