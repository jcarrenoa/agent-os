---
name: tracker-sync
description: This skill should be used whenever agent-os needs to read from or write to the project's issue tracker (Jira or Plane) configured in config.tracker — starting a task from an issue key or URL ("/agent-os:start VR-123"), picking an issue from the board, analysing the board's columns to build the route of action, moving the issue through the route as the pipeline advances, creating the parent issue and sub-tasks, or posting the commits comment. Only the orchestrating command uses it, never a subagent.
version: 1.0.0
---

# Tracker Sync Protocol

Applies only when `.agent-os/config.json` has a `tracker` key. Without it, skip every step that mentions the tracker: nothing changes.

## Rules

- **Only the orchestrator.** Every tracker operation runs in the calling command (main session), never in a subagent: MCP tools do not reach the plugin's subagents. Never put tracker data in an agent prompt beyond what is already in the task file.
- **Only MCP.** `tracker.connection` is `mcp`: use the tools of `tracker.mcp.server`. There is no REST connection.
- **No credentials anywhere.** The MCP server holds the authentication. Never write a token, password or cookie to `.agent-os/`, the task file, a prompt or the output.
- **Allowed writes, and nothing else:**

  | Write | When | Approval |
  |---|---|---|
  | Transition the issue / its created sub-tasks to a `status_route` status | Route events | None: the route confirmed in setup is the consent. Logged |
  | Create the parent issue | `create_parent` | Gate, every time, in every mode |
  | Create the sub-tasks | `create_subtasks` | Gate, every time, in every mode |
  | Comment the commits | `comment_commits` | Gate, every time, in every mode |

  Never edit fields, assignees, descriptions or sprints, never delete, never link, never transition to a status that is not in `status_route`. Never move to Done unless the user mapped `done_pipeline` to Done.
- **Unattended approver mode** does not skip these gates: create and comment writes are queued in `task.external.pending_writes` and asked together at the end of the run (Pending Writes). Transitions still run.
- **Never block the pipeline.** A failed or impossible tracker operation is appended to `task.external.sync_errors`, reported in one line, and the pipeline continues. `/agent-os:tracker sync` retries.
- **Log every write** in `task.external.sync_log` (`at`, `event`, `operation`, `target`, `result`), and show the writes of a stage in that stage's summary (`Tracker: VR-123 → In Progress`). The final report has a Tracker section.

## Tool Discovery

At the first tracker operation of a command, list the available tools whose names start with `mcp__<tracker.mcp.server>__` and map each function below to one of them by name. If a function needed by the current operation has no tool: tell the user once ("El servidor MCP `<server>` no expone <función>"), record a sync error, and continue. If no tool has that prefix at all: the server is not connected in this session; suggest `claude mcp add` (README → Tracker) or `/mcp` to authenticate, record a sync error, and continue.

| Function | Jira (Atlassian Rovo MCP) | Plane (plane-mcp-server) |
|---|---|---|
| Site / cloud ID | `getAccessibleAtlassianResources` | — |
| Current user | `atlassianUserInfo` | the "get me / current user" tool |
| Projects | `getVisibleJiraProjects` | the "list projects" tool |
| Read issue | `getJiraIssue` | the "get work item" tool (by readable identifier when available) |
| Search issues | `searchJiraIssuesUsingJql` | the "list work items" tool |
| Statuses | transitions + statuses seen in a search (Board Analysis) | the "list states" tool |
| Issue types | `getJiraProjectIssueTypesMetadata` | — |
| Transitions | `getTransitionsForJiraIssue` | — |
| Move | `transitionJiraIssue` | the "update work item" tool, field `state` |
| Create | `createJiraIssue` | the "create work item" tool (`parent` for sub-tasks) |
| Comment | `addCommentToJiraIssue` | the "create work item comment" tool |

Tool names depend on the server version: match by meaning, and never guess parameters — read each tool's schema before calling it. Jira tools that take `cloudId` use `tracker.jira.cloud_id`.

