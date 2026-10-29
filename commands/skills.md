---
description: Manage skill, doc, agent, and command extensions for the agent-os pipeline. With no arguments opens an interactive menu. With arguments: list, add, remove, mode — for both stage/role and per-agent targets. Use --global to write to ~/.agent-os/defaults.json instead of the project config.
argument-hint: "[list [--agent <agent> | <stage>[.<role>]]] [add|remove [--agent <agent>] <name|path>] [mode [--agent <agent>] <stage>.<role> extend|replace] [--global] [--auto]"
allowed-tools: [Read, Write, Glob, AskUserQuestion]
---

# Agent OS — Skills & Extensions

Arguments: $ARGUMENTS

## Pre-Check

1. Read `.agent-os/config.json`. If missing and the subcommand is not `list` or the menu: tell the user to run `/agent-os:setup` first and stop.
2. Parse `$ARGUMENTS` to determine the mode:
   - **Empty arguments**: interactive menu mode → follow the `skills-menu` skill with all 4 options (Por agente, Por etapa y rol, Ver configuración actual, Quitar asignaciones). Stop after the skill completes.
   - **Recognized subcommand** (`list`, `add`, `remove`, `mode`): non-interactive mode, continue below.
   - **Unrecognized**: show usage and stop.

---

## Usage (non-interactive)

```
/agent-os:skills                                       ← interactive menu
/agent-os:skills list                                  ← all extensions, combined
/agent-os:skills list <stage>[.<role>]                 ← filtered by stage/role
/agent-os:skills list --agent <agent>                  ← filtered by agent

/agent-os:skills add <stage>[.<role>] <name|path>      ← add to stage/role
/agent-os:skills add --agent <agent> <name|path>       ← add to specific agent
/agent-os:skills add <...> --global                    ← write to ~/.agent-os/defaults.json
/agent-os:skills add qa <command> --auto               ← post-QA command that runs WITHOUT approval

/agent-os:skills remove <stage>[.<role>] <name|path>
/agent-os:skills remove --agent <agent> <name|path>
/agent-os:skills remove <...> --global

/agent-os:skills mode <stage>.<role> extend|replace
/agent-os:skills mode --agent <agent> extend|replace
/agent-os:skills mode <...> --global

Stages: discovery, planning, design, development, qa
Roles:  backend, frontend  (required for design and development)
Agents: requirements-analyst, planner, architect, backend-developer, frontend-developer, qa-engineer
```

---

## Subcommand: list

Show the combined extension configuration, indicating provenance for each entry.

### Steps

1. Read `~/.agent-os/defaults.json` (user level, if it exists).
2. Read `.agent-os/config.json` → `extensions` (project level, if it exists).
3. If a filter is provided (`--agent <agent>` or `<stage>[.<role>]`):
   - Follow the `skill-resolution` skill for that destination (resolve file paths, check existence).
   - Show what that specific agent or stage would receive on the next pipeline run.
4. Without filter: show the full combined view.

#### Full view format

```
─────────────────────────────────────────
Agent OS — Extensions
─────────────────────────────────────────
Context (todas las etapas):
  [path]   [project|user]   [✓|✗]

Por etapa:
  discovery  skills: [name ...]  docs: [path ...]
  planning   (none)
  design     skills: [...]  backend.mode: extend  frontend.mode: replace
  development  backend: [skills]  frontend: [skills]
  qa         skills: [...]  agents: [...]  commands: [name [gate] | name [auto] ...]

Por agente:
  architect          skills: [...]   mode: extend
  backend-developer  skills: [...]   mode: replace
  qa-engineer        agents: [...]   commands: [...]

Plugin defaults (added unless mode=replace; replace never removes stage/role/agent entries):
  backend-developer → tech-adaptation
  frontend-developer → tech-adaptation
  architect → tech-adaptation
─────────────────────────────────────────
```

#### Filtered view (by agent or stage)

Show the resolved result exactly as the `skill-resolution` skill would produce it, including provenance tags (tier and level) and any warnings.

---

## Subcommand: add

Add an entry to extensions. Supports `--agent` and `--global` flags.

### Steps

1. Parse: `add [--agent <agent>] [--global] <name|path>`
   - `--agent <agent>`: agent-level target. Valid agents: `requirements-analyst`, `planner`, `architect`, `backend-developer`, `frontend-developer`, `qa-engineer`.
   - Without `--agent`: first positional arg is `<stage>[.<role>]`.
   - `--global`: write to `~/.agent-os/defaults.json` (create if missing) instead of `.agent-os/config.json`.
   - `--auto`: only valid for post-QA commands. Stores the command as `{ "name": "<command>", "auto": true }` so it runs without an approval gate. Without `--auto`, commands are stored as plain strings and always require approval.
   - `<name|path>`: if starts with `/` or `~` or ends with `.md` → doc (add to `docs[]`). Otherwise → skill/agent/command name (add to `skills[]` for most destinations; for QA, add to `agents[]` if it matches a known agent name, `commands[]` if it matches a command).

2. Read the target file. Merge the new entry into the correct location:
   - `--agent <agent>` → `extensions.agents.<agent>.[docs|skills|agents|commands]`
   - `<stage>[.<role>]` → `extensions.stages.<stage>[.<role>].[docs|skills]`
   - For qa: `extensions.stages.qa.[agents|commands]` or `extensions.agents.qa-engineer.[agents|commands]`

3. Do not add duplicates. For commands, compare by name (string or object `name`); re-adding a command with or without `--auto` replaces its entry.

4. Write the target file.

5. Confirm: "Added `<name>` to `<destination>` in `<file>`."

---

## Subcommand: remove

Remove an entry. Supports `--agent` and `--global` flags.

### Steps

1. Parse same as `add`.
2. Read the target file.
3. Find and remove the entry from the correct list.
4. If not found in the target file: tell the user. If it exists in the other level (project vs user), note where it is.
5. Write the target file.
6. Confirm: "Removed `<name>` from `<destination>` in `<file>`."

---

## Subcommand: mode

Set `mode` for a destination. Supports `--agent` and `--global`.

Valid destinations: `architect`, `backend-developer`, `frontend-developer` (agents), `design`, `design.backend`, `design.frontend`, `development.backend`, `development.frontend` (stages/roles). `mode` is not applicable to discovery, planning, qa, `requirements-analyst`, or `planner`.

### Steps

1. Parse: `mode [--agent <agent> | <stage>.<role>] [--global] extend|replace`
2. Validate destination supports mode (see list above).
3. Read the target file.
4. Set `extensions.agents.<agent>.mode` or `extensions.stages.<stage>.<role>.mode`.
5. Write the target file.
6. Confirm: "Set mode to `<value>` for `<destination>` in `<file>`."
   - If `replace`: remind "replace mode only skips the plugin default (tech-adaptation); stage, role and agent skills still apply. Make sure they give the agent enough stack context."
