---
description: Run Stage 2 (Analysis & Planning) for the active task. Breaks approved requirements into subtasks with dependencies and risk analysis. Requires Stage 1 to be approved first.
allowed-tools: [Read, Write, Glob, Grep, Bash]
---

# Agent OS — Stage 2: Planning

You are running Stage 2 (Analysis & Planning) of the agent-os pipeline.

## Pre-Check

1. Read `.agent-os/config.json` and `.agent-os/state.json`.
2. If `current_stage` is not `"planning"`:
   - If Stage 1 is not done: tell the user to complete and approve Stage 1 first (`/agent-os:discover`).
   - If current stage is later than planning: tell the user the planning stage is already done.
3. Read `.agent-os/tasks/[active_task_id].json`.
4. Verify `stages.discovery.status = "done"`. If not: stop and tell the user to complete Stage 1 first.

## Run Stage 2

Read `${CLAUDE_PLUGIN_ROOT}/workflows/02-planning.md` and follow it completely. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

Set `stages.planning.status = "in_progress"` and `state.json.stage_status = "in_progress"`.

### Resolve Extensions

Follow the `skill-resolution` skill for stage `planning`, role `—`, agent `planner`. Show any warnings to the user.

Spawn a `planner` agent. Pass it:
- The task file path (with approved requirements)
- The project config path
- The full extension resolution block under the heading `## Extensions`

Build the agent prompt with Step 5 of the `skill-resolution` skill (file paths and references only, never pasted content). After the agent returns, verify its `EXTENSIONS APPLIED` section with Step 6 of that skill.

After the agent completes:
1. Record `task.subtasks` and `task.plan` as described in the workflow's State Updates (step 2).
2. Set `stages.planning.completed_at = now` and `stages.planning.output_summary` to a brief summary of the plan.

## Scope Change Check

If the planner identified that implementation requires scope beyond the approved requirements:
- Unattended approver mode: the approver decides (Scope Change in the `approval-gates` skill). `ORIGINAL_SCOPE` continues; `EXPAND` blocks the stage: show the blocker and stop.
- Otherwise: **stop** and flag it to the user.
- Ask: "The planner identified that this task requires [new scope]. Do you want to (1) expand scope and re-run discovery, or (2) proceed with original scope only?"
- **Wait for answer** before updating state.

## Approver Review

Only when `config.approver.enabled` and `"planning"` is in `config.approver.gates`: follow **Approver Review** in the `approval-gates` skill.
- **advisory**, **autonomous** stopped at a human gate, or a stage with a human change request: set `stages.planning.status = "awaiting_approval"`, `state.json.stage_status = "awaiting_approval"` and `pending_approval = { gate: "plan-approved", ..., next_stage: "design", approver_verdict }`. Present the gate with the plan summary and the approver verdict block, then **stop and wait** for `/agent-os:approve`, `/agent-os:review`, or an explicit approval in the conversation (recorded with `approved_by = "human"`). Change requests follow How to Handle Rejection or Change Requests.
- **autonomous** or **unattended** `APPROVE` (or unattended Exhausted Revisions with only major/minor left): record the approval with `approved_by = "approver"` and continue to Complete Stage.
- **unattended** blocked: show the blocker and stop.

## Complete Stage

Without approver review, or once the plan is approved:
1. Set `stages.planning.status = "done"`.
2. Add `"planning"` to `stages_completed` in state.json.
3. Update `state.json.current_stage = "design"`, `stage_status = "not_started"`.

## Report

Show the user:
```
─────────────────────────────────────────
Stage 2 Complete: Planning
─────────────────────────────────────────
Subtasks: [N]
  Backend: [N subtasks]
  Frontend: [N subtasks]
  QA: 1 subtask

Complexity: [Simple|Medium|Complex]
Risks: [N identified]
Approver: [APPROVE (approved by approver) | approved with unresolved R<n>-F-NN | — when not reviewed]

Next: Stage 3 — Design/Architecture
  Run `/agent-os:design` or `/agent-os:start`
─────────────────────────────────────────
```
