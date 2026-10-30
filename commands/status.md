---
description: Show the current agent-os pipeline state, active task progress, and pending approvals.
allowed-tools: [Read, Glob]
---

# Agent OS Status

You are displaying the current agent-os state for this project.

## Step 1: Check Initialization

Check if `.agent-os/config.json` exists. If not:
```
Agent OS is not initialized in this project.
Run `/agent-os:setup` to get started.
```
Stop.

## Step 2: Read State

Read:
- `.agent-os/config.json`
- `.agent-os/state.json`
- `.agent-os/tasks/TASK-NNN.json` (the active task, if any)

List all files in `.agent-os/tasks/` and `.agent-os/decisions/`.

## Step 3: Display Status

Output a clear status report:

```
─────────────────────────────────────────
Agent OS — Status
─────────────────────────────────────────
Project:  [project_name]
Stack:    [frontend] + [backend]
Approver: [off | advisory | autonomous (max_revisions N)] — gates: [requirements, planning | —]
Requirements format: [REQ | user stories]

Pipeline State
  Stage:   [current_stage]
  Status:  [stage_status]
  Task:    [active_task_id or "none"]

[If stage_status = "awaiting_approval"]
  ⚠  PENDING APPROVAL: [gate name]
     [context from pending_approval]
     Approver: [APPROVE | REQUEST_CHANGES (N blocker, N major, N minor) — omit when approver_verdict is null]
     Run `/agent-os:review` to review it, or `/agent-os:approve` to proceed.

[If stage_status = "blocked"]
  ✗  BLOCKED: [blocker description]

[If active task exists]
─────────────────────────────────────────
Active Task: [TASK-NNN] — [title]

  Discovery:   [✓ done | ⏳ in_progress | ⟳ awaiting_approval | ○ not_started] [approved by: human | approver] [approver: APPROVE | REQUEST_CHANGES, N revisions]
  Planning:    [✓ | ⏳ | ⟳ | ○] [approved by ...] [approver: ...]
  Design:      [✓ | ⏳ | ⟳ | ○] [approved by: human]
  Development: [✓ | ⏳ | ⟳ | ○]
  QA:          [✓ | ⏳ | ⟳ | ○]

  (Omit the "approved by" and "approver" parts when the stage has no approval or review.)

  Requirements: [N captured] [+ N user stories, N acceptance criteria — user-stories only]
  Subtasks:     [N total, N done]
  Decisions:    [N recorded]
  Files:        [N modified]

─────────────────────────────────────────
All Tasks

  [TASK-001]: [title] — [done|active|blocked|cancelled]
  [TASK-002]: [title] — [...]
  ...

Decisions: [N total]
  [DEC-001]: [title] — [accepted|proposed]
  ...

─────────────────────────────────────────
Available Commands

  /agent-os:start "[description]"   ← New task (full pipeline)
  /agent-os:review                  ← Approval console (gate, approver verdict, history)
  /agent-os:approve                 ← Approve pending gate
  /agent-os:rollback [stage]        ← Return to a previous stage
  /agent-os:discover "[description]" ← Run Stage 1 only
  /agent-os:plan                    ← Run Stage 2 only
  /agent-os:design                  ← Run Stage 3 only
  /agent-os:develop                 ← Run Stage 4 only
  /agent-os:qa                      ← Run Stage 5 only
─────────────────────────────────────────
```
