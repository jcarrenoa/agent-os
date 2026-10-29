---
description: Run Stage 4 (Development) for the active task. Implements all subtasks in dependency order using backend-developer and frontend-developer agents. Requires approved architecture (Stage 3).
allowed-tools: [Read, Write, Glob, Grep, Bash]
---

# Agent OS — Stage 4: Development

You are running Stage 4 (Development) of the agent-os pipeline.

## Pre-Check

1. Read `.agent-os/config.json` and `.agent-os/state.json`.
2. If `current_stage != "development"`:
   - Inform the user which stage is active.
   - If design is in `awaiting_approval`: remind them to run `/agent-os:approve` first.
3. Read `.agent-os/tasks/[active_task_id].json`.
4. Verify `stages.design.status = "done"` and `stages.design.approved_at` is set. If not: stop.

## Run Stage 4

Read `workflows/04-development.md` and follow it completely.

Set `stages.development.status = "in_progress"` and update state.json.

### Subtask Execution

Read the `subtasks` array from the task file. For each subtask in dependency order:

1. Identify the assigned role: `backend-developer` or `frontend-developer`.
2. Spawn the appropriate agent with:
   - The specific subtask details
   - The task file
   - The project config
   - The relevant ADR files from `.agent-os/decisions/`
3. After the agent completes:
   - Update the subtask `status = "done"` in the task file.
   - Append modified/created files to `task.files_modified`.
4. Continue to the next subtask.

### Parallel Subtasks

If the plan specifies subtasks that can run in parallel (no dependency between them), you may spawn both the backend-developer and frontend-developer agents simultaneously for their respective tasks.

### Subtask Failure

If an agent reports it could not complete a subtask:
- Set `task.status = "blocked"`.
- Add a blocker entry to `task.blockers`.
- Update `state.json.stage_status = "blocked"`.
- Report the blocker to the user and ask how to proceed.

## After All Subtasks Complete

1. Set `stages.development.status = "done"`, `completed_at = now`.
2. Add `"development"` to `stages_completed` in state.json.
3. Update `current_stage = "qa"`, `stage_status = "not_started"`.

Report:
```
─────────────────────────────────────────
Stage 4 Complete: Development
─────────────────────────────────────────
Subtasks completed: [N/N]
Files modified: [N]

Next: Stage 5 — QA
  Run `/agent-os:qa` or `/agent-os:start`
─────────────────────────────────────────
```
