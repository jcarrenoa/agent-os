---
description: Initialize agent-os in the current project. Run once per repository to configure the tech stack, project type, and preferences.
argument-hint: [--reconfigure]
allowed-tools: [Bash, PowerShell, Read, Write, Glob, Grep, AskUserQuestion]
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

Present your findings as formatted Markdown (the terminal renders it). **Render this block exactly in this structure (headings + tables). Do NOT summarize the detection in a prose sentence.**

`output_language` is not chosen yet at this point: write the labels in the language the user is writing in (the template below shows the Spanish labels; translate them for English).

Rules:
- **Role sections** (Backend, Frontend): one heading + table per role. The heading shows the role's `workspace_paths` value (e.g. `api/`), or `raíz` for `single`.
- **Commands**: in `multi-repo`, and in `monorepo` when each subdirectory has its own commands, put Tests/Dev/Build inside each role table. In `single`, or in a `monorepo` whose commands are defined once at the root, drop those three rows from the role tables and show them once in a **🧰 Comandos del proyecto** section.
- **Focus**: when `focus` excludes a role, keep only its heading with the suffix `(fuera de foco)` and a single line with the detected technology, without a table. This keeps the detection visible without suggesting that agents will work on that role.
- **Mobile / Database**: give each its own section (same table style: Tecnología, plus Lenguaje/Gestor de paquetes where relevant) only if detected. Otherwise list it in a single final line: `No detectado: Mobile, Base de datos`.
- **Missing values**: use `—`. Never write "not detected" or "not found" inside a table.

Template (`fullstack`, `multi-repo`):

```markdown
## 🔍 Agent OS — Detección del proyecto

**Proyecto:** `[name]` · **Tipo:** [new|existing|legacy] · **Workspace:** [single|monorepo|multi-repo] · **Focus:** [fullstack|backend|frontend]

### ⚙️ Backend — `[workspace_paths.backend]/`
| | |
|---|---|
| Tecnología | [technology or —] |
| Lenguaje | [language or —] |
| Gestor de paquetes | [package manager or —] |
| Tests | `[test command]` or — |
| Dev | `[dev command]` or — |
| Build | `[build command]` or — |

### 🎨 Frontend — `[workspace_paths.frontend]/`
| | |
|---|---|
| Tecnología | [technology or —] |
| Lenguaje | [language or —] |
| Gestor de paquetes | [package manager or —] |
| Tests | `[test command]` or — |
| Dev | `[dev command]` or — |
| Build | `[build command]` or — |

No detectado: Mobile, Base de datos
```

Shared commands section (`single`, or `monorepo` with root-level commands), placed after the role sections:

```markdown
### 🧰 Comandos del proyecto
| | |
|---|---|
| Tests | `[test command]` or — |
| Dev | `[dev command]` or — |
| Build | `[build command]` or — |
```

Role outside the focus (e.g. `focus = backend`):

```markdown
### 🎨 Frontend — `[path]/` (fuera de foco)
Detectado: [technology or —]
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

## Step 2.1: Review Options

Ask both questions in one AskUserQuestion call, in the language chosen in Step 2. With `--reconfigure`, mark the current values with "(actual)" instead of "(Recomendado)"; for a current `unattended` mode, the confirmation below is still asked.

1. Header: "Approver" — "¿Activar el approver, un agente que revisa los requisitos y el plan antes de sus gates?"
   - "Desactivado (Recomendado)" (description: "Todo funciona como hasta ahora; tú apruebas los gates.")
   - "Advisory" (description: "Revisa requisitos y plan y te muestra su veredicto; tú decides siempre. El plan pasa a tener gate.")
   - "Autonomous" (description: "Si aprueba, avanza sin preguntarte; si pide cambios, se detiene y te lo muestra. La arquitectura siempre la apruebas tú.")
   - "Unattended" (description: "⚠ Nada te pedirá confirmación salvo los commands post-QA marcados [gate]: el approver responde las preguntas, aprueba requisitos, plan y ARQUITECTURA, y reintenta QA. Solo se detiene ante un blocker.")

If the answer is "Unattended", before writing anything show this warning and ask for an explicit confirmation with a second AskUserQuestion (header "Unattended", options "Sí, activar unattended" / "No, elegir otro modo"):

```
⚠ Modo unattended
- El approver (un agente) decide por ti: preguntas aclaratorias, cambios de alcance, requisitos, plan y arquitectura,
  incluidos los casos que normalmente siempre requieren aprobación humana (cambios de modelo de datos, migraciones,
  autenticación, integraciones externas).
- Si QA falla, se relanza desarrollo hasta max_revisions veces; después se continúa con los hallazgos major/minor
  sin resolver, o se detiene si queda un blocker.
