---
name: tracker-sync
description: This skill should be used whenever agent-os needs to read from or write to the project's issue tracker (Jira or Plane) configured in config.tracker — starting a task from an issue key or URL ("/agent-os:start VR-123"), picking an issue from the board, analysing the board's columns to build the route of action, moving the issue through the route as the pipeline advances, creating the parent issue and sub-tasks, or posting the commits comment. Only the orchestrating command uses it, never a subagent.
version: 1.8.0
---

# Tracker Sync Protocol

Applies only when `.agent-os/config.json` has a `tracker` key. Without it, skip every step that mentions the tracker: nothing changes.

## Permissions

`tracker.permissions` says what agent-os may do in the tracker. Each permission is chosen on its own in setup, and this section overrides every other one in this skill: a step that needs a permission the config does not grant is skipped.

| Permission | Allows | Gate |
|---|---|---|
| `read` | Read issues, statuses, types, transitions and comments (Pull, Pick, Board Analysis reads, Comment Commits Step 2). Always granted: nothing else works without it | — |
| `move` | Route events and their transitions (Route Events), `held_done_pipeline` and its recovery, retrying failed transitions. Requires `status_route` | None: the route confirmed in setup is the consent. Logged |
| `create` | `create_parent` and `create_subtasks` (Create). Required for `mode` `create` or `both`; without it `mode` is `pull` | Every time, in every mode |
| `comment` | `comment_commits` (Comment Commits) and its retries | Every time, in every mode |

**Effective permissions**, used by every command:

1. `tracker.permissions` when present, plus `read` if it is missing from the list.
2. Otherwise the legacy `tracker.access`: `"read"` → `["read", "comment"]`; `"write"` or missing → `["read", "move", "create", "comment"]`, so older configs keep working.

Commands refer to these as "the `<name>` permission".

When a permission is missing:

- **Skip, do not fail**: the step does nothing, nothing goes to `sync_errors`, and the pipeline continues. A write that is skipped is not queued in `pending_writes`. This covers every command that uses this skill, including `/agent-os:commit`, `/agent-os:approve` and `/agent-os:review`.
- **Say it once per run and permission**, at the first step skipped, in one line in `output_language`: "Tracker sin permiso para <mover | crear | comentar> en <Jira | Plane>: se omite."
- **Without `move`**: no route event moves anything, created issues and sub-tasks are not moved either, `held_done_pipeline` is not set, and `/agent-os:tracker map` does nothing (say so in one line).
- **Without `create`**: `mode` is `pull`, and comments need a linked issue (`task.external.key`), which only Pull and Pick provide.
- **Without `comment`**: no commits comment is drafted, offered or queued, and `/agent-os:commit` offers none.
- **Retry** (`/agent-os:tracker sync`) runs only what the current permissions allow. Writes queued or held before a permission was removed stay in the task file and are not offered until it is granted again.
- **Setup** (`/agent-os:setup` Step 2.2) asks only the questions of the granted permissions: types and mode with `create`, the route with `move`, the owner name with `comment`.

## Rules

- **Only the orchestrator.** Every tracker operation runs in the calling command (main session), never in a subagent: MCP tools do not reach the plugin's subagents. Never put tracker data in an agent prompt beyond what is already in the task file.
- **Only MCP.** `tracker.connection` is `mcp`: use the tools of `tracker.mcp.server`. There is no REST connection.
- **No credentials anywhere.** The MCP server holds the authentication. Never write a token, password or cookie to `.agent-os/`, the task file, a prompt or the output.
- **Allowed writes, and nothing else**, each only with its permission (Permissions):

  | Write | When | Approval |
  |---|---|---|
  | Transition the issue / its created sub-tasks to a `status_route` status | Route events | None: the route confirmed in setup is the consent. Logged |
  | Create the parent issue | `create_parent` | Gate, every time, in every mode |
  | Create the sub-tasks | `create_subtasks` | Gate, every time, in every mode |
  | Comment the commits | `comment_commits` | Gate, every time, in every mode |

  Never edit fields, assignees, descriptions or sprints, never delete, never link, never transition to a status that is not in `status_route`. Never move to Done unless the user mapped `done_pipeline` to Done.

  On the Atlassian server: never use `executeWrite` or `executeDestructive`. Never pass `commentId` to `addOrEditJiraIssueComment`: it edits an existing comment and replaces its whole body. Never comment through the `update` of `transitionJiraIssue`.
