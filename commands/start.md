---
description: Start a new task and run it through the complete 5-stage pipeline (Discovery → Planning → Design → Development → QA). Pauses at approval gates and waits for your confirmation before advancing.
argument-hint: "<task description | issue key or URL>"
allowed-tools: [Bash, Read, Write, Glob, Grep, AskUserQuestion]
---

# Agent OS — Full Pipeline

You are running a task through the complete agent-os 5-stage pipeline.

Task description: $ARGUMENTS

## Language

After confirming `.agent-os/config.json` exists, read `preferences.output_language` and write ALL pipeline messages, gate prompts, and status updates in that language (`"en"` = English, `"es"` = Spanish).

## Pre-Check

1. Verify `.agent-os/config.json` exists. If not: tell the user to run `/agent-os:setup` first and stop.
2. Read `.agent-os/state.json`. If `current_stage` is not `"idle"` and `stage_status` is not `"done"`:
   - Tell the user: "There is an active task ([active_task_id]) in stage [current_stage] with status [stage_status]. Complete or cancel it before starting a new one. Use `/agent-os:status` to view the current state."
   - Stop.
3. If no task description was provided: when `config.tracker.mode` is `pull` or `both`, run **Pick** of the `tracker-sync` skill; otherwise ask the user to describe the task and wait for their response.

## Tracker

Only when `config.tracker` exists. Every tracker operation follows the `tracker-sync` skill, runs here in the main session (never in a subagent), and never blocks the pipeline: a failure is recorded in `task.external.sync_errors` and the stage continues.

- **Issue as input**: if the argument is an issue key or URL (`tracker-sync` → Pull) and the mode allows pull, read the issue in Step 1: the task's `title` and `description` (verbatim) come from it and `task.external` links it. In `create` mode, tell the user that pull is not enabled and treat the argument as a description.
- **Route events** (fire them at these points): `start` when Stage 1 starts, `development` when Stage 4 starts (and per created sub-task when its developer run starts), `qa` when Stage 5 starts, `done_pipeline` after the post-QA commands of an approved QA, `blocked` whenever a stage becomes `blocked`.
- **Create** (`create` or `both`, task not pulled): `create_parent` after the requirements are approved, `create_subtasks` after the plan is approved. Both are gated (in unattended mode, queued for the end).
- **Comments**: Comment Commits after the post-QA commands.
- Each stage summary lists that stage's tracker writes in one line; the completion report has the Tracker table.

## Unattended Mode

When `config.approver.enabled` and `config.approver.mode = "unattended"`, tell the user once at the start: "Modo unattended: el approver decide las preguntas, los gates [list config.approver.gates] y los reintentos de QA. Solo te preguntaré por los commands post-QA marcados [gate]." Then run every stage without stopping, except when a stage becomes `blocked` (Exhausted Revisions in the `approval-gates` skill): show the blocker and the Approver Decisions Report, and stop.

## Step 1: Create Task

Read `.agent-os/tasks/` to determine the next task ID (TASK-001, TASK-002, etc.).

Create `.agent-os/tasks/TASK-NNN.json` with a new task record (use the template from the state-management skill), then record its git baseline (Recording the git baseline in the same skill). If the input is a tracker issue, fill `title`, `description` and `external` from it (`tracker-sync` → Pull); the requirements-analyst then receives that description verbatim as the task description.

Update `.agent-os/state.json`:
```json
{
  "current_stage": "discovery",
  "stage_status": "not_started",
  "active_task_id": "TASK-NNN",
  ...
}
```

Tell the user: "Created TASK-NNN: [title]. Starting Stage 1: Discovery."

---

## Stage 1: Discovery

Read `${CLAUDE_PLUGIN_ROOT}/workflows/01-discovery.md` and follow it completely. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

Follow the `skill-resolution` skill for stage `discovery`, agent `requirements-analyst`. Show any warnings.

Tracker: fire the `start` route event.

Spawn a `requirements-analyst` agent with:
- The task description (verbatim)
- The path to `config.json`
- The path to the new task file
- The extension resolution block under the heading `## Extensions`

Build the agent prompt with Step 5 of the `skill-resolution` skill (file paths and references only, never pasted content). After the agent returns, verify its `EXTENSIONS APPLIED` section with Step 6 of that skill.

If the agent returned a `QUESTIONS` block: follow the **Clarifying Questions Protocol** of the workflow (AskUserQuestion, or the approver in unattended mode), record the answers in `open_questions`, and re-run the agent with the `Answers:` line. Never answer the questions yourself or let the agent assume them.

When it returns the requirements, record its output in the task file as described in the workflow's State Updates (step 2): `requirements_format`, `requirements`, `user_stories` (with user stories), `assumptions`, `out_of_scope`, and `stages.discovery.output_summary`.

