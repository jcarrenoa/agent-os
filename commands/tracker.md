---
description: Issue tracker integration (Jira or Plane over MCP). Without arguments shows the configuration and the active task's link. Subcommands - setup (connect or change the tracker), map (re-analyse the board columns and confirm the route of action), pick (choose an issue to start a task from), sync (retry failed operations and pending writes), off (disconnect).
argument-hint: "[setup | map | pick | sync | off]"
---

# Agent OS — Tracker

Write everything in `preferences.output_language`. Follow the `tracker-sync` skill for every tracker operation.

## Pre-Check

1. Check `.agent-os/config.json`. If missing: tell the user to run `/agent-os:setup` and stop.
2. Read `config.json` and `.agent-os/state.json`, and the active task file if there is one.
3. If `config.tracker` is missing and the subcommand is not `setup`: say that no tracker is connected, suggest `/agent-os:tracker setup`, and stop.

## No arguments: show

```markdown
## 🔗 Tracker — [Jira | Plane] por MCP (`[mcp.server]`)
| | |
|---|---|
| Proyecto | [jira.project_key + site_url · or plane workspace_slug / project_identifier] |
| Modo | [pull · create · both] |
| Tarea activa | [TASK-NNN → KEY (url), estado · or "sin issue"] |
| Errores pendientes | [N · /agent-os:tracker sync] |
| Escrituras pendientes | [N] |

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

## setup

Run **Step 2.2: Tracker** of `/agent-os:setup` (same questions and checks), then write only the `tracker` key of `config.json` (read it first; keep every other key) and set `updated_at`.

## map

Run **Board Analysis** of the `tracker-sync` skill and write the new `tracker.status_route`. Nothing else changes.

## pick

Only when `tracker.mode` is `pull` or `both` (otherwise say so and stop). If a task is active (`current_stage` not `idle` and `stage_status` not `done`): say so and stop, as `/agent-os:start` does. Otherwise run **Pick** of the `tracker-sync` skill and then continue exactly as `/agent-os:start <KEY>`.

## sync

Requires an active task with `task.external` or `pending_writes`. Run **Retry** of the `tracker-sync` skill and show the result.

## off

Ask for confirmation with AskUserQuestion ("Sí, desconectar" / "No"). On yes: remove the `tracker` key from `config.json`. Task files keep their `external` data. Nothing is written to the tracker.

## Rules

- This command changes `config.tracker` only in `setup`, `map` and `off`, after the user's answers.
- It never writes to the tracker except through the `tracker-sync` skill (transitions of the route, and gated creates and comments).
