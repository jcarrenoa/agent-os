---
description: Run Stage 1 (Discovery & Requirements) for a task. Captures and structures requirements before any planning or implementation. Requires approval before planning can begin.
argument-hint: "<task description | issue key or URL>"
allowed-tools: [Read, Write, Glob, Grep, Bash, AskUserQuestion]
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

If creating a new task: create `TASK-NNN.json`, record its git baseline (Recording the git baseline in the `state-management` skill) and update `state.json`. With `config.tracker` in `pull` or `both` mode, an issue key or URL as the argument fills the task from the issue (`tracker-sync` skill → Pull).

## Run Stage 1

Read `${CLAUDE_PLUGIN_ROOT}/workflows/01-discovery.md` and follow it completely. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.

Set `stages.discovery.status = "in_progress"` in the task file before spawning the agent.
Tracker (only with `config.tracker`): fire the `start` route event (`tracker-sync` skill).
Set `state.json.stage_status = "in_progress"`.

### Resolve Extensions

Follow the `skill-resolution` skill for stage `discovery`, role `—`, agent `requirements-analyst`. Show any warnings to the user.

Spawn a `requirements-analyst` agent. Pass it:
- The task description (verbatim)
- The project config path
- The task file path
- The full extension resolution block under the heading `## Extensions`

Build the agent prompt with Step 5 of the `skill-resolution` skill (file paths and references only, never pasted content). After the agent returns, verify its `EXTENSIONS APPLIED` section with Step 6 of that skill.

If the agent returned a `QUESTIONS` block: follow the **Clarifying Questions Protocol** of the workflow (AskUserQuestion, or the approver in unattended mode), record the answers in `open_questions`, and re-run the agent with the `Answers:` line. Never answer the questions yourself or let the agent assume them.

When it returns the requirements:
1. Record its output in the task file as described in the workflow's State Updates (step 2): `requirements_format`, `requirements`, `user_stories` (with user stories), `assumptions`, `out_of_scope`.
2. Set `stages.discovery.output_summary` to a 1-2 sentence summary.

## Approver Review

Only when `config.approver.enabled` and `"requirements"` is in `config.approver.gates`: follow **Approver Review** in the `approval-gates` skill. In autonomous or unattended mode, an approver approval (`approved_by = "approver"`): tell the user, advance state to `planning / not_started`, suggest `/agent-os:plan` or `/agent-os:start`, and stop. If unattended mode blocked the stage: show the blocker and stop. Otherwise continue to Request Approval.

## Request Approval

1. Set `stages.discovery.status = "awaiting_approval"`, `completed_at = now`.
2. Update `state.json` to `stage_status = "awaiting_approval"` with `pending_approval` filled (including `approver_verdict`).

Follow the approval-gates skill to present the gate.

Show the requirements (or user stories) to the user, with the approver verdict block if the approver reviewed them, and ask for approval.

**Stop and wait.**

When approved (user says "approve", or runs `/agent-os:approve` or `/agent-os:review`):
- Record the approval with `approved_by = "human"` (How to Record an Approval in the approval-gates skill).
- Advance state to `planning / not_started`.
- Suggest next step: `/agent-os:plan` or `/agent-os:start`.

When the user requests changes: follow How to Handle Rejection or Change Requests in the approval-gates skill.