**Approver Review** (only when `config.approver.enabled` and `"requirements"` is in `config.approver.gates`): follow **Approver Review** in the `approval-gates` skill. In autonomous or unattended mode, an `APPROVE` records the approval with `approved_by = "approver"` and continues to Stage 2 without asking; a `REQUEST_CHANGES` re-runs the requirements-analyst up to `max_revisions` times, then stops at the human gate below (autonomous) or follows Exhausted Revisions (unattended).

**Approval Gate: Requirements**

Request it when the approver did not approve the stage and: `config.json.preferences.require_requirements_approval` is true (default), or the approver reviewed this stage in advisory mode, or autonomous mode stopped at a human gate, or the stage has a human change request. Follow the approval-gates skill.

Present the requirements (or user stories) to the user and ask:

```
─────────────────────────────────────────
APPROVAL REQUIRED: Requirements
─────────────────────────────────────────
[List requirements or user stories captured]

[Approver verdict block, if the approver reviewed this stage]

Reply "approve" to continue to Planning, or describe any changes needed.
`/agent-os:review` shows the details and the history.
─────────────────────────────────────────
```

**Wait for the user's reply in this conversation.** When the user approves (any affirmative reply: "approve", "yes", "ok", "looks good", etc.): record the approval with `approved_by = "human"` (How to Record an Approval in the approval-gates skill), then immediately continue to Stage 2 without stopping. When the user requests changes: follow How to Handle Rejection or Change Requests, then present this gate again.

Tracker: once the requirements are approved (by the human or the approver), run `create_parent` (`tracker-sync` → Create) when the mode allows it and the task has no issue yet.

---

## Stage 2: Planning

Read `${CLAUDE_PLUGIN_ROOT}/workflows/02-planning.md` and follow it completely. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

Follow the `skill-resolution` skill for stage `planning`, agent `planner`. Show any warnings.

Spawn a `planner` agent with:
- The approved task file path (with requirements)
- The project config path
- The extension resolution block under the heading `## Extensions`

Build the agent prompt with Step 5 of the `skill-resolution` skill (file paths and references only, never pasted content). After the agent returns, verify its `EXTENSIONS APPLIED` section with Step 6 of that skill.

After the agent returns, record `subtasks` and `plan` in the task file as described in the workflow's State Updates (step 2), plus `stages.planning.output_summary`. If the planner flagged a scope change, follow the workflow's Scope Change Protocol first (in unattended mode the approver decides it: Scope Change in the `approval-gates` skill).

**Approver Review** (only when `config.approver.enabled` and `"planning"` is in `config.approver.gates`): follow **Approver Review** in the `approval-gates` skill.
- **advisory**, **autonomous** stopped at a human gate, or a stage with a human change request: request the `plan-approved` gate with the plan summary (subtasks per role, complexity, risks) and the approver verdict block. Wait for the user's reply as in the requirements gate; on approval, record it with `approved_by = "human"`.
- **autonomous** or **unattended** `APPROVE` (or unattended Exhausted Revisions with only major/minor left): record the approval with `approved_by = "approver"` and continue without asking.

Without approver review, or once approved:
- `stages.planning.status = "done"`
- Add `"planning"` to `stages_completed` in state.json.
- Update `current_stage = "design"`.

Tracker: run `create_subtasks` (`tracker-sync` → Create) when the task has an issue created by agent-os.

---

## Stage 3: Design

Read `${CLAUDE_PLUGIN_ROOT}/workflows/03-design-architecture.md` and follow it completely. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

Follow the `skill-resolution` skill for stage `design`, agent `architect`, using its **Architect: Multiple Roles** rules: only the roles active in `config.focus` are resolved, each with its own mode, and the Extensions block carries a `### Shared` section plus one `### Role: <role>` section per active role. Never combine the roles into a single mode. Show any warnings.

Spawn an `architect` agent with:
- The task file path (requirements + subtasks)
- The project config path
- The extension resolution block under the heading `## Extensions`

Build the agent prompt with Step 5 of the `skill-resolution` skill (file paths and references only, never pasted content). After the agent returns, verify its `EXTENSIONS APPLIED` section with Step 6 of that skill.

After the agent returns:
- `stages.design.status = "awaiting_approval"`
- `stages.design.output_summary` = architecture summary
- `decisions` array = list of created DEC-NNN IDs

**Unattended architecture review** (only when `config.approver.mode = "unattended"`, `enabled`, and `"design"` is in `config.approver.gates`): follow **Approver Review** in the `approval-gates` skill with target `design` (producing agent: `architect`). On approval, continue to Stage 4 without asking. A stage with a human change request still goes to the gate below.

**Approval Gate: Architecture**

This gate is always required (cannot be skipped). It is decided by a human in every mode except the unattended review above.

Present the architecture to the user:

