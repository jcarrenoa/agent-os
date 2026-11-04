---
name: design-sync
description: This skill should be used whenever agent-os needs to read from or write to the project's design source (Figma) configured in config.design_source — finding the Figma links of a task ("/agent-os:start VR-123" whose issue has a Figma link, "/agent-os:design-source link <url>"), taking the design snapshot the agents read, refreshing it, commenting design findings on Figma nodes, or writing Code Connect mappings. Only the orchestrating command uses it, plus the snapshot reader it spawns (Snapshot); never an agent-os agent.
version: 1.2.0
---

# Design Sync Protocol

Applies only when `.agent-os/config.json` has a `design_source` key. Without it, skip every step that mentions the design source: nothing changes.

agent-os agents cannot reach MCP tools: they only get the tools in their frontmatter. Built-in Claude Code agents such as `general-purpose` do get the session's MCP tools. So the orchestrator spawns a `general-purpose` **snapshot reader** that reads the design and writes it to files under `.agent-os/designs/TASK-NNN/`, and the agent-os agents read those files (**Snapshot**). The raw design never enters the orchestrator's context, where it would be re-read on every later turn of the task. agent-os agents never call the design source.

## Permissions

`design_source.permissions` says what agent-os may do in the design source. Each permission is chosen on its own in setup, and this section overrides every other one in this skill: a step that needs a permission the config does not grant is skipped.

| Permission | Allows | Gate |
|---|---|---|
| `read` | Read files, frames, design context, variables, screenshots, metadata, comments and Code Connect mappings. Always granted: nothing else works without it | — |
| `comment` | Post design findings as comments on their Figma nodes (Comment Findings) | Every time, in every mode |
| `code_connect` | Write Code Connect mappings between Figma components and code components (Code Connect) | Every time, in every mode |

`read` is added when it is missing from the list. When a permission is missing:

- **Skip, do not fail**: nothing goes to `sync_errors`, nothing is queued, and the pipeline continues.
- **Say it once per run and permission**, at the first step skipped, in one line in `output_language`: "Figma sin permiso para <comentar | Code Connect>: se omite."
- **Setup** (`/agent-os:setup` Step 2.4) and `/agent-os:design-source permissions` ask only for the permissions; none needs extra questions.

## Rules

- **Only the orchestrator, and its snapshot reader.** Every design-source operation runs in the calling command (main session), except the reads of **Snapshot reader**, which run in the `general-purpose` subagent it spawns. The reader never asks the user, never writes the task file and never writes to Figma. Never put design content in an agent prompt: agents receive the brief path through their Extensions block (`skill-resolution` → Tier 6).
- **Only MCP.** `design_source.connection` is `mcp`: use the tools of `design_source.mcp.server`.
- **No credentials anywhere.** The MCP server holds the authentication. Never write a token or cookie to `.agent-os/`, the task file, a prompt or the output.
- **Allowed writes, and nothing else**, each only with its permission and after its gate: comments on nodes (`comment`) and Code Connect mappings (`code_connect`). Never create, edit, move or delete frames, components, variables, styles or files; never resolve or delete comments; never use a tool that writes to the canvas.
- **Never block the pipeline.** A failed or impossible operation is appended to `task.design.sync_errors` (`at`, `event`, `operation`, `target`, `result`), reported in one line, and the pipeline continues. `/agent-os:design-source refresh` retries the snapshot.
- **Log every operation** in `task.design.sync_log`, with `event` = `snapshot`, `refresh`, `comment_findings` or `code_connect`, and `operation` = `read`, `comment` or `code_connect`.
- **Unattended approver mode** does not skip the gates: the writes are asked at the end of the run, together with the post-QA commands tagged `[gate]`.

## Tool Discovery

At the first design-source operation of a command, list the available tools whose names start with `mcp__<design_source.mcp.server>__` and map each function below to one of them by meaning. Names change between server versions: those in the table were seen, not guaranteed. Read each tool's schema before calling it, and never guess parameters. If a function needed now has no tool: tell the user once ("El servidor MCP `<server>` no expone <función>"), record a sync error, and continue. If no tool has that prefix at all: the server is not connected in this session; suggest `/mcp` to authenticate, record a sync error, and continue.

| Function | Figma MCP server (seen as) | Needed by |
|---|---|---|
| Session | `create_session` | Every operation on a file, best effort |
| Metadata (node tree: pages, frames, names, sizes) | `get_metadata` | Snapshot |
| Design context (layout, styles, texts, reference code) | `get_design_context` (older: `get_code`) | Snapshot |
| Variables (tokens: colors, spacing, typography) | `get_variable_defs` | Snapshot |
| Screenshot | `get_screenshot` (older: `get_image`) | Snapshot, best effort |
| Code Connect map (read) | `get_code_connect_map` | Snapshot, best effort |
| Current user | `whoami` | Setup check, best effort |
| Post comment on a node | a "post / add comment" tool, when exposed | Comment Findings |
| Write Code Connect mapping | `add_code_connect_map` or `send_code_connect_mappings` | Code Connect |

