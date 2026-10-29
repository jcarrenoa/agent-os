---
description: Roll back the active task to a previous stage. Use when a later stage reveals that earlier work needs to be revised.
argument-hint: <stage> (discovery|planning|design|development)
allowed-tools: [Read, Write]
---

# Agent OS Rollback

You are rolling back the active task to a previous stage.

Target stage: $ARGUMENTS

## Step 1: Validate Request

Read `.agent-os/state.json`.

If `active_task_id` is null:
```
No active task. Nothing to roll back.
```
Stop.

If no stage argument was provided, ask:
```
Which stage do you want to roll back to?
  discovery   — Re-run requirements capture
  planning    — Re-run task breakdown
  design      — Re-run architecture design
  development — Re-run implementation
```
Wait for the user's answer.

Valid rollback targets: `discovery`, `planning`, `design`, `development`.
Cannot roll back to `qa` from `done` (task is already complete — create a new task instead).

If the requested stage is NOT before the current stage:
```
Cannot roll back to [requested] from [current]. [requested] is the same or later.
```
Stop.

## Step 2: Confirm Rollback

This is a destructive operation — it will clear stage data. Ask for confirmation:

```
─────────────────────────────────────────
ROLLBACK CONFIRMATION
─────────────────────────────────────────
Task:       [active_task_id] — [title]
Current:    [current_stage] ([stage_status])
Roll back to: [target_stage]

This will clear progress for these stages:
  [list of stages that will be reset]

Are you sure? (yes/no)
─────────────────────────────────────────
```

**Wait for confirmation.** If user says no, stop.

## Step 3: Execute Rollback

Read `.agent-os/tasks/[active_task_id].json`.

For each stage **after or equal to** the target stage, reset:
```json
"stages": {
  "[stage_name]": {
    "status": "not_started",
    "started_at": null,
    "completed_at": null,
    "approved_at": null,
    "output_summary": ""
  }
}
```

Also clear: `requirements`, `subtasks`, `decisions` (only if rolling back to discovery).
- Rolling back to `discovery`: clear `requirements`, `subtasks`, `decisions`
- Rolling back to `planning`: clear `subtasks`, `decisions`
- Rolling back to `design`: clear `decisions` (keep requirements and subtasks)
- Rolling back to `development`: keep everything, just reset development stage

Update `stages_completed` in state.json: remove all stages from target onward.

Update state.json:
```json
{
  "current_stage": "[target_stage]",
  "stage_status": "not_started",
  "pending_approval": null,
  "blocker": null,
  "last_updated": "[now]"
}
```

## Step 4: Confirm

```
─────────────────────────────────────────
Rolled back to: [target_stage]
─────────────────────────────────────────
[TASK-NNN] is now at [target_stage] (not_started).

Run the stage again:
  /agent-os:start         ← Continue full pipeline from here
  /agent-os:[stage_cmd]   ← Run just this stage
─────────────────────────────────────────
```