```
─────────────────────────────────────────
APPROVAL REQUIRED: Architecture
─────────────────────────────────────────
[Architecture summary from architect agent]

Architecture Decisions recorded:
[List ADRs]

Reply "approve" to continue to Development, or describe any changes needed.
─────────────────────────────────────────
```

**Wait for the user's reply in this conversation.** When the user approves (any affirmative reply: "approve", "yes", "ok", "looks good", etc.): record the approval with `approved_by = "human"`, then immediately continue to Stage 4 without stopping. When the user requests changes: follow How to Handle Rejection or Change Requests in the approval-gates skill, then present this gate again.

---

## Stage 4: Development

Read `${CLAUDE_PLUGIN_ROOT}/workflows/04-development.md` and follow it completely. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

Follow the `skill-resolution` skill for stage `development` twice: (role `backend`, agent `backend-developer`) and (role `frontend`, agent `frontend-developer`). For each, check for a scaffold skill and run it first if `project_type = "new"`. Show any warnings.

Read `config.focus` and `config.workspace_type`:
- `"backend"`: skip all subtasks assigned to `frontend-developer`.
- `"frontend"`: skip all subtasks assigned to `backend-developer`.
- `"fullstack"` (default): run all subtasks.

For `monorepo` or `multi-repo`: pass `working_directory: [config.workspace_paths.backend]` to backend agents and `working_directory: [config.workspace_paths.frontend]` to frontend agents.

Run **one agent per role** with all of that role's subtasks (applying focus filtering above), in dependency order, following the Execution Rule in the development workflow: roles run in parallel when neither depends on the other; cross-role dependencies split a role's list into consecutive batches.

Tracker: fire the `development` route event when the stage starts, and for the created sub-tasks of each run when that run starts.

Each agent run receives, per Step 5 of the `skill-resolution` skill (references only, never pasted content):
- The subtask IDs of that run, in order, each with its title
- The task file path, the project config path, and the paths of the relevant ADR files
- The extension resolution block for that role under the heading `## Extensions`
- `working_directory: [path]` if workspace_type is `monorepo` or `multi-repo`

After each run, record the result **per subtask** (status, files modified) in the task file and verify the `EXTENSIONS APPLIED` section (Step 6 of the `skill-resolution` skill).

After all subtasks are complete:
- `stages.development.status = "done"`
- `stages.development.output_summary` = summary of what was built
- Update `files_modified` in the task file

Add `"development"` to `stages_completed`.
Update `current_stage = "qa"`.

---

## Stage 5: QA

Read `${CLAUDE_PLUGIN_ROOT}/workflows/05-testing-qa.md` and follow it completely. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

Follow the `skill-resolution` skill for stage `qa`, agent `qa-engineer`. Show any warnings.

Tracker: fire the `qa` route event.

Spawn a `qa-engineer` agent (and any extra Agents from the resolution result in parallel) with:
- The task file path and project config path
- The extension resolution block under the heading `## Extensions`

Build the agent prompt with Step 5 of the `skill-resolution` skill (file paths and references only, never pasted content). After the agent returns, verify its `EXTENSIONS APPLIED` section with Step 6 of that skill.

After the agent returns:
- If recommendation is `APPROVE`:
  - `stages.qa.status = "done"`
  - Add `"qa"` to `stages_completed`
  - Set `task.status = "done"`, `task.completed_at = now`
  - In unattended mode, show the **Approver Decisions Report** (`approval-gates` skill) first, so the user sees what was decided before any command question.
  - Run any post-QA Commands from extensions: every command tagged `[gate]` (the default) requires an approval gate before it runs, in every approver mode, unattended included; only commands tagged `[auto]` (configured with `auto: true`) run without one
  - Tracker (`tracker-sync`): in unattended mode, the Pending Writes gate; then Comment Commits (gated); then the `done_pipeline` route event
  - Set `state.json: current_stage = "idle", active_task_id = null`
  - Show completion summary

- If recommendation is `REQUEST_CHANGES`:
  - Unattended mode: follow **QA Retries** in the `approval-gates` skill (re-run development with the issues up to `max_revisions`, then Exhausted Revisions). If it ends blocked, show the Approver Decisions Report and the blocker, and stop.
  - Otherwise: present QA findings to user
  - Ask: "Critical issues were found. Do you want to (1) fix them now, (2) defer and mark done anyway, or (3) cancel?"
  - **Wait for user response** and act accordingly.

---

## Completion

When the task is fully done, output:

```
─────────────────────────────────────────
TASK COMPLETE: [TASK-NNN]
─────────────────────────────────────────
Title: [title]
Stages: Discovery ✓ | Planning ✓ | Design ✓ | Development ✓ | QA ✓

Files modified: [N files]
Tests: [N passed]
Decisions recorded: [DEC-NNN, ...]
Tracker: [KEY → status · N errores (/agent-os:tracker sync) · omit without tracker]

Run `/agent-os:status` to view the project state.
─────────────────────────────────────────
```
