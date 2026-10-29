---
name: approval-gates
description: This skill should be used whenever agent-os needs to determine whether a stage transition requires human approval, how to request it, and how to record the outcome. Use it before advancing from any stage to the next.
version: 1.0.0
---

# Approval Gates Protocol

## Gate Map

| Transition | Gate Name | Default | Configurable |
|------------|-----------|---------|--------------|
| Discovery → Planning | `requirements-approved` | **REQUIRED** | `preferences.require_requirements_approval` in config.json |
| Planning → Design | `plan-approved` | OPTIONAL | Only required if scope change detected |
| Design → Development | `architecture-approved` | **REQUIRED** | `preferences.require_architecture_approval` in config.json |
| Development → QA | `implementation-ready` | Automatic | — |
| QA → Done | `qa-passed` | Conditional | Required if critical issues found |

## Mandatory Approval Cases

Always require human approval, regardless of config, for:
- Any change to existing architecture or data models
- Destructive operations (deleting files, dropping tables, removing endpoints)
- Changes to authentication, authorization, or security
- External service integrations (APIs, payment systems, messaging)
- Database migrations on existing data
- Changes to CI/CD pipelines or deployment configuration
- Scope expansions that add new requirements beyond what the user originally described

## How to Request Approval

When a gate is reached, do the following:

1. Update `state.json`:
   ```json
   {
     "stage_status": "awaiting_approval",
     "pending_approval": {
       "gate": "requirements-approved",
       "requested_at": "<now>",
       "context": "<summary of what needs approval>",
       "next_stage": "planning"
     }
   }
   ```

2. Update the current stage in the task file to `"awaiting_approval"`.

3. Present a clear approval request to the user:

   ```
   ─────────────────────────────────────────
   APPROVAL REQUIRED: Requirements
   ─────────────────────────────────────────
   
   [Summary of what was produced in this stage]
   
   To proceed to Planning, confirm:
   - [ ] Requirements are complete and accurate
   - [ ] Scope is agreed upon
   - [ ] No missing stakeholder needs
   
   Run `/agent-os:approve` to proceed, or describe any changes needed.
   ─────────────────────────────────────────
   ```

4. **Stop and wait.** Do not proceed to the next stage. Do not run any agents for the next stage. The next stage starts only after the user explicitly approves.

## How to Record an Approval

When the user approves (via `/agent-os:approve` or explicit "yes" in conversation):

1. Update the task stage record: `"status": "done"`, `"approved_at": "<now>"`.
2. Add the stage to `stages_completed` in state.json.
3. Clear `pending_approval` in state.json.
4. Set `current_stage` to the next stage and `stage_status` to `"not_started"`.

## How to Handle Rejection or Change Requests

If the user rejects or requests changes:
1. Keep `stage_status` as `"awaiting_approval"` (or set to `"in_progress"` if rework is needed).
2. Clear `pending_approval.context` and update it with the user's feedback.
3. Re-run the stage agent with the user's feedback as additional context.
4. Return to this gate protocol after the rework.

## Optional Gates

For `plan-approved`, evaluate complexity before deciding:
- **Require approval if**: planning discovered new requirements, scope expanded significantly, or design phase will take >1 hour of Claude time.
- **Skip if**: planning is straightforward and the user originally described a simple task.

When skipping an optional gate, still update the task stage record to `"done"` and advance state.

## Blocking vs. Approval

A **block** is different from an approval gate:
- A block means the work cannot proceed because something external is missing (e.g., missing credentials, unclear requirements that need research, external dependency not available).
- An approval gate means work is done and the human must review before proceeding.

Set `stage_status: "blocked"` and fill `state.json.blocker` when blocked.
Set `stage_status: "awaiting_approval"` and fill `pending_approval` for gates.
