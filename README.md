# Agent OS

> **A reusable multi-agent software development team for Claude Code.**

Agent OS gives any Claude Code project a structured, repeatable engineering workflow with specialized agents for each phase of development.

---

## What It Does

Every task runs through a 5-stage pipeline:

| Stage | Agent | Gate |
|-------|-------|------|
| 1. Discovery & Requirements | Requirements Analyst | ✓ Human approval |
| 2. Analysis & Planning | Planner | Auto (or manual on scope change, or with the [approver](#approver) in advisory mode) |
| 3. Design & Architecture | Architect | ✓ Human approval |
| 4. Development | Backend Dev + Frontend Dev | Auto |
| 5. Testing & QA | QA Engineer | ✓ Human approval if issues found |

State is persisted between sessions in `.agent-os/` at the project root — fully versionable.

---

## Installation

### From the Agent OS marketplace

```
claude plugins install https://github.com/jcarrenoa/agent-os-marketplace agent-os
```

### Manual (development)

Clone this repository and configure your Claude Code project settings to load it as a local plugin.

---

## Quick Start

```bash
# 1. Initialize in any project (run once)
/agent-os:setup

# 2. Start a task through the full 5-stage pipeline
/agent-os:start "implement user authentication with email and OAuth"

# 3. Check the current state
/agent-os:status

# 4. Approve a gate when prompted
/agent-os:approve
```

---

## Commands

| Command | Description |
|---------|-------------|
| `/agent-os:setup` | Initialize agent-os for this project. Detects stack, offers to connect external skills. |
| `/agent-os:start "<task>"` | Run a task through the complete 5-stage pipeline. With a tracker in pull mode, `/agent-os:start VR-123` (or the issue URL) starts from that issue. |
| `/agent-os:status` | Show current pipeline state, active task, and pending approvals. |
| `/agent-os:approve` | Approve a pending gate and advance to the next stage. |
| `/agent-os:review` | Approval console: pending gate, approver verdict, history; approve, request changes or see the details. |
| `/agent-os:rollback <stage>` | Roll back to a previous stage. |
| `/agent-os:discover "<task>"` | Run Stage 1 (Discovery) only. |
| `/agent-os:plan` | Run Stage 2 (Planning) only. |
| `/agent-os:design` | Run Stage 3 (Design) only. |
| `/agent-os:develop` | Run Stage 4 (Development) only. |
| `/agent-os:qa` | Run Stage 5 (QA) only. |
| `/agent-os:skills list` | List all configured extensions and whether each file exists on disk. |
| `/agent-os:skills add <stage> <name\|path>` | Add a skill, doc, agent, or command to a stage. |
| `/agent-os:skills remove <stage> <name\|path>` | Remove an extension from a stage. |
| `/agent-os:skills mode <stage>.<role> extend\|replace` | Set whether the plugin default (tech-adaptation) is added for a role. |
| `/agent-os:extensions [status\|install\|uninstall\|update]` | Check or install the recommended extensions from the catalog, after approving the exact commands. |
| `/agent-os:commit [KEY]` | One local commit per repo for the task, on a feature branch named by the project's git conventions. Never pushes. See [Git conventions](#git-conventions). |
| `/agent-os:tracker [setup\|map\|pick\|sync\|off]` | Jira / Plane integration over MCP: connect it, re-map the board columns, pick an issue, retry failed syncs. See [Tracker](#tracker). |

---

## Agents

| Agent | Model | Role |
|-------|-------|------|
| Requirements Analyst | Opus | Captures requirements, asks clarifying questions |
| Planner | Opus | Breaks requirements into subtasks, identifies risks |
| Architect | Opus | Designs components, data models, API contracts, records ADRs |
| Backend Developer | Sonnet | Implements backend subtasks, adapts to detected stack |
| Frontend Developer | Sonnet | Implements frontend subtasks, adapts to detected stack |
| QA Engineer | Sonnet | Runs tests, verifies requirements, produces QA report |
| Approver (optional) | Opus | Read-only review of the requirements and the plan; the architecture only in unattended mode. See [Approver](#approver) |

---

---

## Extensions

Agent OS can use skills, docs, agents, and commands that you already have installed — from any Claude Code plugin or your own team guides.

### Two Assignment Types

**By stage/role** — applies to all agents running in that stage or role:
- `discovery`, `planning`, `qa` — all agents in that stage
- `design`, `design.backend`, `design.frontend`
- `development.backend`, `development.frontend`

**By agent** — applies only to one specific agent, regardless of stage:
- `requirements-analyst`, `planner`, `architect`
- `backend-developer`, `frontend-developer`, `qa-engineer`

Agent-level config is applied after stage/role config. If both define `mode`, the agent-level value wins.

### Combination Order

For each agent spawn, the pipeline resolves extensions in five tiers and hands them to the agent in this order (generic → specific):

```
plugin defaults (tech-adaptation, unless mode = replace)
  └── context                 (global docs)
        └── stages.<stage>         (all agents in the stage)
              └── stages.<stage>.<role>  (backend or frontend agents only)
                    └── agents.<agent>   (this specific agent only)
```

Within each tier, user-level entries come first and project-level entries after them. If a name appears twice, only its most specific occurrence is kept. Every agent's `## Extensions` block states the precedence rule explicitly: **later entries override earlier ones on conflict; the project's CLAUDE.md overrides all.**

The architect works for every role, so it gets a shared section plus one section per role active in `focus` (each with its own mode) instead of a merged list.

### Skill names

- `my-skill`: looked up in the project's `.claude/skills/` (the role's subrepo first in monorepo/multi-repo), then `~/.claude/skills/`, then agent-os itself.
- `<plugin>:<skill>`: a skill from an installed plugin, written the same way Claude Code invokes it (`<plugin>` is the plugin's manifest name). agent-os finds the plugin through Claude Code's install registry (`~/.claude/plugins/installed_plugins.json`), or by scanning the plugin cache when there is no registry. `agent-os:<skill>` always points at agent-os's own skills.

A skill that cannot be found produces a warning and is skipped; the pipeline keeps going.

### Three Configuration Levels

| Level | Source | Priority |
|-------|--------|----------|
| Project | `.agent-os/config.json` → `extensions` | Highest (applied last within each tier) |
| User | `~/.agent-os/defaults.json` | Middle |
| Plugin | Built-in tech-adaptation skill | Lowest (applied first) |

Plugin files (workflows, schemas, built-in skills) are always addressed through `${CLAUDE_PLUGIN_ROOT}`, which Claude Code substitutes with the installed plugin's absolute path, so they are found whether the plugin comes from a marketplace, the cache, or `--plugin-dir`.

### Monorepo and multi-repo

- Project-level skills, agents, and commands are looked up first in the role's subrepo (`<workspace_paths.backend|frontend>/.claude/`) and then in the workspace root `.claude/`.
- Developers read `.agent-os/` and the root `CLAUDE.md` from the workspace root, write code inside their subrepo, and also read the subrepo's `CLAUDE.md` (which wins over the root one on conflict).

### Mode: extend vs replace

`mode` controls only the plugin defaults:

- **`extend`** (default): built-in `tech-adaptation` is added first, then your context, stage, role, and agent entries.
- **`replace`**: `tech-adaptation` is not added. Your context, stage, role, and agent entries are still applied — `replace` never removes them. Applies to architect, backend-developer, frontend-developer, and their stage/role equivalents.

### Post-QA commands and approval

Every post-QA command asks for approval before it runs — whatever its description says, and even if it has no frontmatter. To let a command run without approval you must mark it explicitly:

```json
"commands": ["review-pull-request", { "name": "update-changelog", "auto": true }]
```

Here `review-pull-request` is gated and `update-changelog` runs automatically. From the CLI: `/agent-os:skills add qa update-changelog --auto`.

Built-in commands are listed with the `agent-os:` prefix, e.g. `"agent-os:commit"` (see [Git conventions](#git-conventions)).

### How agents apply extensions

- **Read first, verifiably**: before doing anything else, every agent reads each Skills and Docs path from its `## Extensions` block with the Read tool, in order. Its final report ends with an `EXTENSIONS APPLIED` section listing each path and how it was applied (or `not applicable`). The stage command compares that section with the resolution; a missing section or path is reported to you and recorded in the task file (`stages.<stage>.extension_warnings`), without blocking the pipeline.
- **No self-discovery**: agents never read `.claude/skills/`, `.claude/commands/`, `.claude/agents/` (project, subrepo or `~/.claude/`) or other plugins' skills unless that exact path is in their Extensions block. Only the resolution decides what applies. `CLAUDE.md` files are still read.
- **References, not content**: stage commands pass file paths (task, config, ADRs, Extensions) and subtask IDs, never the pasted design, ADRs or `CLAUDE.md` rules, so agents read the real files instead of a summary.
- **Development runs one agent per role**, with that role's subtasks in dependency order. Backend and frontend run in parallel when neither depends on the other; status is still tracked per subtask.

### Three Ways to Configure

**1. Interactive menu** (recommended — no arguments needed):
```
/agent-os:skills
```
Guides you through selecting a destination, choosing elements, setting mode, and saving.

**2. With arguments** (scripting):
```bash
# By stage/role
/agent-os:skills add development.backend my-fastapi-guide
/agent-os:skills add qa ~/docs/team-qa-checklist.md
/agent-os:skills mode development.frontend replace

# By agent
/agent-os:skills add --agent architect my-arch-guide
/agent-os:skills add --agent qa-engineer security-scanner
/agent-os:skills mode --agent backend-developer replace

# Write to user defaults instead of project config
/agent-os:skills add --agent planner my-planning-guide --global

# Post-QA command that runs without approval (all others are gated)
/agent-os:skills add qa update-changelog --auto

# View what a specific agent or stage would load
/agent-os:skills list --agent backend-developer
/agent-os:skills list development.backend
```

**3. Edit JSON directly**: `.agent-os/config.json` → `extensions` (project) or `~/.agent-os/defaults.json` (user). See `schemas/config.schema.json` and `schemas/defaults.schema.json` for the full shape.

### Example

```json
"extensions": {
  "context": ["~/docs/team-principles.md"],
  "agents": {
    "architect": { "mode": "extend", "skills": ["my-arch-patterns"] },
    "backend-developer": { "mode": "replace", "skills": ["my-fastapi-guide"], "docs": ["~/api-spec.md"] }
  },
  "stages": {
    "development": {
      "frontend": { "mode": "extend", "skills": ["react-conventions"] }
    },
    "qa": {
      "agents": ["security-scanner"],
      "commands": ["notify-slack"]
    }
  }
}
```

In this example:
- `architect` gets tech-adaptation + my-arch-patterns (per active role)
- `backend-developer` gets my-fastapi-guide and ~/api-spec.md plus the context doc, without tech-adaptation (`replace` only drops the plugin default)
- `frontend-developer` gets tech-adaptation + react-conventions
- `qa-engineer` runs alongside security-scanner; after QA passes, notify-slack asks for approval before running (it is not marked `auto`)

### Recommended extensions

Agent OS does not ship or depend on these tools. They are third-party projects that fit the pipeline well; **review their code before installing them.** agent-os can install them for you with your approval (see [Automatic installation](#automatic-installation)); the manual steps below still work.

| Tool | What it does | Agents | Requirements |
|------|--------------|--------|--------------|
| [CodeGraph](https://github.com/colbymchenry/codegraph) (`colbymchenry/codegraph`, npm `@colbymchenry/codegraph`) | Local graph of symbols, calls and imports (index in `.codegraph/`). Agents query it instead of grepping and reading files one by one. | planner, backend-developer, frontend-developer, qa-engineer | MCP server + CLI, **not a skill**. Subagents reach it through the `codegraph` CLI, so they need `Bash` (see below). `codegraph init` in every repo you want indexed. |
| [The AntiSlop](https://github.com/aplaceforallmystuff/the-antislop) (`aplaceforallmystuff/the-antislop`, MIT) | Detects and fixes AI-sounding writing patterns. | requirements-analyst, architect | Plugin `the-antislop` (assign `the-antislop:antislop`), or the skill copied to `~/.claude/skills/antislop/` (assign `antislop`). See below. |

> Several unrelated projects are called "codegraph". The one meant here is `colbymchenry/codegraph`.

#### CodeGraph

CodeGraph is not configured through `extensions`: it is not a skill, so listing it under `skills` would only produce a "skill not found" warning. Install and wire it once, then index each repo:

```bash
npm install -g @colbymchenry/codegraph   # or the installer from its README
codegraph install                         # wires the MCP server and adds usage instructions to CLAUDE.md
codegraph init                            # run in each repo to index (creates .codegraph/)
```

- **How agent-os agents use it**: plugin agents only get the tools in their `tools` frontmatter, so the CodeGraph MCP tool does not reach them. They use the CLI (`codegraph explore "<question>"`) through `Bash`, following the instructions `codegraph install` writes to `CLAUDE.md`. That works today for `planner`, `backend-developer`, `frontend-developer` and `qa-engineer`. `architect` and `requirements-analyst` have no `Bash` and keep reading files as usual.
- **Monorepo / multi-repo**: run `codegraph init` inside each subrepo (`workspace_paths.backend`, `workspace_paths.frontend`); developers run their commands from their `working_directory`.
- **When it pays off**: the savings grow with how much code exploration a task needs (large or unfamiliar codebases). On small repos the gain is small, and its answers stay in the agent's context window longer than short file reads.

#### The AntiSlop

Install it either way; the name you assign depends on how it was installed.

**As a plugin (recommended):** the plugin's manifest name is `the-antislop` and its skill is `antislop`, so the name to assign is `the-antislop:antislop`.

```
/plugin marketplace add aplaceforallmystuff/marketplace
/plugin install the-antislop@aplaceforallmystuff
```

```json
{
  "agents": {
    "requirements-analyst": { "skills": ["the-antislop:antislop"] },
    "architect": { "mode": "extend", "skills": ["the-antislop:antislop"] }
  }
}
```

**As a plain skill:** the repository keeps the skill in `skills/antislop/`, so cloning the whole repo into `~/.claude/skills/` does not produce a loadable skill. Copy the skill folder instead and assign it as `antislop`:

```bash
git clone https://github.com/aplaceforallmystuff/the-antislop.git /tmp/the-antislop
cp -r /tmp/the-antislop/skills/antislop ~/.claude/skills/antislop
```

```json
{
  "agents": {
    "requirements-analyst": { "skills": ["antislop"] },
    "architect": { "mode": "extend", "skills": ["antislop"] }
  }
}
```

Both examples go in `~/.agent-os/defaults.json` (all your projects).

It is not recommended for the developers: it targets prose, not code.

If a configured skill is not installed, skill-resolution shows `⚠ skill '<name>' not found — skipped` and the agent continues with its defaults. You can also assign skills with `/agent-os:skills` (menu → "Todos mis proyectos").

### Automatic installation

agent-os can install the recommended extensions itself, but only the ones in its catalog (`catalog/recommended.json`) and only with your explicit approval:

- **During `/agent-os:setup`**: an optional step asks which recommended extensions to install ("Ninguna" installs nothing).
- **Any time**: `/agent-os:extensions status` shows what is in the catalog, what is installed and in which version, and which agents use it. `install <id>`, `uninstall <id>` and `update <id>` change it.

How it works:

- **You approve the exact commands.** Before anything runs, agent-os lists every command it will execute and every file it will write, together with each extension's license and risks (e.g. CodeGraph's telemetry). Optional steps such as `codegraph telemetry off` are included by default; leaving one out takes an explicit choice. Without your approval nothing is installed.
- **Clean `git status`.** CodeGraph's `.codegraph/` is added to each repository's `.git/info/exclude` (local, never committed) if it is not already there. `uninstall` removes that line only where the installer added it, and keeps `~/.codegraph/telemetry.json` so a reinstall keeps telemetry off (the report says so; delete `~/.codegraph/` yourself if you will not reinstall).
- **Pinned versions only.** Each catalog entry is pinned to an exact version or commit (CodeGraph `1.6.2`, The AntiSlop commit `f123fdc`). `update` only moves to the version pinned in the catalog. Nothing outside the catalog can be installed.
- **Wrapper skills.** The installer writes small agent-os skills, `use-codegraph` and `use-antislop`, and assigns them in `~/.agent-os/defaults.json`:
  - `use-codegraph` (planner, backend-developer, frontend-developer, qa-engineer): runs `codegraph sync` and `codegraph explore` before reading code, and falls back to normal reads when there is no index. It never runs `codegraph install`, so your `CLAUDE.md` and MCP configuration are untouched.
  - `use-antislop` (requirements-analyst and architect, after `antislop` itself): audits only the prose the agent wrote. Structural patterns apply in any language; English-specific lexical patterns only when writing in English. It never runs the upstream pattern-refresh commands.
- **Your configuration is preserved.** Assignments are merged into `~/.agent-os/defaults.json`; `uninstall` removes only what the installer added (recorded in `~/.agent-os/installed-extensions.json`). Files the installer did not create are never overwritten.
- **Idempotent and non-blocking.** Running install again does nothing for an extension that is already up to date. A failed step is reported and recorded, and setup and the pipeline continue without that extension.

---

## Supported Technology Stacks

### Frontend
React, Next.js, Vue, Nuxt, Angular, Flutter Web, Svelte

### Backend
FastAPI, NestJS, Spring Boot, .NET, Go, Express, Laravel, Rails

### Mobile
Flutter, React Native, Swift, Kotlin

### Database
PostgreSQL, MySQL, MongoDB, SQLite, Firestore, Supabase

---

## How State Works

Agent OS stores all state in `.agent-os/` at your project root:

```
.agent-os/
├── config.json        ← Project config and tech stack (from /agent-os:setup)
├── state.json         ← Current pipeline position
├── tasks/
│   └── TASK-001.json  ← Task with requirements, subtasks, test results
└── decisions/
    └── DEC-001.json   ← Architecture decisions (ADRs)
```

These files are tracked in git by default, giving your team full history and traceability between requirements, tasks, and code changes.

---

## Approval Gates

Agent OS requires human approval at two mandatory checkpoints:

1. **After Discovery** — You confirm the requirements are complete and accurate before planning begins.
2. **After Architecture Design** — You approve the architecture before any code is written (the approver approves it only in [unattended mode](#unattended-mode)).

Additional approvals are requested automatically when:
- Critical QA issues are found
- Planning detects a scope expansion
- A destructive or irreversible operation is needed

`/agent-os:review` is the approval console: it shows the pending gate (requirements, plan or architecture) with a summary and the paths of its files, the approver verdict when there is one, and the approval history of the task (stage, approved by human or approver, date, revisions). From there you can **Aprobar** (the same state change as `/agent-os:approve`), **Pedir cambios** (you write the changes and the producing agent is re-run with them and with the approver's findings), **Ver detalle** (the full requirements or user stories, plan, or ADRs) or **Cancelar**.

---

## Approver

An optional agent (`approver`, Opus, read-only: Read, Glob, Grep) that reviews the requirements and the plan before their gates (and, only in [unattended mode](#unattended-mode), the architecture). It is **off by default**: without the `approver` key in `config.json`, or with `enabled: false`, the pipeline behaves exactly as without it.

It starts with a clean context: the command gives it only file paths (task file, config, the stage workflow) and its `## Extensions` block, never the conversation or the other agent's report. It follows the same extension rules as every agent (Read its extensions first, no self-discovery, `EXTENSIONS APPLIED`).

- **Requirements review**: completeness against your original request, ambiguity, testable acceptance criteria, INVEST for user stories, conflicts with `CLAUDE.md`, and whether the analyst's assumptions are reasonable.
- **Plan review**: every requirement or story covered by at least one subtask, correct dependencies and role assignment (respecting `focus`), no out-of-scope subtasks, risks identified.
- **Output**: `VERDICT: APPROVE | REQUEST_CHANGES` and findings with severity (`blocker`, `major`, `minor`), each with a concrete proposed change. Any `blocker` or `major` means `REQUEST_CHANGES`. Written in `output_language`.

### Modes

| Mode | APPROVE | REQUEST_CHANGES |
|---|---|---|
| `advisory` (default) | Shown at the human gate; you decide | Shown at the human gate; you decide (you can still approve) |
| `autonomous` | The stage advances without asking, recorded as `approved_by: "approver"` | With `max_revisions > 0`, the producing agent is re-run with the findings, up to `max_revisions` times; then it stops at your gate. With `0`, it stops at your gate directly |
| `unattended` | Same as `autonomous`, also for the architecture | Re-runs up to `max_revisions` (≥ 1); then continues with the `major`/`minor` findings left unresolved, or stops if a `blocker` is left. See [Unattended mode](#unattended-mode) |

In `advisory`, the planning stage, which normally advances on its own, gets a human gate while the approver reviews it.

Once you ask for changes at a gate ("Pedir cambios" or in the conversation), the next gate of that stage is yours in every mode: the approver reviews the new version, but it does not approve it for you. Your change requests never use up `max_revisions`.

### The architecture gate

The architecture gate decides how the code will be built and is the last checkpoint before anything is written, so it is decided by a person in `advisory` and `autonomous`: `config.schema.json` allows `design` in `gates` only when `mode` is `unattended`. Autonomous mode also never skips the planner's scope-change pause.

### What is recorded

Per stage in the task file:
- `approved_by` (`human` | `approver`);
- `review`, the latest approver review (`round`, verdict, date, findings), and `review_history` with the earlier rounds. Finding IDs carry their round (`R1-F-01`, `R2-F-01`), so a reference to a finding never changes meaning after a new review;
- `auto_revisions`, the re-runs started by the approver (and, in unattended mode, by QA), compared with `max_revisions`;
- `change_requests` (`CR-1`, `CR-2`...: what you asked to change, verbatim, with a copy of the findings you cited);
- `unresolved_findings` (unattended mode only).

The task also keeps `open_questions` (each clarifying question with its answer and `resolved_by`: `human`, `approver` or `default`) and `approver_decisions` (every decision the approver took instead of you). `state.json.pending_approval.approver_verdict` holds the verdict of the pending gate. `/agent-os:status` and `/agent-os:review` show them.

### Clarifying questions

The requirements analyst never answers its own questions. When something is ambiguous it returns a `QUESTIONS` block; the command asks you with a multiple-choice question (the analyst's proposal first), stores your answers in `open_questions`, and re-runs the analyst, which reads them from the task file. At most two rounds. In unattended mode the approver answers instead, and anything it cannot decide (who the users are, money, legal or security policy, deleting data) blocks the task.

### Configuration

Choose it in `/agent-os:setup` (Desactivado / Advisory / Autonomous / Unattended), change it with `/agent-os:setup --reconfigure`, or edit `.agent-os/config.json`:

```json
"approver": {
  "enabled": true,
  "mode": "advisory",
  "gates": ["requirements", "planning"],
  "max_revisions": 0
}
```

`gates` accepts `requirements` and `planning`, plus `design` only with `"mode": "unattended"`. `max_revisions` (0–5) applies to `autonomous` and `unattended` (where it must be at least 1). The approver can have its own extensions: `/agent-os:skills add --agent approver <name|path>`.

### Unattended mode

> ⚠ **Nothing asks you for confirmation except post-QA commands tagged `[gate]`.** An agent decides in your place, including the changes that otherwise always need a person: data model changes, migrations on existing data, authentication, external integrations. Use it only on tasks and repositories where you accept that, and read the decisions report at the end.

```json
"approver": {
  "enabled": true,
  "mode": "unattended",
  "gates": ["requirements", "planning", "design"],
  "max_revisions": 2
}
```

What the approver does in this mode:
- **Clarifying questions**: it answers them (`resolved_by: "approver"`). A question it cannot decide blocks the task.
- **Requirements, plan and architecture**: it reviews and approves them. The architecture review is stricter: an unmitigated mandatory case (data model, migration, security, external integration) is at least `major`, and one that risks data loss or a security hole is a `blocker`.
- **Scope change**: it keeps the original scope unless the extra work is needed for a `must` requirement; in that case the task is blocked, because expanding the scope is your call.
- **QA failures**: development is re-run with the QA issues, up to `max_revisions` times.
- **After `max_revisions`**: with only `major`/`minor` left, the pipeline continues and records them in `unresolved_findings`; with any `blocker` (or a QA `critical`), the stage is `blocked` and it stops. It never advances with a blocker.
- **Post-QA commands**: `[auto]` commands run; `[gate]` commands still ask you, because they can have external effects (push, deploy, publish, comments). To let a local-only command (for example, a local commit) run without asking, mark it `auto: true`.

At the end (or when it blocks) you get the **Decisiones del approver** report: every gate, question, scope decision and retry it decided, with where to find the details in the task file. `/agent-os:review` shows the same report in its history.

If you ask for changes yourself at any point, that stage's next gate goes back to you.

### Requirements format: user stories

`preferences.requirements_format` in `config.json` (asked in setup):

- `"req"` (default): functional requirements as `REQ-001 [must]: ...`.
- `"user-stories"`: functional requirements as stories, `US-001 [must]: As a <role>, I want <capability>, so that <benefit>.`, each with testable acceptance criteria (`US-001-AC1: Given ..., when ..., then ...`). Non-functional requirements and constraints stay as `REQ-NNN` sections.

The planner traces each subtask to `REQ-NNN`, `US-NNN` or `US-NNN-ACn`, and QA reports coverage per story and per acceptance criterion. Without the key, everything works as with `"req"`.

---

## Git conventions

agent-os does not hard-code a branching model or a commit format. Each project declares its own in `config.git`, and `/agent-os:setup` proposes it from the last 30 commit subjects and the branches of each repo; you confirm it, edit it, or skip it.

```json
"git": {
  "base_branch": "qa",
  "branch_pattern": "feature/{key}-{slug}",
  "commit_pattern": "{key} | {type}: {summary}",
  "commit_pattern_no_key": "{type}: {summary}",
  "commit_types": ["feat", "fix", "chore"],
  "commit_language": "en",
  "protected_branches": ["master", "qa", "dev", "release/*"],
  "repos": { "frontend-repo": { "commit_types": ["feat", "fix", "chore", "style"] } }
}
```

| Placeholder | Value |
|---|---|
| `{key}` | The tracker issue key (`task.external.key`), or the task ID when there is none. Without an issue key, commits use `commit_pattern_no_key` |
| `{task}` | The task ID (`TASK-007`) |
| `{type}` | `task.change_type`, classified by the requirements analyst and checked by the approver |
| `{summary}` | The task title as an imperative sentence in `commit_language`; the whole subject is at most 72 characters |
| `{scope}` | The repo role, or the main module for a single repo |
| `{slug}` | The title in kebab-case ASCII, at most 5 words |

`repos` overrides fields for one repo when the repos follow different conventions. Without a `git` key, branches are `feature/{key}-{slug}` from the current branch, commits are `{type}: {summary}`, and `main`/`master` are protected.

### /agent-os:commit

Creates one local commit per repo with the task's changes:
- **Only the task's changes**: when a task is created, agent-os records `task.git_baseline` (HEAD and the hash of every uncommitted file). Files that were already dirty and did not change stay out; files that were dirty and changed go in with a warning.
- **Never commits secrets or tooling**: `.env*`, `*.pem`, `*.key`, `*.pfx`, `secrets*`, `credentials*`, local `appsettings.*.json`, `.agent-os/`, `.codegraph/`, `node_modules/`, `bin/`, `obj/` are unstaged and listed.
- **Branch**: commits on the task's branch; from `base_branch` it creates the feature branch; from another task's branch or another protected branch it creates it from HEAD and warns. It never commits on a protected branch.
- **Message**: the subject from the pattern, and one body line per subtask of the repo's role.
- **Safe by design**: no `--no-verify`, `--amend` or `--no-gpg-sign`; a failing hook leaves the changes staged and is reported. **It never pushes.**
- Records `task.commits`, which the tracker posts as a comment (always with your approval).

Run it yourself, or let setup add it as a post-QA command (`"agent-os:commit"` in `extensions.stages.qa.commands`), gated by default.

## Tracker

Optional integration with **Jira Cloud** or **Plane**, configured in `/agent-os:setup` (Step 2.2) or later with `/agent-os:tracker setup`. Without the `tracker` key in `config.json`, nothing changes.

Only the orchestrating command talks to the tracker (the `tracker-sync` skill), never a subagent. The connection is the MCP server you register in Claude Code: agent-os never stores or sees a token. A direct REST connection is not available in this version.

### Connecting the MCP server

| Provider | Server | Add it |
|---|---|---|
| Jira | Atlassian Rovo MCP server (remote, OAuth) | `claude mcp add --transport http atlassian <URL from Atlassian's Rovo MCP Server docs>`, then `/mcp` to sign in |
| Plane | `makeplane/plane-mcp-server` (hosted or local) | Hosted: `claude mcp add --transport http plane https://mcp.plane.so/http/mcp`, then `/mcp` to sign in. Local / self-hosted: run it with `uvx` as its README describes, with your API key in its environment (never in `.agent-os/`) |

The server name you choose is stored in `tracker.mcp.server`; agent-os finds the tools by that prefix and tells you if one it needs is missing. Approve the MCP tools in Claude Code's permission prompts, or allow them in your settings.

### Modes

- `pull`: start from an existing issue. `/agent-os:start VR-123` (or its URL) reads the summary, description and acceptance criteria verbatim as the task description, and keeps the issue's existing sub-tasks as reference for the planner. `/agent-os:tracker pick` (or `/agent-os:start` without arguments) lists the issues in your intake column and lets you choose.
- `create`: agent-os creates the parent issue (`Story` by default) with the approved stories or requirements, and one sub-task per planned subtask (`SUB-NN: title`, role as a label or component).
- `both`: either, per task.

### Route of action

In setup (and with `/agent-os:tracker map`) agent-os reads the project's real statuses and proposes which one each pipeline event moves the issue to. You confirm or change every row:

| Event | When | Typical column |
|---|---|---|
| `intake` | Where `pick` looks (no move) | To Do |
| `start` | Discovery starts | In Progress |
| `development` | Development starts (created sub-tasks: when their developer run starts) | In Progress / In Development |
| `qa` | QA starts | QA / Testing |
| `done_pipeline` | QA approves | Code Review (the column **before** Done: the PR is still missing) |
| `blocked` | A stage is blocked | Blocked, or none |

An event without a column moves nothing. In Jira the move uses the issue's available transitions; if the target status is not reachable from the current one, agent-os skips it and records the error. Done is set only if you map `done_pipeline` to it. The Atlassian MCP tools do not expose the board's column layout, so the analysis uses the statuses found in the project's issues and their transitions; you can type a status that was not found.

### What is written, and when you are asked

| Write | Approval |
|---|---|
| Moving the issue (and created sub-tasks) along the route | None: the route you confirmed is the consent. Every move is logged and shown in the stage summary |
| Creating the parent issue, creating the sub-tasks | Always asked, in every mode |
| The commits comment | Always asked, in every mode |

In unattended approver mode, creates and comments are queued and asked together at the end of the run. Nothing else is ever written: no field edits, assignees, deletions or links.

**Commits comment**: after the post-QA commands, if `task.commits` has entries (written by `/agent-os:commit` or another post-QA command), agent-os drafts one comment on the issue (and, in create mode, one per sub-task with its own commits): per repository, the branch, and per commit the short hash, the subject and 1–3 lines of what it does. Nothing else is commented.

### Errors never block

A tracker failure (server not connected, transition not reachable, network error) is recorded in `task.external.sync_errors`, reported in one line, and the pipeline continues. `/agent-os:tracker sync` retries them (moving to the status of the task's current event) and asks for any pending writes. Every write is in `task.external.sync_log` and in the completion report.

### Configuration example

```json
"tracker": {
  "provider": "jira",
  "connection": "mcp",
  "mcp": { "server": "atlassian" },
  "mode": "both",
  "jira": {
    "site_url": "https://acme.atlassian.net",
    "cloud_id": "…",
    "project_key": "VR",
    "parent_type": "Story",
    "subtask_type": "Sub-task",
    "role_field": "label"
  },
  "pick_assigned_to_me": true,
  "status_route": {
    "intake": { "status": "To Do" },
    "start": { "status": "In Progress" },
    "development": { "status": "In Progress" },
    "qa": { "status": "QA" },
    "done_pipeline": { "status": "Code Review" },
    "blocked": null
  }
}
```

For Plane, replace `jira` with `"plane": { "workspace_slug", "project_id", "project_identifier", "base_url" }` and give each route entry its state `id`.

---

## Design Principles

- **Architecture before implementation** — No code is written without approved requirements and architecture.
- **No assumptions** — Agents ask clarifying questions rather than guessing.
- **Traceability** — Every task links requirements → subtasks → decisions → files → tests.
- **Stack agnostic** — Agents adapt their patterns to the technology you confirm at setup.
- **Respect the codebase** — Agents read `CLAUDE.md` and existing patterns before acting.

---

## Author

**Aaron Jose Rodriguez Carreño**
GitHub: [github.com/jcarrenoa](https://github.com/jcarrenoa)

---

## License

© 2026 Aaron Jose Rodriguez Carreño — [PolyForm Noncommercial 1.0.0](LICENSE)

Use, modification, and distribution permitted for non-commercial purposes. Commercial use requires express authorization from the author.