In the snapshot reader, "tell the user" and "record a sync error" both mean: add `{ target, result }` to the `errors` of its returned block.

Best-effort functions record no sync error when they have no tool or fail. The comment and Code Connect tools may not exist on every server or plan: without one, the permission has no effect, and the user is told once.

**Session.** When the server exposes the Session function, call it once per command run and file, before the first other call on that file, with the file key and a one-line description that states the purpose and that it is read-only (for example "Read-only: snapshot of TASK-002 frames"; for a gated write, name the write instead). Pass the returned session ID to every later call on that file whose schema accepts it. Without the function, or if it fails, call the tools without a session.

**Large results.** A result too large for the context is saved to a file by Claude Code, and the call returns that file's path instead of the data. It is not a failure and records no sync error: read that file in parts (or search it for the node IDs and names you need) instead of calling again. To avoid it, ask for the smallest node that answers the question: the frame or section named by the link before its page, and a page before the whole file.

## Links

A Figma link is a URL on `figma.com` of the form `/(design|file|proto)/<fileKey>/<name>?node-id=<a>-<b>`. Parse it into `{ url, file_key, node_id }` with `node_id` = `<a>:<b>` (the API form). A branch URL `/design/<fileKey>/branch/<branchKey>/…` uses `<branchKey>` as `file_key`. A link without `node-id` points to the whole file: the user chooses the frames (Snapshot, `CHOOSE_FRAMES`).

**Finding the links of a task** (Discovery, before the requirements-analyst runs; `/agent-os:start` and `/agent-os:discover`):

1. Collect the Figma links, without duplicates, from the task argument and `task.description` (which, for a pulled issue, is the issue description verbatim).
2. None found and not unattended mode: AskUserQuestion in `output_language`, header "Diseño", "¿Hay diseño en Figma para esta tarea?": "Pegar link" (then ask the user to paste one or more) / "No hay diseño". Unattended mode: do not ask; continue without design.
3. Store them in `task.design.links`. No links: no `task.design`, and nothing else in this skill runs for the task.

## Snapshot

Takes the design into files the agents can read. Runs right after Links, and again with `/agent-os:design-source refresh` or `link`. The orchestrator keeps every question and every task-file write; the snapshot reader does the reading and writes the design files.

### Orchestrator

1. **Spawn the reader**: Agent tool, `subagent_type: "general-purpose"`, `model: "sonnet"`. Its prompt contains only: the absolute path of this skill and "Follow **Snapshot reader** in it"; the task ID; `design_source.mcp.server`; `task.design.links` as JSON; `max_frames` (default 10); the absolute output directory `<workspace root>/.agent-os/designs/TASK-NNN/`; `output_language`; and, only on a second run, `Frames: <node_id list>`.
2. **Read its block.**
   - `CHOOSE_FRAMES` (a link names no node, or there are more than `max_frames` frames): AskUserQuestion with `multiSelect: true` listing the 4 largest or most recently named frames; the user can type others. Unattended mode: take the first `max_frames` in canvas order and say so in one line. Then spawn the reader again with `Frames:`.
   - `SNAPSHOT`: continue with step 3.
