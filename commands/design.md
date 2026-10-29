---
description: Run Stage 3 (Design & Architecture) for the active task. Designs components, data models, API contracts, and records ADRs. Always requires human approval before development begins.
allowed-tools: [Read, Write, Glob, Grep]
---

# Agent OS — Stage 3: Design & Architecture

You are running Stage 3 (Design & Architecture) of the agent-os pipeline.

## Pre-Check

1. Read `.agent-os/config.json` and `.agent-os/state.json`.
2. If `current_stage != "design"`:
   - If earlier: tell the user which prior stages need to be completed.
   - If later: tell the user design is already done.
3. Read `.agent-os/tasks/[active_task_id].json`.
4. Verify both `stages.discovery.status` and `stages.planning.status` are `"done"`. If not: stop.

## Run Stage 3

Read `${CLAUDE_PLUGIN_ROOT}/workflows/03-design-architecture.md` and follow it completely. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

Set `stages.design.status = "in_progress"` and update state.json.

### Resolve Extensions

Follow the `skill-resolution` skill for stage `design`, agent `architect`, using its **Architect: Multiple Roles** rules: only the roles active in `config.focus` are resolved, each with its own mode, and the Extensions block carries a `### Shared` section plus one `### Role: <role>` section per active role. Never combine the roles into a single mode. Show any warnings to the user.

Spawn an `architect` agent. Pass it:
- The full task file (requirements + subtasks)
- The project config
- The full extension resolution block under the heading `## Extensions`

After the agent completes:
1. The architect will have created ADR files in `.agent-os/decisions/`.
2. Update `task.decisions` with the list of DEC-NNN IDs created.
3. Set `stages.design.status = "awaiting_approval"`, `completed_at = now`.
4. Set `stages.design.output_summary` with the architecture summary.
5. Update state.json to `stage_status = "awaiting_approval"` with `pending_approval` filled.

## Request Approval — Architecture

This gate is always required. Follow the approval-gates skill.

Present the architecture summary, component list, data model changes, and ADRs to the user.

```
─────────────────────────────────────────
APPROVAL REQUIRED: Architecture
─────────────────────────────────────────
[Architecture summary]

Components: [N new/modified]
Data models: [changes description]
ADRs recorded: [DEC-NNN: title, ...]

This approval is required before development begins.

Run `/agent-os:approve` to proceed to Development,
or describe changes needed.
─────────────────────────────────────────
```

**Stop and wait.**

When approved: record approval, advance to `development/not_started`, suggest `/agent-os:develop`.