- Los commands post-QA marcados [gate] (push, deploy, publicar...) te siguen pidiendo aprobación; los [auto] se ejecutan.
- Todo queda registrado en el task file y en el informe "Decisiones del approver".
```

"No, elegir otro modo" → ask question 1 again without the "Unattended" option.

2. Header: "Requisitos" — "¿En qué formato se escriben los requisitos funcionales?"
   - "REQ (Recomendado)" (description: "Lista REQ-001 [must]: ... como hasta ahora.")
   - "Historias de usuario" (description: "US-001: Como <rol> quiero <capacidad> para <beneficio>, con criterios de aceptación.")

Map the answers:
- "Desactivado" → no `approver` key in config.json (with `--reconfigure`, set `"enabled": false` if the key existed, keeping its other values).
- "Advisory" → `"approver": { "enabled": true, "mode": "advisory", "gates": ["requirements", "planning"], "max_revisions": 0 }`.
- "Autonomous" → `"approver": { "enabled": true, "mode": "autonomous", "gates": ["requirements", "planning"], "max_revisions": 0 }`. With `max_revisions = 0`, a `REQUEST_CHANGES` stops at your gate; raising it lets the producing agent retry with the findings (see the README).
- "Unattended" (confirmed) → `"approver": { "enabled": true, "mode": "unattended", "gates": ["requirements", "planning", "design"], "max_revisions": 2 }`. `config.schema.json` requires `max_revisions ≥ 1` in this mode.
- "REQ" → `preferences.requirements_format = "req"`; "Historias de usuario" → `"user-stories"`.

The architecture gate (`design`) can be given to the approver only in unattended mode: `config.schema.json` rejects it in `gates` for any other mode.

## Step 2.2: Tracker (optional)

Connects Jira or Plane through an MCP server. Writes to the tracker follow the `tracker-sync` skill: moves along the route you confirm here are automatic; creating issues and commenting always ask first.

1. AskUserQuestion, in `output_language`. With `--reconfigure`, mark the current value with "(actual)".
   - Header: "Tracker" — "¿Conectar un gestor de tareas?"
   - "No (Recomendado)" (description: "Sin integración; nada cambia.")
   - "Jira" (description: "Jira Cloud por el servidor MCP de Atlassian.")
   - "Plane" (description: "Plane por su servidor MCP oficial.")

   "No" → no `tracker` key (with `--reconfigure`, remove it if it existed, after confirming). Continue to Step 2.5.
2. **MCP server.** List the MCP tools available in this session and find the servers whose tools match the provider (`tracker-sync` → Tool Discovery). One match: use it. Several: ask which one. None: show the `claude mcp add` command for the provider from the README (Tracker section), tell the user to add it, authenticate with `/mcp`, and run `/agent-os:tracker setup` later. Do not write a `tracker` key; continue to Step 2.5.
3. **Project.**
   - Jira: get the accessible sites (one → use it; several → ask), keep `site_url` and `cloud_id`; list the visible projects and ask for the project (the 4 most relevant as options; the user can type the key). Read its issue types and propose `parent_type` (Story) and `subtask_type` (Sub-task, or the project's sub-task type); confirm in the next question.
   - Plane: list the projects of the workspace and ask; keep `workspace_slug`, `project_id`, `project_identifier` and, for self-hosted instances, `base_url`.
4. AskUserQuestion with:
   - Header "Modo" — "¿Cómo se relacionan las tareas con el tablero?": "Ambos (Recomendado)" (description: "Empezar desde un issue existente o crear uno nuevo") / "Pull" (description: "Solo tomar issues que ya existen") / "Create" (description: "agent-os crea la Story y sus sub-tareas, siempre con tu aprobación").
   - Jira only, header "Tipos" — "Tipos para crear: [parent_type] + [subtask_type]?": "Sí (Recomendado)" / "Cambiar" (then ask the user to type them).
5. **Route of action**: run **Board Analysis** of the `tracker-sync` skill.
6. Keep the result for Step 4: `"tracker": { provider, connection: "mcp", mcp: { server }, mode, jira | plane, pick_assigned_to_me: true, status_route }`. No credentials: the MCP server handles authentication.

If any MCP call fails, say so, write no `tracker` key, suggest `/agent-os:tracker setup`, and continue: the tracker never blocks setup.

## Step 2.3: Git Conventions (optional)

Proposes how `/agent-os:commit` names branches and writes commit subjects. Nothing is fixed in the plugin: the conventions come from the project's own history. The repos are those of `/agent-os:commit` Step 1. Skip this step when none of them is a git repo.

1. For each repo, read-only: `git -C <repo> log --no-merges -30 --format=%s`, `git -C <repo> branch -a --format=%(refname:short)` and `git -C <repo> log --merges -30 --format=%s`. First strip a trailing ` (#NN)` (the PR number GitHub adds on squash merges) from every subject. Infer, using the placeholders of `/agent-os:commit` (Step 1):
   - `commit_pattern`: the shape of the subjects that carry an issue key, with every separator and space copied exactly as the project writes it, never normalized (`VR-91 | feat : Expand contact and group APIs (#12)` → `{key} | {type} : {summary}`; `VR-91 | feat: ...` → `{key} | {type}: {summary}`; `feat(VR-91): ...` → `{type}({key}): {summary}`).
   - `commit_pattern_no_key`: the shape of the subjects without a key (`fix: ...` → `{type}: {summary}`).
   - `commit_types`: the types used, most frequent first. `commit_language`: the language of the summaries.
   - `branch_pattern`: the shape of the feature branches (`feature/VR-91-expand-contact` → `feature/{key}-{slug}`).
   - `base_branch`: the long-lived branch feature branches are born from and merged into. Use the first signal that gives a clear answer:
     1. Merge subjects: `Merge pull request #12 from org/feature/VR-91-... into qa`, `Merge branch 'feature/...' into qa`. Squash merges leave none, so this often finds nothing.
     2. The branch model in the project's `CLAUDE.md` or `AGENTS.md` (workspace root and the repo), then the user's global `~/.claude/CLAUDE.md` and the files it imports, when they describe it ("feature/* branches from qa").
     3. The repo itself: the remote default branch (`git -C <repo> symbolic-ref --short refs/remotes/origin/HEAD`), unless the feature branches diverge from another long-lived branch: for up to 5 `feature/*` branches, compare `git -C <repo> merge-base <feature> <candidate>` across the long-lived branches and take the one most of them diverge from most recently.

     When none of them is clear, leave it `null` and say so.
   - `protected_branches`: the long-lived branches that exist (`main`, `master`, `qa`, `dev`, `develop`, `staging`, and `release/*` when release branches exist).
   - A field without a clear pattern in at least 5 subjects or branches stays empty (`—`): `/agent-os:commit` uses its default.
2. Show the proposal, one column per repo (a single column when they agree), with an example subject for this project:
   ```markdown
   ### 🌿 Convenciones de git
   | | backend | frontend |
   |---|---|---|
   | Rama base | qa | qa |
   | Rama | feature/{key}-{slug} | feature/{key}-{slug} |
   | Commit | {key} \| {type} : {summary} | {key} \| {type} : {summary} |
   | Commit sin key | {type}: {summary} | {type}: {summary} |
   | Tipos | feat, fix, chore | feat, fix, chore |
   | Idioma | en | en |
   | Protegidas | master, qa, dev, release/* | master, qa, dev |
   | Ejemplo | VR-91 \| feat : Expand contact and group APIs | ... |
   ```
3. Ask both questions in one AskUserQuestion call, in `output_language` (with `--reconfigure`, mark the current values with "(actual)"):
   - Header "Git" — "¿Usar estas convenciones para ramas y commits?": "Detectado (Recomendado)" (description: "Guarda la tabla tal cual") / "Editar" (description: "Escribe qué cambiar; te muestro la tabla de nuevo") / "Sin convención" (description: "Ramas feature/{key}-{slug} desde la rama actual y commits {type}: {summary}").
   - Header "Commit" — "¿Hacer un commit local por repo al terminar QA (/agent-os:commit)? Nunca hace push.": "Sí, con aprobación (Recomendado)" (description: "Te pregunta antes de commitear") / "Sí, sin preguntar" (description: "Commitea al terminar QA; sigue sin hacer push") / "No" (description: "Lo lanzas tú con /agent-os:commit cuando quieras").
4. Keep for Step 4:
   - "Detectado" or the edited table → `config.git` with the values the repos share; the fields that differ go to `config.git.repos.<repo path>`. Empty fields are not written. "Sin convención" → no `git` key.
   - "Sí, con aprobación" → `"agent-os:commit"` in `extensions.stages.qa.commands`; "Sí, sin preguntar" → `{ "name": "agent-os:commit", "auto": true }`; "No" → neither. Never add it twice; with `--reconfigure`, replace the existing entry.

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
    "output_language": "[en or es, default en]",
    "requirements_format": "[req | user-stories, from Step 2.1]"
  },
  "approver": "[only if Step 2.1 enabled it: { enabled, mode, gates, max_revisions }]",
  "tracker": "[only if Step 2.2 connected one: { provider, connection, mcp, mode, jira | plane, pick_assigned_to_me, status_route }]",
  "git": "[only if Step 2.3 kept conventions: { base_branch, branch_pattern, commit_pattern, commit_pattern_no_key, commit_types, commit_language, protected_branches, repos? }]",
  "extensions": "[from Step 2.5, plus agent-os:commit in stages.qa.commands if Step 2.3 enabled it]"
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