- **Unattended approver mode** does not skip these gates: create and comment writes are queued in `task.external.pending_writes` and asked together at the end of the run (Pending Writes). Transitions still run, with the `move` permission.
- **Never block the pipeline.** A failed or impossible tracker operation is appended to `task.external.sync_errors`, reported in one line, and the pipeline continues. `/agent-os:tracker sync` retries.
- **Log every write** in `task.external.sync_log` (`at`, `event`, `operation`, `target`, `result`; a comment is recorded before it is posted, Comment Commits Step 9), and show the writes of a stage in that stage's summary (`Tracker: VR-123 → [status]`). The final report has a Tracker section. `event` is the route event, the write (`create_parent`, `create_subtasks`, `comment_commits`), `pull` or `retry`, never a gate name such as `requirements-approved`.
- **A linked issue means `task.external.key`.** `task.external` can exist without a key, to hold the log, the errors and the queue of a task whose issue is not created. It then has only `provider` (`config.tracker.provider`) and those fields, and it does not count as a linked issue. If one of those has to be written and `task.external` is missing, create it that way first.

## Tool Discovery

At the first tracker operation of a command, list the available tools whose names start with `mcp__<tracker.mcp.server>__` and map each function below to one of them by name. If a function needed by the current operation has no tool, or its tool returns no data: tell the user once ("El servidor MCP `<server>` no expone <función>"), record a sync error, and continue. If no tool has that prefix at all: the server is not connected in this session; suggest `claude mcp add` (README → Tracker) or `/mcp` to authenticate, record a sync error, and continue.

| Function | Jira (Atlassian Rovo MCP) | Plane (plane-mcp-server) |
|---|---|---|
| Site / cloud ID | `getAccessibleAtlassianResources` | — |
| Current user | Two steps: `atlassianUserInfo` gives the `accountId`, then `getPeopleInfo` (`cloudId`, `accountId`) gives the name, read by meaning | the "get me / current user" tool |
| Projects | `executeRead` name `listJiraProjects`, inputs `{ query, action: "view", expand: "issueTypes" }` | the "list projects" tool |
| Read issue | `getJiraIssue` | the "get work item" tool (by readable identifier when available) |
| Search issues | `searchJiraIssuesUsingJql` | the "list work items" tool |
| Statuses | `executeRead` name `listJiraStatuses`, inputs `{ mode: "project", projectKey }` | the "list states" tool |
| Issue types | In the Projects response (`expand: "issueTypes"`); as an alternative, `executeRead` name `listJiraProjectIssueTypesMetadata`, inputs `{ projectIdOrKey }` | — |
| Transitions | `executeRead` name `listJiraIssueTransitions`, inputs `{ issueIdOrKey }`, not `getJiraIssue` | — |
| Move | `transitionJiraIssue` (`cloudId`, `issueIdOrKey`, `transitionId`) | the "update work item" tool, field `state` |
| Create | `createJiraIssue` (`cloudId`, `projectKey`, `summary`, `issueType`; optional `parent`, `description`, `labels`) | the "create work item" tool (`parent` for sub-tasks) |
| Comment | `addOrEditJiraIssueComment` (`cloudId`, `issueIdOrKey`, `commentBody`) | the "create work item comment" tool |
| Read comments | `listJiraIssueComments` (`cloudId`, `issueIdOrKey`), not `getJiraIssue`, reading every page with `startAt` until `isLast` | the "list work item comments" tool, reading every page |

Tool names depend on the server version and tenants differ: match by meaning, and never guess parameters — read each tool's schema before calling it. Get `executeRead` operation names from `discover`, never by guessing: those in the table are examples that were seen, not a guarantee. Jira tools that take `cloudId` use `tracker.jira.cloud_id`.

`executeRead` answers inside `{ "data": ... }`; read the result by meaning. Its `responseFields` filter, when wrongly aimed, returns `{ "data": {} }` with no error, so an empty result does not mean there is no data: ask again without the filter or with the right fields before treating it as "returns no data".

Two functions are best effort and record no sync error when they have no tool or fail: Current user (Owner Name) and Read comments (Comment Commits, Step 2).

## Board Analysis (route of action)

Used by `/agent-os:setup` Step 2.2 and `/agent-os:tracker map`. Builds `tracker.status_route` with the user.

1. **Read the real statuses.**
   - Jira: read the project's statuses (Statuses), with their category when the response gives one. If an issue exists in a to-do status and another in an in-progress one (Search issues), read their transitions, to learn the statuses reachable from each. The MCP tools do not expose a board's column configuration, and team-managed projects do not share one: say so, and let the user add a status that was not found by typing it.
   - Plane: list the project's states (name, group `backlog | unstarted | started | completed | cancelled`, ID).