## Board Analysis (route of action)

Used by `/agent-os:setup` Step 2.2 and `/agent-os:tracker map`. Builds `tracker.status_route` with the user.

1. **Read the real statuses.**
   - Jira: search `project = <KEY> ORDER BY updated DESC` (up to 50 issues) and collect the distinct statuses with their category (To Do / In Progress / Done). Take one issue in a To Do status and one in an In Progress status and read their transitions, to learn the statuses reachable from each. The MCP tools do not expose the board's column configuration: say so, and let the user add a status that was not found by typing it.
   - Plane: list the project's states (name, group `backlog | unstarted | started | completed | cancelled`, ID).
2. **Propose a route** for the events:

   | Event | When | Proposal |
   |---|---|---|
   | `intake` | Where `pick` looks (not a move) | first To Do / `unstarted` status |
   | `start` | Discovery starts | first In Progress / `started` status |
   | `development` | Development starts | an "In Development"-like status, else the `start` one |
   | `qa` | QA starts | a status named like QA, Testing or Test |
   | `done_pipeline` | QA approves | the status **before** Done (Code Review, QA, Ready for Review): the PR is still missing. Never propose Done |
   | `blocked` | A stage is blocked | a status named like Blocked, else none |

3. **Show it as a table** (`Evento | Cuándo | Columna propuesta | Otras disponibles`), then confirm each row with AskUserQuestion (at most 4 questions per call, so two calls): options = the proposal "(Recomendado)", up to two other statuses, and "Sin columna" (that event does not move anything). The user can type another status.
4. Write `status_route` with `{ status, id }` per event (`id` when known; Plane needs it) and `null` for "Sin columna". Show the final table.

## Route Events

The commands fire these events; each one moves the parent issue (if `task.external` exists) and, in create mode, the affected created sub-tasks to `status_route.<event>`:

| Event | Fired by | Sub-tasks moved (create mode) |
|---|---|---|
| `start` | discovery start (`/agent-os:start`, `/agent-os:discover`) | — |
| `development` | development start (`/agent-os:develop`, Stage 4 of start) | each sub-task when its developer run starts |
| `qa` | QA start | all created sub-tasks |
| `done_pipeline` | QA approved, after post-QA commands | all created sub-tasks |
| `blocked` | any stage set to `blocked` | the sub-tasks of the blocked run, or all |

