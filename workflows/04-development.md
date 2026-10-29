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

## Execution Rule

**One agent per role, running its subtasks in dependency order.** The development stage does not spawn one agent per subtask.

1. Group the subtasks (after focus filtering) by role: `backend-developer` and `frontend-developer`. Inside each group, order them by `task.subtasks[].depends_on`.
2. If no subtask of one role depends on a subtask of the other role, spawn both role agents in parallel, each with all of its subtasks.
3. If there are cross-role dependencies, split each role's list into consecutive batches at every cross-role dependency. Run the batches in dependency order, one agent run per batch; batches of different roles run in parallel when neither waits for the other.
4. State is still tracked **per subtask**: the agent reports each subtask as complete or blocked, and the calling command updates each subtask's status in the task file. An agent that hits a blocked subtask stops and does not start the rest of its run.

## Execution Order

Within each role, follow the dependency order defined in the plan (`task.subtasks[].depends_on`).

For subtasks with no dependencies between them, run in this preferred order:
1. Backend subtasks first (frontend depends on APIs)
2. Frontend subtasks after backend
3. Integration subtasks last

Exception: If the task has no backend (frontend-only), run frontend subtasks in whatever order the plan specifies.

## Context Passed to Each Agent

Never paste into the prompt the design, ADR content, requirements or rules from any `CLAUDE.md`. Never read test plans, checklists or expected-results files from the workspace to pass hints to agents. After each run, the calling command verifies the agent's `EXTENSIONS APPLIED` section (Step 6 of the `skill-resolution` skill).

Before spawning each developer agent, the calling command follows the `skill-resolution` skill for stage `development` with the agent's role (`backend` or `frontend`). If a scaffold skill is configured and `project_type = "new"`, it is run before the first subtask for that role.

Each developer agent run receives references, not content (Step 5 of the `skill-resolution` skill). The agent reads every file itself:
1. The IDs and titles of the subtasks in this run, in execution order (the details stay in the task file).
2. The task file path (to read requirements and architecture).
3. The config file path (for tech stack).
4. The paths of the ADR files from `task.decisions`.
5. The extension resolution block for their role under `## Extensions` (includes resolved absolute skill and doc paths, and mode).
6. In `monorepo` / `multi-repo`: `working_directory` for their role. The agent prefixes only project code paths with it; `.agent-os/` and the Extensions paths are used as-is from the workspace root.

## Parallel Execution

The two role agents (or batches) may run in parallel when:
- Neither has a subtask that depends on a subtask of the other.
- No frontend subtask calls backend APIs that don't exist yet.

When in doubt: run the backend agent first, then the frontend agent.

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
