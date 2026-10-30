---
name: approval-gates
description: This skill should be used whenever agent-os needs to determine whether a stage transition requires human approval, how to request it, how to run the optional approver review before a gate, and how to record the outcome. Use it before advancing from any stage to the next.
version: 1.3.0
---

# Approval Gates Protocol

## Gate Map

| Transition | Gate Name | Default | Configurable |
|------------|-----------|---------|--------------|
| Discovery → Planning | `requirements-approved` | **REQUIRED** | `preferences.require_requirements_approval` in config.json; approver gate `requirements` |
| Planning → Design | `plan-approved` | OPTIONAL | Required if scope change detected, or while the approver reviews `planning` in `advisory` mode |
| Design → Development | `architecture-approved` | **REQUIRED** | Human, except in `unattended` mode with `design` in `config.approver.gates` |
| Development → QA | `implementation-ready` | Automatic | — |
| QA → Done | `qa-passed` | Conditional | Required if critical issues found (in `unattended`: QA Retries) |

Gate ↔ stage ↔ producing agent:

| Approver gate | Stage | Gate name | Producing agent | Workflow |
|---|---|---|---|---|
| `requirements` | `discovery` | `requirements-approved` | `requirements-analyst` | `01-discovery.md` |
| `planning` | `planning` | `plan-approved` | `planner` | `02-planning.md` |
| `design` (unattended only) | `design` | `architecture-approved` | `architect` | `03-design-architecture.md` |

## Approver Modes

| | advisory | autonomous | unattended |
|---|---|---|---|
| Gates it can review | requirements, planning | requirements, planning | requirements, planning, design |
| `APPROVE` | human gate | approves | approves |
| `REQUEST_CHANGES` | human gate | revision runs up to `max_revisions`, then human gate | revision runs up to `max_revisions`, then Exhausted Revisions |
| Clarifying questions, scope change | human | human | approver |
| QA `REQUEST_CHANGES` | human | human | QA Retries |
| Post-QA commands `[gate]` | human | human | human |

`unattended` exists so a task can run without the user. Nothing in it ever runs a post-QA command tagged `[gate]` without the human: those may have external effects (push, deploy, publish, posting comments).

## Approver Review

Runs only when `config.approver.enabled = true` and the stage's approver gate is in `config.approver.gates`. If the `approver` key is missing or `enabled = false`, skip this section entirely: the pipeline behaves exactly as without it.

Run it after the producing agent's output has been written to the task file, and before presenting the gate:

1. **Resolve extensions**: follow the `skill-resolution` skill for the stage under review, agent `approver`.
2. **Spawn the `approver` agent** with a clean context. Its prompt contains only (Step 5 of `skill-resolution`):
   - `Review target: requirements`, `planning` or `design`
   - the task file path, `.agent-os/config.json`, the absolute path of the stage's workflow (`PLUGIN_ROOT/workflows/...`), and for `design` the ADR paths of `task.decisions`
   - the `## Extensions` block

   Never put the conversation, the user's answers, the producing agent's report, or any requirement, subtask or design text in its prompt.
3. Verify its `EXTENSIONS APPLIED` section (Step 6 of `skill-resolution`).
4. **Record the review** in the task file:
   - If `stages.<stage>.review` exists, append it to `stages.<stage>.review_history`.
   - `round` = previous round + 1 (1 the first time).
   - Write `stages.<stage>.review = { round, verdict, reviewed_at, findings }`. Each finding keeps `severity`, `target`, `finding` and `proposed_change` from the report, and its `id` becomes `R<round>-<id>` (`F-01` in round 2 → `R2-F-01`).
   - Set `state.json.pending_approval.approver_verdict` when the gate is requested.
5. **Act by mode** (`config.approver.mode`, default `advisory`). If `stages.<stage>.change_requests` is not empty, a human has already asked for changes at this gate: act as `advisory` for this stage, whatever the mode.
   - **advisory**: always request a human gate (How to Request Approval), including `plan-approved` and including when `preferences.require_requirements_approval = false`. The human decides; the verdict is only shown.
   - **autonomous** and **unattended**:
     - `APPROVE` → record the approval with `approved_by = "approver"` (How to Record an Approval), log it (Approver Decisions, kind `gate`), and tell the user in one block that the approver approved the stage, with any `minor` findings. Do not ask. Then continue as after a human approval.
     - `REQUEST_CHANGES` and `stages.<stage>.auto_revisions < config.approver.max_revisions` → increment `auto_revisions`, log it (kind `revision`), re-run the producing agent as a revision (Revision Runs), then repeat this section from step 1.
     - `REQUEST_CHANGES` otherwise → **autonomous**: stop and request a human gate with the verdict and findings. **unattended**: Exhausted Revisions.

Autonomous mode never skips the `architecture-approved` gate, the planner's Scope Change Protocol, or a `blocked` stage. Unattended mode never skips a `blocked` stage or a `[gate]` post-QA command.

