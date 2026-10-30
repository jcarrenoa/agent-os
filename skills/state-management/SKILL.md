---
name: state-management
description: This skill should be used whenever agent-os needs to read or write state files in .agent-os/. Use it before reading config.json, state.json, or any task/decision file, and before writing any of them. Never modify state files without following this protocol.
version: 1.1.0
---

# State Management Protocol

All agent-os state lives in `.agent-os/` at the root of the target project.

## Directory Structure

```
.agent-os/
├── config.json          ← Project config (created by /agent-os:setup)
├── state.json           ← Current pipeline state (read/write by commands)
├── tasks/
│   ├── TASK-001.json
│   ├── TASK-002.json
│   └── ...
└── decisions/
    ├── DEC-001.json
    ├── DEC-002.json
    └── ...
```

## Reading State

Always read before writing. Never assume state is valid without reading it.

1. Check if `.agent-os/` exists before reading. If it does not exist:
   - For setup command: proceed to create it.
   - For any other command: stop and tell the user "Agent OS is not initialized. Run `/agent-os:setup` first."

2. Read `config.json` first — it defines the project context.
3. Read `state.json` to understand the current pipeline position.
4. Read the active task file (`tasks/TASK-NNN.json`) if `state.json.active_task_id` is not null.

## Writing State

### Rules

1. **Always read the file before writing it.** Never overwrite without loading current state.
2. **Atomic updates only.** Never partially update a file. Write the complete updated object.
3. **Validate before writing.** The written object must conform to its schema in `${CLAUDE_PLUGIN_ROOT}/schemas/` (the agent-os plugin root, not the project).
4. **Update `last_updated`** in `state.json` every time it changes.

### Updating state.json

```json
{
  "current_stage": "discovery",
  "stage_status": "in_progress",
  "active_task_id": "TASK-001",
  "last_updated": "2026-10-29T10:00:00Z",
  "stages_completed": [],
  "pending_approval": null,
  "blocker": null
}
```

### Creating a new task

Next task ID = count of existing files in `tasks/` + 1, zero-padded to 3 digits.
Example: if `tasks/` has TASK-001.json and TASK-002.json, next is TASK-003.json.

Template for a new task:
```json
{
  "id": "TASK-NNN",
  "title": "<task title from user>",
  "description": "<user's original description>",
  "status": "active",
  "created_at": "<ISO 8601 timestamp>",
  "completed_at": null,
  "stages": {
    "discovery":   { "status": "not_started", "started_at": null, "completed_at": null, "approved_at": null, "output_summary": "" },
    "planning":    { "status": "not_started", "started_at": null, "completed_at": null, "approved_at": null, "output_summary": "" },
    "design":      { "status": "not_started", "started_at": null, "completed_at": null, "approved_at": null, "output_summary": "" },
    "development": { "status": "not_started", "started_at": null, "completed_at": null, "approved_at": null, "output_summary": "" },
    "qa":          { "status": "not_started", "started_at": null, "completed_at": null, "approved_at": null, "output_summary": "" }
  },
  "requirements": [],
  "subtasks": [],
  "decisions": [],
  "blockers": [],
  "files_modified": [],
  "test_results": null,
  "git_baseline": {}
}
```

### Recording the git baseline

Right after creating a task file, record the uncommitted changes that already exist, so post-QA commands (for example a commit command) can tell them apart from the task's own changes. Use only read-only git commands: never stash, commit, reset or checkout.

1. Repos: `single` and `monorepo` → the workspace root, key `"."`; `multi-repo` → each path in `config.workspace_paths`, key = that path. Skip a path where `git -C <path> rev-parse --is-inside-work-tree` fails.
2. For each repo:
   - `head` = `git -C <repo> rev-parse HEAD`, or `null` when the repo has no commits yet.
   - `dirty`: from `git -C <repo> status --porcelain=v1 --untracked-files=all`, every modified, added, renamed (its new path), deleted or untracked path, relative to the repo. Exclude paths under `.agent-os/` and `.codegraph/`. For each one, `blob` = `git -C <repo> hash-object -- <path>`, or `null` when the file is deleted.
3. Write `task.git_baseline = { "<repo>": { "head": "<sha>", "dirty": [ { "path": "...", "blob": "..." } ] } }`.
4. If any repo has `dirty` entries, tell the user in one line (in `output_language`): "Aviso: hay N cambios sin commitear previos a la tarea en [repos]; quedan registrados en git_baseline y no se atribuyen a la tarea."

### Recording a stage start

Update the task file:
```json
"stages": {
  "discovery": {
    "status": "in_progress",
    "started_at": "<now>",
    ...
  }
}
```

And update state.json:
```json
{
  "current_stage": "discovery",
  "stage_status": "in_progress",
  ...
}
```

### Recording a stage completion

Update the task file stage record:
```json
"stages": {
  "discovery": {
    "status": "awaiting_approval",
    "completed_at": "<now>",
    "output_summary": "Captured 5 requirements: REQ-001 through REQ-005. Scope confirmed: user authentication with email/password and OAuth."
  }
}
```

And update state.json to reflect the pending approval.

### Recording an approval

Update the task file stage record:
```json
"stages": {
  "discovery": {
    "status": "done",
    "approved_at": "<now>",
    ...
  }
}
```

Update `stages_completed` in state.json and clear `pending_approval`.

### Creating a decision

Next decision ID = count of existing files in `decisions/` + 1, zero-padded to 3 digits.

## What NOT to do

- Do not write to `.agent-os/` files without reading them first.
- Do not skip stage records — always record start, completion, and approval timestamps.
- Do not advance `current_stage` in state.json without first recording the prior stage as done in the task file.
- Do not create or modify project source code files from within state management operations.
