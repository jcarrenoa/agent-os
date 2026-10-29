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

Also detect workspace structure:
- Use Glob with pattern `*/.git` to find sub-repos (directories containing their own `.git`).
- Use Glob with patterns `backend/package.json`, `api/package.json`, `server/package.json`, `frontend/package.json`, `client/package.json`, `web/package.json`, `backend/pyproject.toml`, `api/pyproject.toml`, `server/pyproject.toml` to find sub-project directories.
- Record: `sub_repos_found` (list of sub-directories with `.git`), `sub_project_dirs` (directories with project files detected above).

## Step 1.5: Determine Workspace Structure

Using the workspace detection results from Step 1, suggest a workspace type:
- If any sub-repos were found (e.g., `backend/.git`): suggest `multi-repo`
- Else if current directory is a git repo AND backend/frontend sub-project dirs were found: suggest `monorepo`
- Else: suggest `single`

Ask:

```
AskUserQuestion:
  question: "¿Cómo está estructurado este proyecto?"
  header: "Estructura"
  multiSelect: false
  options:
    - "Proyecto único" (add "(Recomendado)" if single is suggested)
      description: "Este directorio es el proyecto completo. Un repo, un stack."
    - "Monorepo" (add "(Recomendado)" if monorepo is suggested)
      description: "Este repo git raíz contiene back y front como subcarpetas del mismo repo."
    - "Multi-repo" (add "(Recomendado)" if multi-repo is suggested)
      description: "El back y el front son repos git separados dentro de este directorio. La raíz no es un repo git."
```

**After workspace type is selected:**

**If `single`:**
- If BOTH frontend AND backend were detected: use AskUserQuestion to ask:
  ```
  question: "¿Qué parte del stack activas para este proyecto?"
  header: "Alcance"
  options:
    - "Fullstack (Recomendado)" — "Se usarán backend-developer y frontend-developer."
    - "Solo backend" — "Solo backend-developer. No habrá subtareas de frontend."
    - "Solo frontend" — "Solo frontend-developer. No habrá subtareas de backend."
  ```
- If only frontend detected: set `focus = "frontend"` automatically. Inform user.
- If only backend detected: set `focus = "backend"` automatically. Inform user.
- If neither detected clearly: set `focus = "fullstack"`.
- Set `workspace_paths = {}` (not used for single).

**If `monorepo` or `multi-repo`:**
- Set `focus = "fullstack"` automatically (both repos are active).
- Glob all immediate subdirectories for project file indicators to build a candidate list.
- Ask two separate AskUserQuestion calls (≤3 candidates + Other each):
  ```
  question: "¿En qué carpeta está el backend?"
  header: "Backend path"
  options: [up to 3 detected backend candidates] + Other
  ```
  ```
  question: "¿En qué carpeta está el frontend?"
  header: "Frontend path"
  options: [up to 3 detected frontend candidates, excluding chosen backend dir] + Other
  ```
- After paths are confirmed, detect the tech stack within each path by reading their dependency files (package.json, pyproject.toml, etc.). This supplements or replaces the top-level detection for frontend/backend fields.
- Set `workspace_paths = { "backend": "[chosen path]", "frontend": "[chosen path]" }`.

## Step 2: Present Detection Results and Collect Configuration

Present your findings clearly:

```
─────────────────────────────────────────
Agent OS — Project Detection
─────────────────────────────────────────
Project name:   [name]
Project type:   [new|existing|legacy]
Workspace:      [single|monorepo|multi-repo]
Focus:          [fullstack|backend|frontend]
Backend path:   [path or "—" if single]
Frontend path:  [path or "—" if single]

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

Then use the **AskUserQuestion tool** to collect configuration. For `multi-repo` projects, present **3 questions** (skip question 3 below — the workspace root is not a git repo, so `.agent-os/` cannot be committed from there). For `single` and `monorepo`, present all **4 questions** simultaneously:

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

## Step 2.5: Configure Extensions (optional)

Run this step after receiving the user's answers from Step 2 but before writing any files.

1. Glob for external skills:
   - `.claude/skills/*/SKILL.md` (project-level)
   - `~/.claude/skills/*/SKILL.md` (user-level, if accessible)
   - For `monorepo` / `multi-repo`: also `<backend path>/.claude/skills/*/SKILL.md` and `<frontend path>/.claude/skills/*/SKILL.md` (the paths chosen in Step 1.5)
   
2. If **no external skills, agents, or commands found AND no user defaults exist**: skip to Step 3. Do not write an `extensions` key in config.json.

3. If anything is available, ask:
   ```
   AskUserQuestion:
     question: "¿Quieres conectar skills o docs al pipeline ahora?"
     header: "Extensiones"
     options:
       - "Por agente (Recomendado)"
       - "Por etapa y rol"
       - "Omitir por ahora"
     multiSelect: false
   ```

4. If user chose "Por agente" or "Por etapa y rol": follow the `skills-menu` skill, starting directly from **Step B** (skip Step A — the action is already determined). After the menu finishes, continue to Step 3.

5. If "Omitir por ahora": skip to Step 3. Do not write an `extensions` key.

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
  "workspace_type": "[single|monorepo|multi-repo]",
  "focus": "[fullstack|backend|frontend]",
  "workspace_paths": {
    "backend": "[relative path or omit if single]",
    "frontend": "[relative path or omit if single]"
  },
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

**If `workspace_type = "multi-repo"`:**
- Skip the git tracking question — it was not asked.
- Inform the user: "`.agent-os/` is stored in the workspace root. Since the root is not a git repo, it will not be tracked in git. To version it, you can copy it into one of your sub-repos manually."

**If `workspace_type = "single"` or `"monorepo"`:**

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
Project:   [name]
Workspace: [single|monorepo|multi-repo]
Focus:     [fullstack|backend|frontend]
Stack:     [frontend] + [backend]
[Backend path: [path]]   ← only shown if monorepo or multi-repo
[Frontend path: [path]]  ← only shown if monorepo or multi-repo

Next steps:
  /agent-os:start "describe your first task"   ← Full 5-stage pipeline
  /agent-os:discover "describe your task"       ← Start with Discovery only
  /agent-os:status                              ← View current state
─────────────────────────────────────────
```
