---
description: Design source integration (Figma over MCP). Without arguments shows the configuration and the active task's design. Subcommands - setup (connect or change it), permissions (choose what agent-os may do - comment, code_connect), link (add Figma links to the active task and take its snapshot), refresh (take the snapshot again), comment (comment the QA design findings), code-connect (map Figma components to code), off (disconnect).
argument-hint: "[setup | permissions | link <url>... | refresh | comment [TASK-NNN] | code-connect [TASK-NNN] | off]"
allowed-tools: [Bash, Read, Write, Glob, Grep, AskUserQuestion]
---

# Agent OS — Design Source

Write everything in `preferences.output_language`. Before any design-source operation, read `${CLAUDE_PLUGIN_ROOT}/skills/design-sync/SKILL.md` and follow it. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

## Pre-Check

1. Check `.agent-os/config.json`. If missing: tell the user to run `/agent-os:setup` and stop.
2. Read `config.json` and `.agent-os/state.json`, and the active task file if there is one.
3. If `config.design_source` is missing and the subcommand is not `setup`: say that no design source is connected, suggest `/agent-os:design-source setup`, and stop.

## No arguments: show

```markdown
## 🎨 Diseño — Figma por MCP (`[mcp.server]`)
| | |
|---|---|
| Permisos | [Leer · Comentar hallazgos · Code Connect: only the granted ones] |
| Frames máximos por tarea | [max_frames, default 10] |
| Tarea activa | [TASK-NNN → N frames, brief path, snapshot_at · or "sin diseño" · or "sin tarea activa"] |
| Cambió después de Discovery | [changed_after · omit when not set] |
| Errores pendientes | [N · /agent-os:design-source refresh · omit when 0] |
```

## setup

Run **Step 2.4: Design Source** of `/agent-os:setup` (same questions and checks), then write only the `design_source` key of `config.json` (read it first; keep every other key) and set `updated_at`.

## permissions

Ask the **Permissions** question of `/agent-os:setup` Step 2.4, marking the current ones with "(actual)". Read `config.json` first, write only `design_source.permissions`, set `updated_at`, and show the new permissions in one line.

## link

Needs an active task (otherwise say so and stop). Parse the URLs of the argument (`design-sync` → Links); with none, ask for them as in Links step 2. Add the new ones to `task.design.links` without duplicates, then run **Snapshot** for all the task's links. Past Discovery, Snapshot's **Refresh after Discovery** applies.

## refresh

Needs an active task with `task.design.links` (otherwise say so and stop). Run **Snapshot** again for its links, replacing the files under `.agent-os/designs/TASK-NNN/`, and remove the snapshot errors from `task.design.sync_errors` when it succeeds. Past Discovery, **Refresh after Discovery** applies.

## comment

For `TASK-NNN`, or the active task, or else the last finished one: run **Comment Findings** of the skill. Without the `comment` permission, say so (Permissions) and stop.

## code-connect

Same task choice as `comment`: run **Code Connect** of the skill. Without the `code_connect` permission, say so and stop.

## off

Ask for confirmation with AskUserQuestion ("Sí, desconectar" / "No"). On yes: remove the `design_source` key from `config.json`. Task files and `.agent-os/designs/` are kept, and the agents keep receiving the briefs of tasks that already have one. Nothing is written to the design source.

## Rules

- This command changes `config.design_source` only in `setup`, `permissions` and `off`, after the user's answers.
- It never writes to the design source except through the `design-sync` skill (gated comments and Code Connect mappings).
