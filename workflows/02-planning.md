# Workflow: Stage 2 — Analysis & Planning

## Purpose

Transform approved requirements into a concrete, sequenced implementation plan with subtasks, dependencies, risk analysis, and agent assignments.

## Entry Conditions

- `stages.discovery.status = "done"` in the task file.
- `task.requirements` contains at least one `must` requirement, or `task.user_stories` contains at least one `must` story.
- `state.json.current_stage = "planning"` and `stage_status = "in_progress"`.

## Exit Conditions (all must be true before marking stage done)

- [ ] Every approved requirement (`REQ-NNN`) and, with user stories, every story (`US-NNN`, all its acceptance criteria) is addressed by at least one subtask.
- [ ] Every subtask has an assigned agent role.
- [ ] Every subtask has a verifiable completion criterion.
- [ ] Dependency order is defined.
- [ ] Risks are identified.
- [ ] Complexity is assessed.

## The Agent's Job

Before spawning the agent, the calling command follows the `skill-resolution` skill for stage `planning` and passes the result to the agent under `## Extensions`. The agent reads any listed skills and docs before starting.

The `planner` agent:
1. Reads `config.json` and the task file.
2. Inspects the codebase for relevant existing patterns.
3. Breaks requirements into concrete, atomic subtasks.
4. Assigns each subtask to `backend-developer`, `frontend-developer`, or `qa-engineer`.
5. Identifies dependencies between subtasks.
6. Identifies risks and proposes mitigations.
7. Returns the plan to the command.

## Subtask Assignment Rules

- Assign to `backend-developer`: API endpoints, business logic, data models, migrations, authentication, background jobs.
- Assign to `frontend-developer`: UI components, client state, forms, routing, API integration from the client.
- Assign to `architect`: subtasks that require deciding between architectural approaches (rare — should already be resolved by Stage 3).
- `qa-engineer` is NOT assigned subtasks here; QA runs holistically in Stage 5.

## Scope Change Protocol

If the planner discovers that implementing the requirements requires work not covered by the approved requirements:
1. The planner explicitly notes the scope expansion.
2. The command **pauses** and presents it to the user:
   ```
   ─────────────────────────────────────────
   SCOPE CHANGE DETECTED
   ─────────────────────────────────────────
   The planner identified work beyond the approved scope:
   [description of extra work]
   
   Options:
   1. Expand scope (re-run Stage 1 with updated description)
   2. Proceed with original scope only (exclude extra work)
   ─────────────────────────────────────────
   ```
3. The command waits for user decision before updating state.

In unattended approver mode the approver decides instead of the user (Scope Change in the `approval-gates` skill): `ORIGINAL_SCOPE` → option 2; `EXPAND` → the stage is `blocked`, because re-running Stage 1 with a new description is the user's call.

## Output Format

The planner returns:

```
PLANNING OUTPUT
===============
Complexity: [Simple|Medium|Complex]

AFFECTED FILES (predicted)
  [path]: [why]

SUBTASKS
  SUB-01: [title]
    Role: backend-developer
    Requirements: REQ-001, REQ-002
    Depends on: none
    Criterion: [verifiable condition]

  SUB-02: [title]
    Role: frontend-developer
    Requirements: REQ-001
    Depends on: SUB-01
    Criterion: [verifiable condition]

EXECUTION ORDER
  1. SUB-01
  2. SUB-02 (after SUB-01)

RISKS
  - [risk] → Mitigation: [approach]

ARCHITECT INVOLVEMENT REQUIRED: [yes|no]
===============
```

## State Updates

1. When starting: `task.stages.planning.status = "in_progress"`, `started_at = now`.
2. When agent completes:
   - Populate `task.subtasks` from agent output (`id`, `title`, `assigned_to`, `status = "pending"`, `depends_on`, `requirements`, `criterion`).
   - Populate `task.plan`: `complexity` (lowercase), `risks[{ description, mitigation }]`, `external_dependencies`, `scope_change` (or null).
   - `task.stages.planning.completed_at = now`, `task.stages.planning.output_summary = "[N subtasks: N backend, N frontend]"`.
   - On a revision run, replace `task.subtasks` and `task.plan` with the new version.
3. Scope Change Protocol first (above), if the planner flagged one.
4. Approver review (only when `config.approver.enabled` and `"planning"` is in `config.approver.gates`): follow **Approver Review** in the `approval-gates` skill.
   - **advisory**, **autonomous** stopped at a human gate, or a stage with a human change request: `task.stages.planning.status = "awaiting_approval"`, `state.json.stage_status = "awaiting_approval"`, `state.json.pending_approval = { gate: "plan-approved", context: "...", next_stage: "design", approver_verdict: [verdict] }`. Wait for the approval (How to Record an Approval).
   - **autonomous** or **unattended** `APPROVE`, or unattended Exhausted Revisions with only major/minor left: record the approval with `approved_by = "approver"` (and `stages.planning.unresolved_findings` in the second case).
   - **unattended** Exhausted Revisions with a blocker: `stage_status = "blocked"`, `state.json.blocker` filled; stop.
5. Without approver review (or once approved):
   - `task.stages.planning.status = "done"` (with `approved_at` and `approved_by` only if a gate was approved).
   - Add `"planning"` to `stages_completed` in state.json.
   - `state.json.current_stage = "design"`, `stage_status = "not_started"`.
