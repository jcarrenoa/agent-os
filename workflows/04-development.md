# Workflow: Stage 4 — Development

## Purpose

Implement all approved subtasks in the correct order, following the approved architecture and the technology conventions of this project.

## Entry Conditions

- `stages.design.status = "done"` and `stages.design.approved_at` is set.
- `task.subtasks` contains at least one subtask.
- `state.json.current_stage = "development"` and `stage_status = "in_progress"`.

## Exit Conditions (all must be true before marking stage done)

- [ ] All subtasks have `status = "done"`.
- [ ] All files in `task.files_modified` have been written.
- [ ] The project's test suite passes (or failures are explicitly flagged).
- [ ] No blocking issues remain.

## Execution Order

Follow the dependency order defined in the plan (`task.subtasks[].depends_on`).

For subtasks with no dependencies between them, run in this preferred order:
1. Backend subtasks first (frontend depends on APIs)
2. Frontend subtasks after backend
3. Integration subtasks last

Exception: If the task has no backend (frontend-only), run frontend subtasks in whatever order the plan specifies.

## Context Passed to Each Agent

Before spawning each developer agent, the calling command follows the `skill-resolution` skill for stage `development` with the agent's role (`backend` or `frontend`). If a scaffold skill is configured and `project_type = "new"`, it is run before the first subtask for that role.

Each developer agent receives:
1. Their specific subtask object from `task.subtasks`.
2. The full task file path (to read requirements and architecture).
3. The config file path (for tech stack).
4. The list of ADR IDs from `task.decisions` (to read architecture decisions).
5. Implementation order guidance from Stage 3.
6. The extension resolution block for their role under `## Extensions` (includes resolved skill and doc paths, and mode).

## Parallel Execution

Two subtasks may be run in parallel when:
- One is assigned to `backend-developer` and the other to `frontend-developer`.
- Neither depends on the other.
- The frontend subtask does not call backend APIs that don't exist yet.

When in doubt: run sequentially.

## Blocker Protocol

If a developer agent cannot complete a subtask:
1. Set `task.status = "blocked"`.
2. Add to `task.blockers`: `{ description: "[what is blocking]", created_at: now }`.
3. Set `state.json.stage_status = "blocked"`, `state.json.blocker = { description: "...", created_by: "[agent]" }`.
4. Report to the user:
   ```
   ─────────────────────────────────────────
   DEVELOPMENT BLOCKED
   ─────────────────────────────────────────
   Subtask: [SUB-XX] — [title]
   Blocker: [description]
   
   Options:
   1. Resolve the blocker and run `/agent-os:develop` again
   2. Skip this subtask (marks it as deferred)
   3. Rollback to design stage (`/agent-os:rollback design`)
   ─────────────────────────────────────────
   ```
5. Wait for user decision.

## After Each Subtask

Update `task.subtasks[SUB-XX].status = "done"` after each subtask completes.
Append all new/modified files to `task.files_modified`.

## State Updates

1. When starting: `task.stages.development.status = "in_progress"`, `started_at = now`.
2. After all subtasks: 
   - `task.stages.development.status = "done"`, `completed_at = now`.
   - `task.stages.development.output_summary = "[N subtasks done. Modified N files.]"`.
   - Add `"development"` to `stages_completed`.
   - `state.json.current_stage = "qa"`, `stage_status = "not_started"`.
