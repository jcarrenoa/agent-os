---
name: approval-gates
description: This skill should be used whenever agent-os needs to determine whether a stage transition requires human approval, how to request it, how to run the optional approver review before a gate, and how to record the outcome. Use it before advancing from any stage to the next.
version: 1.1.0
---

# Approval Gates Protocol

## Gate Map

| Transition | Gate Name | Default | Configurable |
|------------|-----------|---------|--------------|
| Discovery → Planning | `requirements-approved` | **REQUIRED** | `preferences.require_requirements_approval` in config.json; approver gate `requirements` |
| Planning → Design | `plan-approved` | OPTIONAL | Required if scope change detected, or while the approver reviews `planning` in `advisory` mode |
| Design → Development | `architecture-approved` | **REQUIRED** | Always human. The approver never reviews or approves it, in any mode |
| Development → QA | `implementation-ready` | Automatic | — |
| QA → Done | `qa-passed` | Conditional | Required if critical issues found |

Gate ↔ stage ↔ producing agent:

| Approver gate | Stage | Gate name | Producing agent | Workflow |
|---|---|---|---|---|
| `requirements` | `discovery` | `requirements-approved` | `requirements-analyst` | `01-discovery.md` |
| `planning` | `planning` | `plan-approved` | `planner` | `02-planning.md` |

## Approver Review

Runs only when `config.approver.enabled = true` and the stage's approver gate is in `config.approver.gates` (default: both). If the `approver` key is missing or `enabled = false`, skip this section entirely: the pipeline behaves exactly as without it.

Run it after the producing agent's output has been written to the task file, and before presenting the gate:

1. **Resolve extensions**: follow the `skill-resolution` skill for the stage under review, agent `approver`.
2. **Spawn the `approver` agent** with a clean context. Its prompt contains only (Step 5 of `skill-resolution`):
   - `Review target: requirements` or `Review target: planning`
   - the task file path, `.agent-os/config.json`, and the absolute path of the stage's workflow (`PLUGIN_ROOT/workflows/...`)
   - the `## Extensions` block

   Never put the conversation, the user's answers, the producing agent's report, or any requirement or subtask text in its prompt.
3. Verify its `EXTENSIONS APPLIED` section (Step 6 of `skill-resolution`).
4. **Record the review** in the task file: `stages.<stage>.review = { verdict, reviewed_at, findings, revisions }`, with `findings` parsed from its report (`id`, `severity`, `target`, `finding`, `proposed_change`) and `revisions` kept from the previous review of this stage (0 the first time). Set `state.json.pending_approval.approver_verdict` when the gate is requested.
5. **Act by mode** (`config.approver.mode`, default `advisory`):
   - **advisory**: always request a human gate (How to Request Approval), including `plan-approved` and including when `preferences.require_requirements_approval = false`. The human decides; the verdict is only shown.
   - **autonomous**:
     - `APPROVE` → record the approval with `approved_by = "approver"` (How to Record an Approval) and tell the user in one block that the approver approved the stage, with any `minor` findings. Do not ask. Then continue as after a human approval.
     - `REQUEST_CHANGES` and `revisions < config.approver.max_revisions` → increment `revisions`, re-run the producing agent as a revision (Revision Runs), then repeat this section from step 1.
     - `REQUEST_CHANGES` otherwise → stop and request a human gate with the verdict and findings. Nothing advances without the human.

Autonomous mode never skips: the `architecture-approved` gate, the planner's Scope Change Protocol (it pauses for the user before any review), or a `blocked` stage.

## Revision Runs

When the producing agent must rework its output (approver findings in autonomous mode, or a human change request):

1. Re-run the same agent (`requirements-analyst` or `planner`) with the same inputs as its first run (Step 5 of `skill-resolution`), plus one line: `Revision <n>: address every finding in stages.<stage>.review.findings and every entry in stages.<stage>.change_requests of the task file. The current version is in the task file.`
2. The findings and change requests are read from the task file; never paste them, or the previous output, into the prompt.
3. Write the new output to the task file, replacing the previous version of that stage's fields.

## Presenting the Approver Verdict

Whenever a gate is shown to a human for a stage with a `review`, add this block after the stage summary (labels in `output_language`):

```markdown
### 🛡️ Approver — [APPROVE | REQUEST_CHANGES]
| ID | Severidad | Objetivo | Hallazgo | Cambio propuesto |
|---|---|---|---|---|
| F-01 | blocker | REQ-002 | ... | ... |

Revisiones: [n] · Modo: [advisory | autonomous]
```

With no findings, write `Sin hallazgos.` instead of the table. The human can approve even with `REQUEST_CHANGES`; the approver never blocks a human decision.

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
       "next_stage": "planning",
       "approver_verdict": null
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
   
   Run `/agent-os:approve` to proceed, `/agent-os:review` to see the details, or describe any changes needed.
   ─────────────────────────────────────────
   ```

   If the stage has an approver `review`, add the verdict block from Presenting the Approver Verdict.

4. **Stop and wait.** Do not proceed to the next stage. Do not run any agents for the next stage. The next stage starts only after the user explicitly approves.

## How to Record an Approval

This is the only way an approval is written. `/agent-os:approve`, `/agent-os:review`, an explicit "yes" in the conversation, and an autonomous approver `APPROVE` all use it:

1. Update the task stage record: `"status": "done"`, `"approved_at": "<now>"`, `"approved_by": "human"` (or `"approver"` for an autonomous approval; never `"approver"` for `design`).
2. Add the stage to `stages_completed` in state.json.
3. Clear `pending_approval` in state.json.
4. Set `current_stage` to the next stage and `stage_status` to `"not_started"`.
5. Update `last_updated` in state.json.

## How to Handle Rejection or Change Requests

If the user rejects or requests changes (in the conversation, or with "Pedir cambios" in `/agent-os:review`):
1. Append `{ requested_at, text }` to `stages.<stage>.change_requests` in the task file, with the user's text verbatim, and increment `stages.<stage>.review.revisions` if a review exists.
2. Set `stage_status` to `"in_progress"` and update `pending_approval.context` with a one-line summary of the request.
3. Re-run the stage agent as a revision (Revision Runs). For `design`, re-run the `architect` with the same inputs plus the same revision line pointing to `stages.design.change_requests`.
4. If the approver reviews this stage, run the Approver Review again.
5. Return to this gate protocol after the rework.

## Optional Gates

For `plan-approved`, if the approver reviews `planning`, the gate is decided by the Approver Review (advisory: human gate; autonomous: as described there). Otherwise, evaluate complexity before deciding:
- **Require approval if**: planning discovered new requirements, scope expanded significantly, or design phase will take >1 hour of Claude time.
- **Skip if**: planning is straightforward and the user originally described a simple task.

When skipping an optional gate, still update the task stage record to `"done"` and advance state.

## Blocking vs. Approval

A **block** is different from an approval gate:
- A block means the work cannot proceed because something external is missing (e.g., missing credentials, unclear requirements that need research, external dependency not available).
- An approval gate means work is done and the human must review before proceeding.

Set `stage_status: "blocked"` and fill `state.json.blocker` when blocked.
Set `stage_status: "awaiting_approval"` and fill `pending_approval` for gates.
