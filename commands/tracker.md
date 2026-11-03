---
description: Issue tracker integration (Jira or Plane over MCP). Without arguments shows the configuration and the active task's link. Subcommands - setup (connect or change the tracker), map (re-analyse the board columns and confirm the route of action), pick (choose an issue to start a task from), sync (retry failed operations and pending writes, also of finished tasks), owner (the name that closes the commits comments), off (disconnect).
argument-hint: "[setup | map | pick | sync [TASK-NNN] | owner [name] | off]"
---

# Agent OS — Tracker

Write everything in `preferences.output_language`. Before any tracker operation, read `${CLAUDE_PLUGIN_ROOT}/skills/tracker-sync/SKILL.md` and follow it. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

## Pre-Check

1. Check `.agent-os/config.json`. If missing: tell the user to run `/agent-os:setup` and stop.
2. Read `config.json` and `.agent-os/state.json`, and the active task file if there is one.
3. If `config.tracker` is missing and the subcommand is not `setup`: say that no tracker is connected, suggest `/agent-os:tracker setup`, and stop.

## No arguments: show

```markdown
## 🔗 Tracker — [Jira | Plane] por MCP (`[mcp.server]`)
| | |
|---|---|
| Acceso | [Solo lectura · Lectura y escritura, also when `access` is missing] |
| Proyecto | [jira.project_key + site_url · or plane workspace_slug / project_identifier] |
| Modo | [pull · create · both] |
| Tarea activa | [TASK-NNN → KEY (url), estado · or "sin issue"] |
| Errores pendientes | [N · /agent-os:tracker sync] |
| Escrituras pendientes | [N] |
| Tareas terminadas por reintentar | [N finished tasks of this tracker with something to retry (`tracker-sync` → Retry) · /agent-os:tracker sync · omit when 0] |
| Encargado | [display_name of ~/.agent-os/profile.json · or "cuenta del tracker, o se pregunta al comentar"] |

### Ruta de acción
| Evento | Cuándo | Columna |
|---|---|---|
| intake | pick busca aquí | [status or —] |
| start | empieza discovery | ... |
| development | empieza desarrollo | ... |
| qa | empieza QA | ... |
| done_pipeline | QA aprueba | ... |
| blocked | etapa bloqueada | ... |
```

With `access: "read"`, omit the "Ruta de acción" section and every row about writes (errors, pending writes, finished tasks to retry, owner).

## setup

Run **Step 2.2: Tracker** of `/agent-os:setup` (same questions and checks), then write only the `tracker` key of `config.json` (read it first; keep every other key) and set `updated_at`. The name from that step's owner question goes to `~/.agent-os/profile.json`, never to `config.json`.

## map

Run **Board Analysis** of the `tracker-sync` skill and write the new `tracker.status_route`. Nothing else changes. With `access: "read"`, follow **Read-Only Access** in the skill instead.

## pick

Only when `tracker.mode` is `pull` or `both` (otherwise say so and stop). If a task is active (`current_stage` not `idle` and `stage_status` not `done`): say so and stop, as `/agent-os:start` does. Otherwise run **Pick** of the `tracker-sync` skill and then continue exactly as `/agent-os:start <KEY>`.

## sync

Works with or without an active task. Read the task files in `.agent-os/tasks/` and run **Retry** of the `tracker-sync` skill for the tasks it covers (`sync TASK-NNN`: only that task). Show the result, or say in one line that nothing is pending. With `access: "read"`, **Read-Only Access** in the skill limits what runs.

## owner

With a name as argument: save it (**Owner Name → Configuring it** in the `tracker-sync` skill, without asking). Without one: ask that section's question and save the answer. Nothing is written to the tracker or to `config.json`.

## off

Ask for confirmation with AskUserQuestion ("Sí, desconectar" / "No"). On yes: remove the `tracker` key from `config.json`. Task files keep their `external` data. Nothing is written to the tracker.

## Rules

- This command changes `config.tracker` only in `setup`, `map` and `off`, after the user's answers, and `~/.agent-os/profile.json` only in `setup` and `owner`.
- It never writes to the tracker except through the `tracker-sync` skill (transitions of the route, and gated creates and comments).