### Exhausted Revisions (unattended only)

When `auto_revisions` has reached `max_revisions` and the latest review (or QA report) still asks for changes:
- **Any `blocker`** (approver) or **`critical`** (QA) left → set the stage `blocked`: `state.json.stage_status = "blocked"`, `state.json.blocker = { description: "<stage>: unresolved <IDs>", created_at, created_by: "approver" }`, `task.status = "blocked"`. Log it (kind `blocked`). Stop. Nothing advances with a blocker.
- **Only `major` / `minor`** → write their IDs to `stages.<stage>.unresolved_findings`, log it (kind `continue-unresolved`), record the approval with `approved_by = "approver"` and continue.

### Approver Decisions

Every decision the approver takes instead of a human (autonomous and unattended) is appended to `task.approver_decisions`: `{ at, stage, kind, decision, reason, ref }`, where `ref` says where the details are (`stages.planning.review (round 2)`, `open_questions Q-3`, `stages.qa.issues`). One entry per decision: each answered question, each revision run and each gate is its own entry, never one entry for several. Unattended mode ends with the Approver Decisions Report.

## Clarifying Questions (unattended only)

When the requirements-analyst returns a `QUESTIONS` block, the Clarifying Questions Protocol of `01-discovery.md` records them in `task.open_questions`. In unattended mode they are answered by the approver instead of the user:

1. Follow the `skill-resolution` skill for stage `discovery`, agent `approver`, and spawn it with `Review target: questions`, the task file path, `.agent-os/config.json` and the `## Extensions` block.
2. For each answer in its report, set `resolution` (verbatim), `resolved_by = "approver"` and `resolved_at` on that question, and log it (kind `question`, `ref: "open_questions Q-N"`).
3. A question the approver marks `NEEDS_HUMAN` stays unresolved and the stage is `blocked` (Exhausted Revisions, blocker branch).

## Scope Change (unattended only)

When the planner flags a scope change, the approver decides instead of the user: spawn it with `Review target: scope-change` (stage `planning`) and the same inputs as a planning review. Record its `DECISION` (`ORIGINAL_SCOPE` or `EXPAND`) and log it (kind `scope-change`). `ORIGINAL_SCOPE` → the planner's extra work is excluded and planning continues. `EXPAND` → the stage is `blocked`: expanding the scope means re-running discovery with a new description, which is the user's call.

## QA Retries (unattended only)

When QA returns `REQUEST_CHANGES` (critical issues):
1. Write the report's issues to `stages.qa.issues` (`id`, `severity`, `location`, `description`).
2. If `stages.qa.auto_revisions < config.approver.max_revisions`: increment it, log it (kind `qa-retry`), set `stages.development.status = "in_progress"` and re-run development (`04-development.md`) for the roles whose files or subtasks are named by a `critical` or `major` issue, with the line `QA revision <n>: fix every critical and major issue in stages.qa.issues of the task file.` Then run QA again.
3. Otherwise: Exhausted Revisions (a `critical` left blocks; only `major`/`minor` → close the task with them in `stages.qa.unresolved_findings`).

## Approver Decisions Report

At the end of an unattended run (task done or blocked), and in `/agent-os:review`, show `task.approver_decisions` as this table (labels in `output_language`). It is always a table with one row per entry, in order: never prose, never a summary such as "answered 5 questions, see open_questions". The user must be able to review every decision without opening the task file, so each `Decisión` cell carries the content, read from the place `ref` points to:
- `question`: `Q-N: <question> → <resolution>` (from `open_questions`).
- `gate`, `revision`: the verdict and the finding IDs of that round with their severity (`REQUEST_CHANGES: R1-F-01 blocker, R1-F-02 major`).
- `scope-change`: `ORIGINAL_SCOPE` or `EXPAND` and the scope the planner proposed.
- `qa-retry`: the `critical` and `major` issue IDs of `stages.qa.issues`.
- `continue-unresolved`, `blocked`: every unresolved ID with its severity.

```markdown
### 🤖 Decisiones del approver
| Fecha | Etapa | Tipo | Decisión | Motivo | Detalle en |
|---|---|---|---|---|---|
| 2026-10-29 10:12 | discovery | question | Q-2: ¿Borrado físico o lógico de contactos? → Lógico (DeletedAt) | ... | `open_questions Q-2` |
| 2026-10-29 10:31 | planning | revision | REQUEST_CHANGES: R1-F-01 major, R1-F-02 minor | ... | `stages.planning.review (round 1)` |
| 2026-10-29 10:40 | design | gate | APPROVE | ... | `stages.design.review (round 1)` |

Sin resolver:
| ID | Etapa | Severidad | Hallazgo |
|---|---|---|---|
| R2-F-03 | planning | major | [one-line finding] |

Commands post-QA: [name — ejecutado automáticamente ([auto]) | aprobado por ti | rechazado por ti]
```

