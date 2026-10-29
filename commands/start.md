---
description: Start a new task and run it through the complete 5-stage pipeline (Discovery → Planning → Design → Development → QA). Pauses at approval gates and waits for your confirmation before advancing.
argument-hint: "<task description>"
allowed-tools: [Bash, Read, Write, Glob, Grep]
---

# Agent OS — Full Pipeline

You are running a task through the complete agent-os 5-stage pipeline.

Task description: $ARGUMENTS

## Language

After confirming `.agent-os/config.json` exists, read `preferences.output_language` and write ALL pipeline messages, gate prompts, and status updates in that language (`"en"` = English, `"es"` = Spanish).

## Pre-Check

1. Verify `.agent-os/config.json` exists. If not: tell the user to run `/agent-os:setup` first and stop.
2. Read `.agent-os/state.json`. If `current_stage` is not `"idle"` and `stage_status` is not `"done"`:
   - Tell the user: "There is an active task ([active_task_id]) in stage [current_stage] with status [stage_status]. Complete or cancel it before starting a new one. Use `/agent-os:status` to view the current state."
   - Stop.
3. If no task description was provided: ask the user to describe the task and wait for their response.

## Step 1: Create Task

Read `.agent-os/tasks/` to determine the next task ID (TASK-001, TASK-002, etc.).

Create `.agent-os/tasks/TASK-NNN.json` with a new task record (use the template from the state-management skill).

Update `.agent-os/state.json`:
```json
{
  "current_stage": "discovery",
  "stage_status": "not_started",
  "active_task_id": "TASK-NNN",
  ...
}
```

Tell the user: "Created TASK-NNN: [title]. Starting Stage 1: Discovery."

---

## Stage 1: Discovery

Read `workflows/01-discovery.md` and follow it completely.

Spawn a `requirements-analyst` agent with:
- The task description
- The path to `config.json`
- The path to the new task file

After the agent returns, record its output in the task file:
- `stages.discovery.status = "in_progress"` → update to `"awaiting_approval"` after agent completes
- `stages.discovery.output_summary` = brief summary of requirements captured
- `requirements` array = populate from agent output

**Approval Gate: Requirements**

Check `config.json.preferences.require_requirements_approval`.
If true (default): follow the approval-gates skill to request approval.

Present the requirements to the user and ask:

```
─────────────────────────────────────────
APPROVAL REQUIRED: Requirements
─────────────────────────────────────────
[List requirements captured]

Reply "approve" to continue to Planning, or describe any changes needed.
─────────────────────────────────────────
```

**Wait for the user's reply in this conversation.** When the user approves (any affirmative reply: "approve", "yes", "ok", "looks good", etc.) or requests changes and you apply them: record approval in the task file and state.json, then immediately continue to Stage 2 without stopping.

---

## Stage 2: Planning

Read `workflows/02-planning.md` and follow it completely.

Spawn a `planner` agent with:
- The approved task file (with requirements)
- The project config

After the agent returns:
- `stages.planning.status = "done"`
- `stages.planning.output_summary` = brief summary
- `subtasks` array = populate from planner output

Add `"planning"` to `stages_completed` in state.json.
Update `current_stage = "design"`.

---

## Stage 3: Design

Read `workflows/03-design-architecture.md` and follow it completely.

Spawn an `architect` agent with:
- The task file (requirements + subtasks)
- The project config

After the agent returns:
- `stages.design.status = "awaiting_approval"`
- `stages.design.output_summary` = architecture summary
- `decisions` array = list of created DEC-NNN IDs

**Approval Gate: Architecture**

This gate is always required (cannot be skipped).

Present the architecture to the user:

```
─────────────────────────────────────────
APPROVAL REQUIRED: Architecture
─────────────────────────────────────────
[Architecture summary from architect agent]

Architecture Decisions recorded:
[List ADRs]

Reply "approve" to continue to Development, or describe any changes needed.
─────────────────────────────────────────
```

**Wait for the user's reply in this conversation.** When the user approves (any affirmative reply: "approve", "yes", "ok", "looks good", etc.) or requests changes and you apply them: record approval, then immediately continue to Stage 4 without stopping.

---

## Stage 4: Development

Read `workflows/04-development.md` and follow it completely.

For each subtask in the plan:
1. Identify the assigned agent role (`backend-developer` or `frontend-developer`).
2. Spawn the appropriate agent with the subtask details, task file, and project config.
3. Wait for the agent to complete.
4. Record the result (files modified, completion status) in the task file.

Respect dependency order from the plan — do not run a subtask before its dependencies are done.

After all subtasks are complete:
- `stages.development.status = "done"`
- `stages.development.output_summary` = summary of what was built
- Update `files_modified` in the task file

Add `"development"` to `stages_completed`.
Update `current_stage = "qa"`.

---

## Stage 5: QA

Read `workflows/05-testing-qa.md` and follow it completely.

Spawn a `qa-engineer` agent with the full task file and project config.

After the agent returns:
- If recommendation is `APPROVE`:
  - `stages.qa.status = "done"`
  - Add `"qa"` to `stages_completed`
  - Set `task.status = "done"`, `task.completed_at = now`
  - Set `state.json: current_stage = "idle", active_task_id = null`
  - Show completion summary

- If recommendation is `REQUEST_CHANGES`:
  - Present QA findings to user
  - Ask: "Critical issues were found. Do you want to (1) fix them now, (2) defer and mark done anyway, or (3) cancel?"
  - **Wait for user response** and act accordingly.

---

## Completion

When the task is fully done, output:

```
─────────────────────────────────────────
TASK COMPLETE: [TASK-NNN]
─────────────────────────────────────────
Title: [title]
Stages: Discovery ✓ | Planning ✓ | Design ✓ | Development ✓ | QA ✓

Files modified: [N files]
Tests: [N passed]
Decisions recorded: [DEC-NNN, ...]

Run `/agent-os:status` to view the project state.
─────────────────────────────────────────
```
