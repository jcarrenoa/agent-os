---
description: Initialize agent-os in the current project. Run once per repository to configure the tech stack, project type, and preferences.
argument-hint: [--reconfigure]
allowed-tools: [Bash, Read, Write, Glob, Grep, AskUserQuestion]
---

# Agent OS Setup

You are initializing the agent-os framework for this project.

## Pre-Check

1. Check if `.agent-os/config.json` exists.
   - If it **exists** and arguments do NOT include `--reconfigure`: tell the user "Agent OS is already configured for this project. Run `/agent-os:setup --reconfigure` to update the configuration." Then show the current config and stop.
   - If it **exists** and `--reconfigure` is in arguments: tell the user you will update the configuration, then continue.
   - If it **does not exist**: continue with fresh setup.

## Step 1: Automatic Repository Discovery

Follow the repo-discovery protocol to inspect the project.

Run these commands:
```bash
git status 2>/dev/null || echo "Not a git repository"
git log --oneline -5 2>/dev/null || echo "No commits"
git branch --show-current 2>/dev/null || echo "No branch"
```

Use Glob to check for: `package.json`, `pyproject.toml`, `go.mod`, `pom.xml`, `*.csproj`, `pubspec.yaml`, `Gemfile`, `composer.json`, `bun.lockb`, `pnpm-lock.yaml`, `yarn.lock`, `uv.lock`, `poetry.lock`.

Read each file that exists to detect the tech stack.

Also read: `CLAUDE.md`, `README.md` (first 80 lines each, if they exist).

Detect:
- Project name (from package.json `name`, pyproject.toml `name`, directory name as fallback)
- Frontend technology
- Backend technology  
- Mobile technology
- Database
- Package manager
- Test command (from package.json scripts, Makefile, pyproject.toml)
- Dev server command
- Build command

## Step 2: Present Detection Results and Collect Configuration

Present your findings clearly:

```
─────────────────────────────────────────
Agent OS — Project Detection
─────────────────────────────────────────
Project name:   [name]
Project type:   [new|existing|legacy]

Frontend:       [detected technology or "not detected"]
Backend:        [detected technology or "not detected"]
Mobile:         [detected technology or "not detected"]
Database:       [detected technology or "not detected"]
Package mgr:    [detected or "not detected"]

Test command:   [command or "not found"]
Dev command:    [command or "not found"]
Build command:  [command or "not found"]
─────────────────────────────────────────
```

Then use the **AskUserQuestion tool** to collect all configuration in one step. Present these 4 questions simultaneously:

1. Header: "Stack" — "Is the detected stack correct?"
   - "Yes, everything is correct" (description: "Proceed with the detected values")
   - "I need to correct something" (description: "I'll describe the corrections after this")

2. Header: "Team size" — "What is your team size?"
   - "Solo — just me" (description: "Lighter approval flow, no review gates")
   - "Small team (2–5)" (description: "Standard collaboration setup")
   - "Large team (6+)" (description: "Full review gates enabled")

3. Header: "Git tracking" — "Track .agent-os/ in git?"
   - "Yes — track in git (Recommended)" (description: "Full history and team collaboration")
   - "No — keep it local only" (description: "State stays on this machine only")

4. Header: "Language" — "Preferred language for agent responses?"
   - "English" (description: "All agents respond in English")
   - "Español" (description: "Todos los agentes responden en español")

**Wait for the user's selections before continuing.**

## Step 3: Apply Corrections (if needed)

If the user selected "I need to correct something" for the stack question:
- Ask them to describe what needs to change (single follow-up message).
- Apply the corrections.
- Show the corrected stack inline before proceeding.

Otherwise skip directly to Step 4.

## Step 4: Create `.agent-os/` Structure

After confirmation, create these files and directories:

### `.agent-os/config.json`

```json
{
  "project_id": "[slug from project name]",
  "project_name": "[name]",
  "created_at": "[ISO timestamp]",
  "updated_at": "[ISO timestamp]",
  "description": "[from README or empty string]",
  "project_type": "[new|existing|legacy]",
  "tech_stack": {
    "frontend": "[value]",
    "backend": "[value]",
    "mobile": "[value]",
    "database": "[value]",
    "package_manager": "[value]",
    "language": {
      "frontend": "[typescript|javascript|dart|etc]",
      "backend": "[python|typescript|java|go|etc]"
    },
    "test_command": "[value or null]",
    "dev_command": "[value or null]",
    "build_command": "[value or null]",
    "lint_command": "[value or null]"
  },
  "team": {
    "type": "[solo|small|large]",
    "review_required": [true if large, false otherwise]
  },
  "preferences": {
    "track_state_in_git": [true if user said yes],
    "require_architecture_approval": true,
    "require_requirements_approval": true,
    "output_language": "[en or es, default en]"
  }
}
```

### `.agent-os/state.json`

```json
{
  "current_stage": "idle",
  "stage_status": "not_started",
  "active_task_id": null,
  "last_updated": "[ISO timestamp]",
  "stages_completed": [],
  "pending_approval": null,
  "blocker": null
}
```

Create empty directories:
- `.agent-os/tasks/`
- `.agent-os/decisions/`

Create a placeholder `.agent-os/tasks/.gitkeep` and `.agent-os/decisions/.gitkeep`.

## Step 5: Git Configuration

If the user chose to track `.agent-os/` in git:
- Do NOT add anything to `.gitignore`.
- Inform the user: "`.agent-os/` will be tracked in git. Commit it as part of your normal workflow."

If the user chose NOT to track it:
- Check if `.gitignore` exists.
- If yes, append `.agent-os/` to it.
- If no, create `.gitignore` with `.agent-os/` as the first entry.

## Step 6: Confirm Setup Complete

Output:

```
─────────────────────────────────────────
Agent OS — Setup Complete
─────────────────────────────────────────
Project: [name]
Stack:   [frontend] + [backend]

Next steps:
  /agent-os:start "describe your first task"   ← Full 5-stage pipeline
  /agent-os:discover "describe your task"       ← Start with Discovery only
  /agent-os:status                              ← View current state
─────────────────────────────────────────
```