The `Sin resolver` table has one row per ID in every stage's `unresolved_findings`, with the finding (or QA issue) text; write `Sin resolver: ninguno` when there are none.

## Revision Runs

A revision run is triggered by exactly one source, and its prompt names it:

1. Re-run the same agent (`requirements-analyst`, `planner` or `architect`) with the same inputs as its first run (Step 5 of `skill-resolution`), plus one line:
   - approver: `Revision <n>: address every finding of stages.<stage>.review (round <r>) in the task file.`
   - human: `Revision <n>: address change request CR-<k> in stages.<stage>.change_requests of the task file, including its cited_findings.`

   `<n>` counts all revision runs of the stage (auto and human).
2. The findings and change requests are read from the task file; never paste them, or the previous output, into the prompt.
3. Write the new output to the task file, replacing the previous version of that stage's fields (for `design`, the architect updates its ADR files).

## Presenting the Approver Verdict

Whenever a gate is shown to a human for a stage with a `review`, add this block after the stage summary (labels in `output_language`):

```markdown
### 🛡️ Approver — [APPROVE | REQUEST_CHANGES] (ronda [round])
| ID | Severidad | Objetivo | Hallazgo | Cambio propuesto |
|---|---|---|---|---|
| R1-F-01 | blocker | REQ-002 | ... | ... |

Revisiones automáticas: [auto_revisions] · Cambios pedidos por ti: [number of change_requests] · Modo: [advisory | autonomous | unattended]
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

The only exception is `unattended` mode, which the user chose so that nothing asks: there the approver decides these at the gate where they appear, and the approver agent reports each one as a finding of at least `major` severity when it has no mitigation. Scope expansions still block (Scope Change). Post-QA commands tagged `[gate]` are never an exception.

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

This is the only way an approval is written. `/agent-os:approve`, `/agent-os:review`, an explicit "yes" in the conversation, and an approver `APPROVE` in autonomous or unattended mode all use it:

1. Update the task stage record: `"status": "done"`, `"approved_at": "<now>"`, `"approved_by": "human"` (or `"approver"` for an approver approval; `"approver"` for `design` only in unattended mode).
   - For `design`, whoever approved: for each ID in `task.decisions`, set `.agent-os/decisions/DEC-NNN.json` to `"status": "accepted"` and `"accepted_at": "<now>"`; then, for each ID in its `supersedes`, set that earlier ADR to `"status": "superseded"` and `"superseded_by"` to the new ID. An approved design never leaves its ADRs `proposed`.
2. Add the stage to `stages_completed` in state.json.
3. Clear `pending_approval` in state.json.
4. Set `current_stage` to the next stage and `stage_status` to `"not_started"`.
5. Update `last_updated` in state.json.
6. With `config.tracker`: after `discovery` is approved, run `create_parent`; after `planning` is done (approved or its optional gate skipped), run `create_subtasks` (`tracker-sync` skill → Create; both gated, never blocking).

## How to Handle Rejection or Change Requests

If the user rejects or requests changes (in the conversation, or with "Pedir cambios" in `/agent-os:review`), always record the request and re-run the agent. Never decide yourself that the request is already covered by the current output and skip the re-run: if it is, the agent says so in its Revision Notes.

1. Append `{ id: "CR-<k>", requested_at, text, cited_findings }` to `stages.<stage>.change_requests`, with the user's text verbatim. For every finding ID the text cites, copy that finding into `cited_findings`: a full ID (`R1-F-02`) is looked up in `review` and `review_history`; a bare one (`F-02`) refers to the latest `review` round. Human change requests never change `auto_revisions`.
2. Set `stage_status` to `"in_progress"` and update `pending_approval.context` with a one-line summary of the request.
3. Re-run the stage agent as a revision with the human line of Revision Runs. For `design`, the agent is the `architect`, resolved with the **Architect: Multiple Roles** rules of `skill-resolution`.
4. If the approver reviews this stage, run the Approver Review again. Because the stage now has a change request, its next gate is human in every mode (Approver Review, step 5).
5. Return to this gate protocol after the rework.

## Optional Gates

For `plan-approved`, if the approver reviews `planning`, the gate is decided by the Approver Review. Otherwise, evaluate complexity before deciding:
- **Require approval if**: planning discovered new requirements, scope expanded significantly, or design phase will take >1 hour of Claude time.
- **Skip if**: planning is straightforward and the user originally described a simple task.

When skipping an optional gate, still update the task stage record to `"done"` and advance state.

## Blocking vs. Approval

A **block** is different from an approval gate:
- A block means the work cannot proceed because something external is missing (e.g., missing credentials, unclear requirements that need research, external dependency not available), or, in unattended mode, a blocker is left after `max_revisions`.
- An approval gate means work is done and the human must review before proceeding.

Set `stage_status: "blocked"` and fill `state.json.blocker` when blocked. With `config.tracker`, fire the `blocked` route event (`tracker-sync` skill).
Set `stage_status: "awaiting_approval"` and fill `pending_approval` for gates.