3. **Record** in the task file: `task.design = { links, snapshot_at, brief, frames: [{ node_id, name, file_key, url, raw, screenshot }] }` from the block (paths relative to the workspace root), a `snapshot` (or `refresh`) entry in `sync_log`, and each of its `errors` in `sync_errors`.
4. **Report** in one line: "Diseño: N frames de <file> → .agent-os/designs/TASK-NNN/brief.md".
5. **Fallback.** If the reader returns `NO_MCP` (no tool with the server's prefix is available to it), or the Agent tool fails, run **Snapshot reader** yourself in the main session, asking the user directly where it would return `CHOOSE_FRAMES`, and say in one line that the design went into the orchestrator's context this time.

### Snapshot reader

Runs in the `general-purpose` subagent (or, as a fallback, in the orchestrator). It never asks the user, never writes the task file, never writes to Figma, and returns no design content: only one of the blocks below.

1. **Tools.** Follow Tool Discovery. No tool with the prefix `mcp__<server>__`: return `NO_MCP` and stop.
2. **Resolve the frames.** With `Frames:` in the prompt, take exactly those. Otherwise, for each link, read its metadata. A frame or component is taken as is. A page, section or file: list its top-level frames. A page's metadata is often too large: follow **Large results** in Tool Discovery, and list only the top-level frames (name, `node_id`, size) from it. If a link names no node, or there are more than `max_frames` frames, return `CHOOSE_FRAMES` and stop:
   ```
   CHOOSE_FRAMES
   node_id | name | size | page
   ```
3. **Read each frame**: design context, variables, the Code Connect map (best effort) and the screenshot (best effort).
4. **Write the files** under the output directory:
   - `frames/<node>.md` (`<node>` = `node_id` with `:` replaced by `-`): the design context exactly as returned, then the variables of that frame. Nothing is summarised.
   - `screens/<node>.png`: only when the screenshot tool returns a downloadable URL; download it with `curl -sSfL` and check the file is a PNG. An image returned inline cannot be saved: write nothing, and describe it in the brief instead. Never record a screenshot path that was not written.
   - `brief.md`: the agents' entry point, in `output_language`, with exactly these sections:
     - **How to use this design** (copy verbatim, translated to `output_language`): "This is the approved UI design for the task. requirements-analyst: derive UI acceptance criteria from it (states, texts, validations) and ask about anything it does not cover. architect: map its components to existing code components before proposing new ones. frontend-developer: implement it visually as designed, using the project's stack, design tokens and existing components; the reference code in frames/ is a hint, never copied verbatim, and never a reason to add a library the project does not use. qa-engineer: compare the implementation with each frame and record each difference as an issue with location `design:<node_id>`, and each state the design does not cover (loading, empty, error, disabled) as a `minor` issue with the same location."
     - **Frames**: one row per frame: name, `node_id`, link, screenshot path or "sin imagen", raw file path.
     - **Per frame**: layout and hierarchy; the components it uses (name, `node_id`, variant, and its Code Connect mapping when known); every visible text, verbatim; the tokens it uses (name → value); the states and variants present, and those missing among loading, empty, error and disabled.
     - **Tokens**: the union of the variables read, name → value.
5. **Return** only this block, with paths relative to the workspace root:
   ```
   SNAPSHOT
   { "file_name": "<Figma file name>", "brief": ".agent-os/designs/TASK-NNN/brief.md",
     "frames": [{ "node_id", "name", "file_key", "url", "raw", "screenshot": "<path or null>" }],
     "errors": [{ "target", "result" }] }
   ```

Once `brief.md` exists, the `skill-resolution` skill passes it to the requirements-analyst, approver, architect, frontend-developer and qa-engineer (Tier 6).

**Refresh after Discovery.** When the snapshot is refreshed while the task is past Discovery, set `task.design.changed_after` to the current stage and warn in one line that the approved requirements or architecture may not match the new design, suggesting `/agent-os:rollback` if they must be revised. Nothing is rolled back automatically.

## Comment Findings

Only with the `comment` permission. Runs after an approved QA, after the post-QA commands and the tracker writes. Also from `/agent-os:design-source comment` for the active or last finished task.

1. Candidates: the issues in `stages.qa.issues` whose `location` starts with `design:` and whose ID has no `sync_log` entry with `operation: "comment"` and `result` `posted` or `declined by user`.
2. None: stop, silently.
3. Draft one comment per issue, in `output_language`: the issue's description, its severity, and the task (`TASK-NNN`, plus the tracker key when linked). No code, no file paths, no secrets.
4. Gate: show every draft with its frame name and link, and AskUserQuestion: "Sí, comentar todas" / "Elegir cuáles" (then ask which, by issue ID) / "No".
5. On approval, for each chosen comment: post it on the node (`node_id` of the location, file of that frame), then log `{ at, event: "comment_findings", operation: "comment", target: <issue id>, result: "posted" | <error> }`. A failure also goes to `sync_errors`, and the next ones are still posted. Declined ones are logged with `result: "declined by user"`.

## Code Connect

Only with the `code_connect` permission. Runs after an approved QA, after Comment Findings. Also from `/agent-os:design-source code-connect`.

1. Candidates: the components listed in the brief that have no Code Connect mapping, matched by name to a component file in `task.files_modified` of the frontend repo. Only an unambiguous match is a candidate; never guess.
2. None: stop, silently.
3. Gate: a table `Componente Figma | node_id | Componente en código | Archivo`, and AskUserQuestion "Sí, mapear todos" / "Elegir cuáles" / "No".
4. On approval, write each mapping and log `{ event: "code_connect", operation: "code_connect", target: <node_id>, result }`. Failures go to `sync_errors`.

## Reporting

- After Discovery: the Snapshot line, and any error in one line.
- In the completion report: a `### 🎨 Diseño` table (`Cuándo | Operación | Destino | Resultado`) from `task.design.sync_log`, plus the open `sync_errors`.
