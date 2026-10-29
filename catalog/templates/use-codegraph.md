---
name: use-codegraph
description: agent-os pipeline wrapper for the CodeGraph CLI, written by /agent-os:extensions. Only applies when it is listed in an agent-os agent's Extensions block; it tells the agent to query the local code graph before searching or reading code.
version: 1.0.0
---

# Use CodeGraph

Installed by agent-os from its recommended catalog (`@colbymchenry/codegraph`, pinned version). CodeGraph keeps a local index of symbols, calls and imports in `.codegraph/`.

## How to apply it

This needs the Bash tool. If you do not have it, report this skill as `not applicable: no Bash`.

1. Your code directory is your `working_directory` if you were given one, otherwise the workspace root.
2. If that directory contains `.codegraph/`:
   - Run `codegraph sync` once at the start, from that directory.
   - Before searching or reading code, ask the graph first: `codegraph explore "<what you need to understand>"`. It returns the relevant symbols' source, their call paths and their impact in one call.
   - Read only the files its answer points to, or the files you are about to edit.
   - `codegraph impact <symbol>` and `codegraph callers <symbol>` are useful before changing a symbol.
3. If there is no `.codegraph/`, or the `codegraph` command is missing or fails: continue with Read, Grep and Glob as usual, and say so in your `EXTENSIONS APPLIED` line.

## Limits

- Never run `codegraph install`, `codegraph init`, `codegraph uninit`, `codegraph upgrade` or `codegraph telemetry`. Installing and configuring CodeGraph is done only by `/agent-os:extensions`, with the user's approval.
- `.agent-os/` files, `CLAUDE.md` files and the paths in your Extensions block are still read with the Read tool, as your agent instructions say.
