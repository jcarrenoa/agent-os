# Workflow: Stage 3 — Design & Architecture

## Purpose

Define the complete technical architecture before a single line of implementation code is written. Record all significant decisions as ADRs. Require human approval before development begins.

## Entry Conditions

- `stages.planning.status = "done"` in the task file.
- `task.subtasks` contains at least one subtask.
- `state.json.current_stage = "design"` and `stage_status = "in_progress"`.

## Exit Conditions (all must be true before requesting approval)

- [ ] All components to be created or modified are defined with clear responsibilities.
- [ ] Data models and schema changes are specified.
- [ ] All new API endpoints have defined contracts.
- [ ] At least one ADR is recorded for significant design decisions.
- [ ] Implementation order for developers is defined.
- [ ] Risks are identified.

## The Agent's Job

Before spawning the agent, the calling command follows the `skill-resolution` skill for stage `design` and passes the result to the agent under `## Extensions`. Only the roles active in `config.focus` are resolved; each active role gets its own section and mode in the block (see the `skill-resolution` skill, Architect: Multiple Roles). Roles are never merged into one mode. The agent reads listed skills and docs before designing.

The `architect` agent:
1. Reads `config.json` for the tech stack.
2. Reads the task file for requirements and subtasks.
3. Reads `CLAUDE.md` and relevant existing codebase files.
4. Designs components, data models, and API contracts.
5. Creates ADR files in `.agent-os/decisions/` for each significant decision.
6. Returns the architecture blueprint to the command.

## ADR Creation Rules

An ADR is **required** for:
- Choosing a new library or framework.
- Deviating from existing project patterns.
- Database schema changes on existing tables.
- Choosing between two or more viable implementation approaches.
- Any decision with significant long-term consequences.

An ADR is NOT needed for:
- Obvious, conventional choices (adding a standard CRUD endpoint using the existing pattern).
- Trivial implementation details.

The architect creates ADR files directly at `.agent-os/decisions/DEC-NNN.json` using the decision schema.

## Output Format

The architect returns:

```
ARCHITECTURE OUTPUT
===================
COMPONENTS
  [ComponentName] — [path/to/file]
    Responsibility: [one sentence]
    Interface: [key methods]
    Dependencies: [what it calls]

DATA MODELS
  [ModelName] (table: [name])
    [field]: [type] [constraints]
    Migration needed: [yes|no]

API CONTRACTS
  [METHOD] [path]
  Request: [schema]
  Response: [schema]
  Auth: [requirement]

ADRs CREATED
  DEC-001: [title]
  DEC-002: [title]

IMPLEMENTATION ORDER FOR DEVELOPERS
  1. [what to build first]
  2. [what comes next]

RISKS
  [risk] → Mitigation: [approach]
===================
```

## State Updates

1. When starting: `task.stages.design.status = "in_progress"`, `started_at = now`.
2. After agent creates ADR files: update `task.decisions` with the DEC-NNN IDs.
3. When requesting approval:
   - `task.stages.design.status = "awaiting_approval"`, `completed_at = now`.
   - `task.stages.design.output_summary = "[N components, N data models, N ADRs]"`.
   - `state.json.stage_status = "awaiting_approval"`.
   - `state.json.pending_approval = { gate: "architecture-approved", next_stage: "development", ... }`.
   - Unattended approver mode with `"design"` in `config.approver.gates` (and no human change request on the stage): the approver reviews the design before the gate (Approver Review in the `approval-gates` skill, target `design`, producing agent `architect`). It writes `task.stages.design.review`, may re-run the architect, and approves with `approved_by = "approver"` or blocks the stage.
4. When approved:
   - `task.stages.design.status = "done"`, `approved_at = now`, `approved_by = "human" | "approver"` (`approver` only in unattended mode).
   - Every ADR in `task.decisions` becomes `accepted`, and the ADRs in their `supersedes` become `superseded` (How to Record an Approval in the `approval-gates` skill), for both kinds of approval.
   - Add `"design"` to `stages_completed`.
   - `state.json.current_stage = "development"`, `stage_status = "not_started"`.

## Approval Criteria

The user should approve when:
- The architecture is coherent with existing patterns.
- Data models are correct and safe.
- API contracts match the requirements.
- They understand and accept the ADRs.

The user should reject when:
- A component design is incorrect or incomplete.
- A data model change is risky.
- An ADR decision is wrong.
- The architecture introduces unnecessary complexity.