## Step 5.5: Recommended Extensions (optional)

Offer the extensions of the agent-os catalog. This step is optional and never blocks setup.

1. Follow Steps 1–2 of the `extension-installer` skill (read-only) for every catalog entry in `${CLAUDE_PLUGIN_ROOT}/catalog/recommended.json`.
2. If every entry is already up to date or unavailable: say so in one line and continue to Step 6.
3. Otherwise ask, in `output_language`:
   ```
   AskUserQuestion:
     question: "¿Instalar extensiones recomendadas? Antes de ejecutar nada verás los comandos exactos y tendrás que aprobarlos."
     header: "Extensiones"
     options:
       - "<name> — <one-line description>" (one per entry that is not installed or outdated; at most 3, paginate as in the skills-menu skill if there are more)
       - "Ninguna" — no instalar nada
     multiSelect: true
   ```
4. If nothing is selected, or "Ninguna" is among the selections: install nothing and continue to Step 6.
5. Otherwise follow Steps 3–8 of the `extension-installer` skill for the selected entries. If the user does not approve the commands, nothing is installed; continue to Step 6 either way.

## Step 6: Confirm Setup Complete

Output formatted Markdown in `output_language`, using the same style as the Step 2 detection (headings + one table per role). **Render this block exactly in this structure. Do NOT summarize it in a prose sentence.** Apply the same rules as Step 2 for paths (`raíz` / root for `single`), roles outside the focus, shared commands, undetected Mobile/Database and `—` for missing values, using the values saved in `config.json`.

