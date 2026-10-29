---
description: Run Stage 1 (Discovery & Requirements) for a task. Captures and structures requirements before any planning or implementation. Requires approval before planning can begin.
argument-hint: "<task description>"
allowed-tools: [Read, Write, Glob, Grep, Bash]
---

# Agent OS — Stage 1: Discovery

You are running Stage 1 (Discovery & Requirements) of the agent-os pipeline.

Task: $ARGUMENTS

## Pre-Check

1. Read `.agent-os/config.json`. If missing: tell user to run `/agent-os:setup` first.
2. Read `.agent-os/state.json`.
3. Determine context:
   - If `current_stage = "idle"` or `current_stage = "discovery"` with `stage_status = "not_started"`: create a new task (or use the active one).
   - If `current_stage = "discovery"` with `stage_status = "awaiting_approval"`: the stage is already done and awaiting approval. Show existing requirements and remind the user to run `/agent-os:approve`.
   - If `current_stage` is any other stage: tell the user which stage is active and offer to rollback.

If creating a new task: create `TASK-NNN.json` and update `state.json`.

## Run Stage 1

Read `${CLAUDE_PLUGIN_ROOT}/workflows/01-discovery.md` and follow it completely. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

Set `stages.discovery.status = "in_progress"` in the task file before spawning the agent.
Set `state.json.stage_status = "in_progress"`.

### Resolve Extensions

Follow the `skill-resolution` skill for stage `discovery`, role `—`, agent `requirements-analyst`. Show any warnings to the user.

Spawn a `requirements-analyst` agent. Pass it:
- The task description (verbatim)
- The project config path
- The task file path
- The full extension resolution block under the heading `## Extensions`

Build the agent prompt with Step 5 of the `skill-resolution` skill (file paths and references only, never pasted content). After the agent returns, verify its `EXTENSIONS APPLIED` section with Step 6 of that skill.

After the agent completes:
1. Update `task.requirements` from the agent's output.
2. Set `stages.discovery.status = "awaiting_approval"`, `completed_at = now`.
3. Set `stages.discovery.output_summary` to a 1-2 sentence summary.
4. Update `state.json` to `stage_status = "awaiting_approval"` with `pending_approval` filled.

## Request Approval

Follow the approval-gates skill to present the gate.

Show requirements to the user and ask for approval.

**Stop and wait.**

When approved (user says "approve" or runs `/agent-os:approve`):
- Record approval in task file.
- Advance state to `planning / not_started`.
- Suggest next step: `/agent-os:plan` or `/agent-os:start`.
