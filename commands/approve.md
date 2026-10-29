---
description: Approve a pending approval gate and advance the pipeline to the next stage. Use this when the pipeline is paused at an "awaiting_approval" gate.
argument-hint: [notes]
allowed-tools: [Read, Write]
---

# Agent OS Approve

You are approving a pending pipeline gate and advancing to the next stage.

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

If arguments were provided (`$ARGUMENTS`), record them as approval notes.

## Step 3: Record Approval

Read `.agent-os/tasks/[active_task_id].json`.

Update the current stage record:
```json
"stages": {
  "[current_stage]": {
    "status": "done",
    "approved_at": "[now ISO timestamp]"
  }
}
```

Write the updated task file.

Update `.agent-os/state.json`:
- Add `current_stage` to `stages_completed`
- Set `current_stage = [next_stage from pending_approval]`
- Set `stage_status = "not_started"`
- Set `pending_approval = null`
- Set `last_updated = [now]`

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
