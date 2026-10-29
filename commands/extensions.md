---
description: Install, update, uninstall, or check the recommended extensions from the agent-os catalog (e.g. CodeGraph, The AntiSlop). Shows the exact commands and asks for one explicit approval before changing anything. Only catalog entries can be installed.
argument-hint: "[status | install <id>... | uninstall <id>... | update <id>...]"
allowed-tools: [Bash, PowerShell, Read, Write, Glob, AskUserQuestion]
---

# Agent OS — Recommended Extensions

Arguments: $ARGUMENTS

Follow the `extension-installer` skill for every subcommand. The catalog is `${CLAUDE_PLUGIN_ROOT}/catalog/recommended.json`; if that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

`.agent-os/config.json` is optional here: it is only used to find the repositories for per-repository steps (`workspace_type`, `workspace_paths`, `focus`) and the output language. Without it, per-repository steps run in the current directory.

## Subcommands

| Usage | What it does |
|-------|--------------|
| `/agent-os:extensions` or `/agent-os:extensions status` | Read-only. For each catalog entry: pinned version, installed version (if any), state (not installed / up to date / outdated / present, not managed / unavailable), and the agents it is assigned to in `~/.agent-os/defaults.json`. |
| `/agent-os:extensions install <id>...` | Install the given entries (installer Steps 2–6). Entries that are already up to date only get their missing assignments. |
| `/agent-os:extensions uninstall <id>...` | Uninstall entries that agent-os installed (installer Step 7). Removes only the assignments the installer added. |
| `/agent-os:extensions update <id>...` | Re-run the install recipe for outdated entries, to the version pinned in the catalog only. Up-to-date entries are left alone. |

`install` with no id: list the catalog entries that are not installed and ask which to install (AskUserQuestion, multiSelect, plus a "Ninguna" / "None" option), then continue as `install <selected ids>`.

## Rules

- Unknown ids are refused: "`<id>` is not in the agent-os catalog. Only catalog entries can be installed." Never install a package, repository or skill that is not in the catalog, and never a version other than the pinned one.
- `install`, `uninstall` and `update` always show the plan and wait for the single approval of installer Step 4. Without "Sí, ejecutar" nothing changes.
- Finish with the installer's Step 8 report.