If Step 5.5 installed or tried to install extensions, add its Step 8 report table as a `### 🧩 Extensiones recomendadas` section before `### ▶️ Próximos pasos`, and add the row `/agent-os:extensions` (estado y gestión de las extensiones) to the next-steps table.

Template (Spanish labels; translate them when `output_language = "en"`):

```markdown
## ✅ Agent OS — Configuración completa

**Proyecto:** `[name]` · **Workspace:** [single|monorepo|multi-repo] · **Focus:** [fullstack|backend|frontend] · **Idioma:** [en|es]

### ⚙️ Backend — `[workspace_paths.backend]/`
| | |
|---|---|
| Tecnología | [tech_stack.backend or —] |
| Lenguaje | [tech_stack.language.backend or —] |
| Tests | `[test command]` or — |

### 🎨 Frontend — `[workspace_paths.frontend]/`
| | |
|---|---|
| Tecnología | [tech_stack.frontend or —] |
| Lenguaje | [tech_stack.language.frontend or —] |
| Tests | `[test command]` or — |

### 🛡️ Revisión
| | |
|---|---|
| Approver | [Desactivado · or Advisory / Autonomous / ⚠ Unattended (max_revisions N)] |
| Gates del approver | [requisitos, plan · requisitos, plan, arquitectura · or —] |
| Gate de arquitectura | [Humano · or "Approver (unattended)"] |
| Commands post-QA [gate] | Siempre te piden aprobación |
| Formato de requisitos | [REQ · or Historias de usuario] |

### 🔗 Tracker
| | |
|---|---|
| Gestor | [No conectado · or Jira / Plane por MCP (`server`)] |
| Proyecto | [project key · or workspace / identifier · or —] |
| Modo | [pull · create · both · or —] |
| Ruta | [start → In Progress · development → … · qa → … · done_pipeline → … · blocked → … · or —] |
| Escrituras | Mover por la ruta: automático · Crear y comentar: siempre con tu aprobación |

(Omit every row but "Gestor" when no tracker is connected.)

### 🌿 Git
| | |
|---|---|
| Convenciones | [Detectadas · Por repo · Sin convención (valores por defecto)] |
| Rama | [branch_pattern] desde [base_branch · la rama actual] |
| Commit | [commit_pattern] · sin key: [commit_pattern_no_key] |
| Protegidas | [protected_branches] |
| Commit tras QA | [Con aprobación · Sin preguntar · No] — nunca hace push |

### ▶️ Próximos pasos
| Comando | Para qué |
|---|---|
| `/agent-os:start "describe tu primera tarea"` | Pipeline completo de 5 etapas |
| `/agent-os:discover "describe tu tarea"` | Empezar solo con Discovery |
| `/agent-os:status` | Ver el estado actual |
| `/agent-os:review` | Consola de aprobaciones (gate pendiente, veredicto del approver, historial) |
| `/agent-os:start VR-123` · `/agent-os:tracker pick` | Empezar desde un issue (solo con tracker en pull o both) |
| `/agent-os:commit` | Commit local por repo con las convenciones de git (nunca hace push) |
```

To change these options later: `/agent-os:setup --reconfigure`, or edit `approver` and `preferences.requirements_format` in `.agent-os/config.json` (README → Approver). The tracker: `/agent-os:tracker setup | map | off`.
