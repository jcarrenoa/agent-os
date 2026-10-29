---
description: Run Stage 5 (Testing & QA) for the active task. Verifies requirements coverage, runs tests, checks for regressions. Closes the task if all checks pass.
allowed-tools: [Read, Write, Glob, Grep, Bash]
---

# Agent OS — Stage 5: QA

You are running Stage 5 (Testing & QA) of the agent-os pipeline.

## Pre-Check

1. Read `.agent-os/config.json` and `.agent-os/state.json`.
2. If `current_stage != "qa"`:
   - If earlier: tell the user which stages need to be completed first.
3. Read `.agent-os/tasks/[active_task_id].json`.
4. Verify `stages.development.status = "done"`. If not: stop.

## Run Stage 5

Read `workflows/05-testing-qa.md` and follow it completely.

Set `stages.qa.status = "in_progress"` and update state.json.

Spawn a `qa-engineer` agent with:
- The full task file
- The project config

After the agent returns:

### If APPROVE

No critical or major issues:
1. Set `stages.qa.status = "done"`, `completed_at = now`.
2. Update `task.test_results` from QA report.
3. Add `"qa"` to `stages_completed`.
4. Set `task.status = "done"`, `task.completed_at = now`.
5. Set `state.json`:
   ```json
   {
     "current_stage": "idle",
     "stage_status": "done",
     "active_task_id": null,
     "pending_approval": null,
     "blocker": null
   }
   ```
6. Output completion summary.

### If REQUEST_CHANGES

Critical issues found:
1. Set `stages.qa.status = "awaiting_approval"`.
2. Present QA report to the user.
3. Ask: "QA found critical issues. How would you like to proceed?"
   - "Fix now" → set `current_stage = "development"`, `stage_status = "not_started"`, clear development stage in task. User can then run `/agent-os:develop`.
   - "Accept and close" → record acceptance and close the task (mark with QA notes).
   - "Create new task for fixes" → close this task as done, advise user to create a new task for the remaining issues.
4. **Wait for user's decision.**

## Completion Output

```
─────────────────────────────────────────
TASK COMPLETE: [TASK-NNN]
─────────────────────────────────────────
[title]

Pipeline:
  ✓ Discovery   — [summary]
  ✓ Planning    — [N subtasks]
  ✓ Design      — [N ADRs]
  ✓ Development — [N files]
  ✓ QA          — [N tests passed]

Decisions: [DEC-NNN, ...]

Run `/agent-os:status` for full history.
─────────────────────────────────────────
```