2. **Propose a route** for the events. Status names can be localized (`Por hacer`, `En curso`): the proposals below describe a status's category or meaning, in any language, and always use the name as read.

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

The commands fire these events; each one moves the parent issue (if the task has a linked issue) and, in create mode, the affected created sub-tasks to `status_route.<event>`:

| Event | Fired by | Sub-tasks moved (create mode) |
|---|---|---|
| `start` | discovery start (`/agent-os:start`, `/agent-os:discover`) | — |
| `development` | development start (`/agent-os:develop`, Stage 4 of start) | each sub-task when its developer run starts |
| `qa` | QA start | all created sub-tasks |
| `done_pipeline` | QA approved, last of all: after the post-QA commands, Comment Commits and the Pending Writes gate. Held only if a post-QA command failed (Held `done_pipeline`): a declined or failed comment does not hold it | all created sub-tasks |
| `blocked` | any stage set to `blocked` | the sub-tasks of the blocked run, or all |

**Transition protocol**:
1. If the route entry is `null` or missing: do nothing.
2. Read the issue's current status. If it is already the target: do nothing.
3. Jira: list the issue's transitions, pick the one whose target status matches the route status (by ID, else by name, case-insensitive), and move by its `transitionId`. A transition is not its target status, even when they look alike: `transitionName` is the transition's own name, so never pick by it. Read where the target status is from the tool's schema or the response (seen as `to.name`). A transition marked as not available (seen as `isAvailable: false`) is never picked. Plane: update the work item's `state` to the route `id`.
4. No such transition (not reachable from the current status), a target status that cannot be determined, or an error: append a sync error (`result`: "<target> not reachable from <current>", or the reason or error message without secrets) and continue, without guessing.
5. Confirm the new status from the move's result or by reading the issue. If it is the route status: update `task.external.status` (or the sub-task's `status`) and log it. If it is not, or it cannot be determined: append a sync error.

## Pull

Only when `tracker.mode` is `pull` or `both`.

**From an argument** (`/agent-os:start VR-123`, or a URL): it is an issue reference when it is a key (`^[A-Z][A-Z0-9_]+-[0-9]+$`), a Jira URL containing `/browse/<KEY>` or `selectedIssue=<KEY>`, or a Plane work item URL. Otherwise it is a normal task description.

1. Read the issue. If it does not exist or cannot be read: tell the user and stop (nothing was created).
2. Fill the new task: `title` = the issue summary; `description` = the issue description, verbatim (converted to plain Markdown), followed by the acceptance criteria field when the issue has one (under a heading "Acceptance criteria"). Nothing is added or summarised.
3. `task.external = { provider, key, id, url, origin: "pulled", status, epic, existing_subtasks: [{ key, summary, status }] }`. Log a `pull` / `read` entry.
4. **Epic** (Jira): `epic = { key, summary }` of the issue's parent when that parent is an epic, read from the same issue response (seen as `fields.parent`, with its issue type). A parent is an epic when its issue type has hierarchy level 1, or, when the level is not given, its type name means epic in any language (`Epic`, `Épica`). A sub-task's parent is a story, not an epic: then read that story and take its parent the same way. No parent, or a parent that is not an epic: no `epic` field. Plane: no `epic` field. It is a read: it needs only the `read` permission.
5. The planner may cite `existing_subtasks` keys in its subtasks; agent-os never modifies them.

**Pick** (`/agent-os:tracker pick`, or `/agent-os:start` without arguments when the mode allows pull):
1. Search the issues in `status_route.intake` (Jira: `project = <KEY> AND status = "<intake>"` plus `AND assignee = currentUser()` when `pick_assigned_to_me` is true, `ORDER BY updated DESC`; Plane: the same filters). Without an `intake` status, use the project's first To Do status.
2. Show up to 10 as a table (`Clave | Título | Tipo | Prioridad | Actualizado`) and ask with AskUserQuestion: the 4 most recent as options; the user can type another key. None found: say so and ask for a key or a description.
3. Continue as **From an argument** with the chosen key.

## Create

Only when `tracker.mode` is `create` or `both`. A pulled task never creates anything. Log both writes in `task.external.sync_log` with `event` = the write, `operation: "create"`, and `target`: the new key, or `parent` / the `SUB-NN` when nothing was created.