**Transition protocol**:
1. If the route entry is `null` or missing: do nothing.
2. Read the issue's current status. If it is already the target: do nothing.
3. Jira: get the issue's transitions and pick the one whose target status matches the route status (by ID, else by name, case-insensitive). Plane: update the work item's `state` to the route `id`.
4. No such transition (not reachable from the current status), or an error: append a sync error (`result`: "<target> not reachable from <current>" or the error message without secrets) and continue.
5. On success: update `task.external.status` (or the sub-task's `status`) and log it.

## Pull

Only when `tracker.mode` is `pull` or `both`.

**From an argument** (`/agent-os:start VR-123`, or a URL): it is an issue reference when it is a key (`^[A-Z][A-Z0-9_]+-[0-9]+$`), a Jira URL containing `/browse/<KEY>` or `selectedIssue=<KEY>`, or a Plane work item URL. Otherwise it is a normal task description.

1. Read the issue. If it does not exist or cannot be read: tell the user and stop (nothing was created).
2. Fill the new task: `title` = the issue summary; `description` = the issue description, verbatim (converted to plain Markdown), followed by the acceptance criteria field when the issue has one (under a heading "Acceptance criteria"). Nothing is added or summarised.
3. `task.external = { provider, key, id, url, origin: "pulled", status, existing_subtasks: [{ key, summary, status }] }`. Log a `pull` / `read` entry.
4. The planner may cite `existing_subtasks` keys in its subtasks; agent-os never modifies them.

**Pick** (`/agent-os:tracker pick`, or `/agent-os:start` without arguments when the mode allows pull):
1. Search the issues in `status_route.intake` (Jira: `project = <KEY> AND status = "<intake>"` plus `AND assignee = currentUser()` when `pick_assigned_to_me` is true, `ORDER BY updated DESC`; Plane: the same filters). Without an `intake` status, use the project's first To Do status.
2. Show up to 10 as a table (`Clave | Título | Tipo | Prioridad | Actualizado`) and ask with AskUserQuestion: the 4 most recent as options; the user can type another key. None found: say so and ask for a key or a description.
3. Continue as **From an argument** with the chosen key.

## Create

Only when `tracker.mode` is `create` or `both` and the task has no `task.external` (a pulled task never creates a parent).

**`create_parent`**, after the requirements are approved (by the human or the approver):
1. Draft: type `jira.parent_type` (Plane: a work item), summary = task title, description = the user stories with their acceptance criteria (or the functional REQs), then non-functional requirements and out of scope, in `output_language`.
2. Gate (Write Gates). On approval: create it, set `task.external = { provider, key, id, url, origin: "created" }`, log it, then apply the current route event (`start`).
3. Declined: log `result: "declined by user"`; the task continues without a tracker issue.

**`create_subtasks`**, after the plan is approved, only if the parent exists:
1. Draft one sub-task per subtask: type `jira.subtask_type` (Plane: a work item with `parent`), parent = `task.external.key`, summary = `SUB-NN: <title>`, description = its criterion and the requirement IDs it covers, role (`backend` / `frontend`) as a label or component (`jira.role_field`; Plane: a label).
2. One gate for all of them (Write Gates). On approval: create them, store `task.external.subtasks["SUB-NN"] = { key, id }`, log each one.

## Comment Commits

After the post-QA commands have run, if `task.commits` has entries not yet commented (no `comment_commits` entry in `sync_log` for them) and the task has a tracker issue:

1. Draft, in `output_language`, one comment for the parent issue: per repository (grouped by the `repo` value of `task.commits`, the repo path), the branch, then per commit the short hash, the subject and 1–3 lines of what it does, written from the titles and criteria of the subtasks in its `subtasks`. Nothing else: no code, no file lists, no secrets.
2. In create mode, also one comment per created sub-task with only the commits whose `subtasks` include it.
3. One gate for all the comments (Write Gates). On approval: post them and log each one.

## Write Gates

Present gated writes in one block and ask with AskUserQuestion ("Sí, escribir en <Jira | Plane>" / "No"):

```
─────────────────────────────────────────
TRACKER: escrituras pendientes en [Jira | Plane]
─────────────────────────────────────────
- Crear [Story] "[title]" en [VR]
- Crear [N] sub-tareas bajo [VR-123]: SUB-01 …, SUB-02 …
- Comentar commits en [VR-123] (+ [N] sub-tareas)
[the full text of each item, collapsed per item]
─────────────────────────────────────────
```

An empty or negative answer writes nothing and logs `declined by user`.

## Pending Writes (unattended mode)

In unattended approver mode, do not stop for `create_parent`, `create_subtasks` or `comment_commits`: append them to `task.external.pending_writes`. At the end of the run, after the Approver Decisions Report and the post-QA commands, show one Write Gate with every pending write, in order (parent, then sub-tasks, then comments). On approval run them, then apply the current route event to the new issues (`done_pipeline` if QA approved). Clear `pending_writes` either way.

## Retry (`/agent-os:tracker sync`)

1. For each entry in `sync_errors`: a transition is retried towards the route status of the task's **current** event (not the old one); a read is retried. Successful retries are logged and removed from `sync_errors`.
2. `pending_writes`: show the Write Gate.
3. Report what was fixed and what still fails.

## Reporting

- In each stage summary: `Tracker: [KEY] → [status]` per transition, and any error in one line.
- In the completion report: a `### 🔗 Tracker` table (`Cuándo | Operación | Destino | Resultado`) from `sync_log`, plus the open `sync_errors` and a reminder of `/agent-os:tracker sync` when there are any.
