# Workflow: Stage 2 — Analysis & Planning

## Purpose

Transform approved requirements into a concrete, sequenced implementation plan with subtasks, dependencies, risk analysis, and agent assignments.

## Entry Conditions

- `stages.discovery.status = "done"` in the task file.
- `task.requirements` contains at least one `must` requirement.
- `state.json.current_stage = "planning"` and `stage_status = "in_progress"`.

## Exit Conditions (all must be true before marking stage done)

- [ ] Every approved requirement is addressed by at least one subtask.
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
   - Populate `task.subtasks` from agent output.
   - `task.stages.planning.status = "done"`, `completed_at = now`.
   - `task.stages.planning.output_summary = "[N subtasks: N backend, N frontend]"`.
   - Add `"planning"` to `stages_completed` in state.json.
   - `state.json.current_stage = "design"`, `stage_status = "not_started"`.