Each create runs once, however many callers fire it. This applies to the callers that fire a create; the Pending Writes gate runs what is queued. Skip `create_parent` when the task has a linked issue, when it is already in `pending_writes`, when `sync_log` has an entry with `event: "create_parent"`, or when `sync_errors` has one whose call was sent, because that failure is ambiguous and creating again could duplicate the issue. An error before the call is recorded with a `result` that starts with `not sent:`. It does not count, so the create is asked again the next time it is fired. Skip `create_subtasks` when it is already in `pending_writes`, or when `sync_log` has an entry with `event: "create_subtasks"` and `result: "declined by user"`; otherwise draft only the subtasks not yet in `task.external.subtasks`, and stop if none is left.

Jira: `jira.parent_type` and `jira.subtask_type` are the type names as the project shows them, which can be localized, and there is no default. If the one a create needs is missing, read the project's issue types and ask for it right before the gate. If it still cannot be obtained, record a sync error with `result: "not sent: <reason>"` and do not create.

**`create_parent`**, after the requirements are approved (by the human or the approver), only if the task has no linked issue:
1. Draft: type `jira.parent_type` (Plane: a work item), summary = task title, description = the user stories with their acceptance criteria (or the functional REQs), then non-functional requirements and out of scope, in `output_language`.
2. Gate (Write Gates). On approval: create it, add `key`, `id`, `url` and `origin: "created"` to `task.external`, keeping its other fields, log it, then apply the current route event (`start`).
3. Declined: log `result: "declined by user"`; the task continues without a tracker issue.

**`create_subtasks`**, after the plan is approved, only if agent-os created the parent (`origin: "created"`) or its `create_parent` is queued in `pending_writes`:
1. Draft one sub-task per subtask: type `jira.subtask_type` (Plane: a work item with `parent`), parent = `task.external.key`, summary = `SUB-NN: <title>`, description = its criterion and the requirement IDs it covers, role (`backend` / `frontend`) as a label or component (`jira.role_field`; Plane: a label).
2. One gate for all of them (Write Gates). On approval: create them, store `task.external.subtasks["SUB-NN"] = { key, id }`, log each one.

## Comment Commits

Runs at three points:
- **Post-QA**: after the post-QA commands of an approved QA (`/agent-os:qa`, `/agent-os:start`).
- **Manual**: when the user runs `/agent-os:commit` directly (that command's Step 5).
- **Retry**: `/agent-os:tracker sync`.

**Pending commits.** For a target (the parent issue, or a created sub-task), the pending commits are the entries of `task.commits` (for a sub-task, only those whose `subtasks` include it) whose `hash` is in no `sync_log` entry with `operation = "comment"`, that `target`, and `result` equal to `attempting`, `posted` or `declined by user`. A target without pending commits gets no comment, so running this again never repeats a commit.

**Comment entries**, in `task.external.sync_log`: `{ at, event, operation: "comment", target: <key>, result, commits: [<hashes in that comment>] }`, with `event` = `comment_commits` (`retry` during Retry). `result` is one of:

| `result` | Meaning |
|---|---|
| `attempting` | Written before the call, with no outcome recorded: whether the comment reached the tracker is unknown |
| `posted` | The comment is on the issue |
| `failed` | The call returned an error. The call may still have reached the tracker. The error is in `sync_errors` |
| `declined by user` | The user answered no at the gate |

1. If the task has no linked issue and no queued `create_parent`: stop, silently. If `task.external.provider` is not `config.tracker.provider` (the tracker changed since), say so in one line and stop.
2. **Verify earlier attempts.** For each `attempting` entry, and each `failed` entry with commits still pending, read that issue's comments (Read comments) and look for the entry's hashes. A comment is posted as a whole, so one hash found means `posted`. None found means `failed`. If the comments cannot be read (no tool, or the read fails), the entry stays as it is.
3. If no target has pending commits or `attempting` entries: stop, silently.
4. **Post-QA in unattended approver mode**: append `{ operation: "comment_commits", target: "parent", queued_at }` to `pending_writes` (once) and stop. The comments are drafted when the Pending Writes gate is shown. Manual and Retry were asked for by the user: they always continue to the gate now.
5. **Attempts still unknown.** Never post these commits blindly. For each `attempting` entry left, ask with AskUserQuestion, header "Comentario": "No se pudo comprobar si el comentario con [hashes] llegó a [KEY]. Revísalo en [url].", options "Ya está publicado" (→ `posted`) / "No está, ofrecerlo de nuevo" (→ `failed`) / "Dejarlo sin resolver" (stays `attempting`, those commits are not offered, and `/agent-os:tracker sync` asks again). If the gate would still include commits of a `failed` entry that could not be verified, say in the gate that a duplicate is possible.
6. Draft, in `output_language`, one comment for the parent issue with its pending commits: per repository (grouped by the `repo` value of `task.commits`, the repo path), the branch, then per commit the short hash, the subject and 1–3 lines of what it does, written from the titles and criteria of the subtasks in its `subtasks`. In create mode, also one comment per created sub-task with its pending commits. No code, no file lists, no secrets. If nothing is left to comment, stop.
7. Resolve the name (Owner Name). When there is one, the last line of every comment is the owner line. Nothing comes after it, and nothing else is added to the comment.
8. One gate for all the comments (Write Gates).
9. **Record, then post.** On approval, for each comment, in order:
   1. Write its `attempting` entry to the task file. If that write is denied or fails: do not post this comment or the following ones, and tell the user in one line. Their commits stay pending.
   2. Post the comment.
   3. Update that entry's `result`: `posted`, or `failed` plus a `sync_errors` entry with the same fields and the error as `result`. After a failure, continue with the next comment. If this update cannot be written, tell the user in one line: the entry stays `attempting`, and the next run verifies it instead of posting again.

   A declined gate logs one entry per target in `task.external.sync_log`, with `result: "declined by user"` and its `commits`. If it cannot be written, tell the user in one line that those commits will be offered again.

## Owner Name

The owner line closes every commits comment: `Encargado: <name>` when `output_language` is `es`, `Owner: <name>` when it is `en`. The owner is the person using agent-os, not the issue's assignee: never read the assignee for this, and never edit it.

Resolve the name once per command run, first match wins:

1. `display_name` in `~/.agent-os/profile.json` (schema: `PLUGIN_ROOT/schemas/profile.schema.json`).
2. The display name of the tracker account, from the Current user function. Use only the name, never the email or account ID: a result without a name, or whose name has the form of an email, counts as not obtained. Do not store it.
3. Ask, right before the gate's yes/no question, so that the gate shows the final text. AskUserQuestion in `output_language`, header "Encargado": "No hay nombre configurado ni se pudo leer de tu cuenta de <Jira | Plane>. ¿Qué nombre cierra el comentario?", options "Escribir el nombre" (then ask the user to type it) / "Omitir la línea". The answer, trimmed and on one line, applies to this gate only, and an empty one omits the line. Afterwards, mention `/agent-os:tracker owner` in one line. In unattended mode this is asked when the Pending Writes gate is shown, never before.

Never invent the name or take it from anywhere else (git config, the OS user, the issue).

**Configuring it** (`/agent-os:setup` Step 2.2, `/agent-os:tracker owner`). The name is personal and `.agent-os/config.json` may be shared through git, so it is only stored at user level. AskUserQuestion in `output_language`, header "Encargado", "¿Qué nombre cierra los comentarios de commits en el tracker? Se guarda solo en tu usuario (`~/.agent-os/profile.json`), no en el proyecto.":
- "Mantener <display_name> (actual)": only when one is set. Nothing changes.
- "No guardar nombre": "Se usa el de tu cuenta de <Jira | Plane> o se pregunta al comentar." Add "(Recomendado)" when no name is set. Removes `display_name`.
- "Escribir otro": then ask the user to type it. Save it trimmed and on one line. An empty answer changes nothing.

To save: read the file (create it and `~/.agent-os/` if missing), change only `display_name`, and write it back. Tell the user the result in one line.

## Write Gates

Present gated writes in one block and ask with AskUserQuestion ("Sí, escribir en <Jira | Plane>" / "No"):

```
─────────────────────────────────────────
TRACKER: escrituras pendientes en [Jira | Plane]
─────────────────────────────────────────
- Crear [parent_type] "[title]" en [VR]
- Crear [N] sub-tareas bajo [VR-123]: SUB-01 …, SUB-02 …
- Comentar commits en [VR-123] (+ [N] sub-tareas) · [Encargado: <name> | sin línea de encargado]
[the full text of each item, collapsed per item, comments ending with their owner line]
─────────────────────────────────────────
```

An empty or negative answer writes nothing and logs `declined by user`.

A gated write is resolved in this run or stays queued. It is never lost:

1. Show the block and ask. If AskUserQuestion is not available, show the block as text, ask yes or no, and end the turn waiting for the answer. Never replace the gate with a note or a suggestion to use `/agent-os:tracker sync`.
2. Only if the gate really cannot be shown, because the run is ending, append the write to `task.external.pending_writes` and say so: `/agent-os:tracker sync` will offer it. If that cannot be written either, tell the user in one line what was not written.

## Pending Writes

Queued writes wait here for their gate. In unattended approver mode they are always queued. In any other mode a write is queued only when its gate could not be shown (Write Gates), and it waits for `/agent-os:tracker sync`.

In unattended approver mode, do not stop for `create_parent`, `create_subtasks` or `comment_commits`: append them to `task.external.pending_writes`. At the end of the run, after the Approver Decisions Report, the post-QA commands and Post-QA Comment Commits (which queues its write), show one Write Gate with every pending write, in order (parent, then sub-tasks, then comments). Skip the gate when nothing is queued.

Draft everything queued when showing the gate. Comments use Steps 5–7 of Comment Commits: when the parent or the sub-tasks are queued too, all their commits are pending, and their comments name them as the issues to be created. Sub-tasks are drafted with the parent as the issue to be created.

On approval run the writes in order: the parent first, then the sub-tasks under its new key, then the comments (as in Step 9), and then apply the current route event to the new issues (`done_pipeline` if QA approved and not held). If the parent is not created, nothing that depends on it runs: no sub-tasks and no comments. If the gate is declined, log `declined by user` for the creates and create nothing. Clear `pending_writes` once the gate is answered, either way; if it could not be shown, they stay queued. Declined comments are logged as in Step 9 of Comment Commits only when their target already has a key; otherwise their commits simply stay pending.

## Held `done_pipeline`

When a post-QA command fails (`05-testing-qa.md` → Post-QA Command Failures), `done_pipeline` is not applied: set `task.external.held_done_pipeline = true`, unless the route entry is `null` or the task has neither an issue nor a queued `create_parent`. Clear it if that queued parent ends up not being created. Report in one line that the issue was not moved and how to recover: fix the cause (for a commit, `/agent-os:commit`), then `/agent-os:tracker sync`.

Recovery, in Retry: this is the only transition of a finished task that is offered, because it never ran and the user confirms it. Read the issue's current status, then ask with AskUserQuestion, header "Mover issue": "El command [name] falló: [reason]. [KEY] está en [current]. ¿Moverlo a [route status]?":
- "Sí, mover": apply the `done_pipeline` event (Transition protocol) and clear the flag. A failed transition goes to `sync_errors`, and the flag stays.
- "Todavía no": nothing changes. Sync asks again next time.
- "No, dejarlo así": clear the flag. Nothing is moved.

## Retry (`/agent-os:tracker sync`)

**Tasks covered.** A task has something to retry when it has `pending_writes`, `held_done_pipeline`, `attempting` comment entries, comment errors in `sync_errors`, or pending commits on an existing issue, or, only while it is active, any other entry in `sync_errors`. Retry covers the active task, then the finished ones (`status = "done"`, newest first) that have something to retry. With `/agent-os:tracker sync TASK-NNN`, only that task, and commits declined earlier count as pending again. Without the argument, a declined commit is not offered again. Finishing a task never closes its comment retries.

A task whose `task.external.provider` is not `config.tracker.provider` is skipped, and reported in one line: its issue is in another tracker.

For each task, with one Write Gate per task and the name resolved once:

1. `sync_errors`, active task: a transition is retried towards the route status of the task's **current** event (not the old one); a read is retried. Successful retries are logged and removed from `sync_errors`. Finished task: transitions and reads are not retried, because the issue may have moved on since. They may be listed in one line as not retried, and do not count as something pending. The held event is the exception (Held `done_pipeline`).
2. `pending_writes`, then Comment Commits for the pending commits, in that one gate. Comment Commits verifies earlier attempts first (its Step 2), so nothing is posted again blindly. A target's comment errors are removed from `sync_errors` once its comment is posted or declined, or when the target has no pending commits left, whether or not a gate was shown.
3. `held_done_pipeline`: the recovery question of Held `done_pipeline`, after the comments.
4. Report what was fixed and what still fails.

## Reporting

- In each stage summary: `Tracker: [KEY] → [status]` per transition, and any error in one line.
- In the completion report: a `### 🔗 Tracker` table (`Cuándo | Operación | Destino | Resultado`) from `sync_log`, plus the open `sync_errors`, and a reminder of `/agent-os:tracker sync` only when the task has something to retry (Retry → Tasks covered). A finished task's transition or read errors are shown without that reminder.
