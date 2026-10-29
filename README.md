# Agent OS

> **A reusable multi-agent software development team for Claude Code.**

Agent OS gives any Claude Code project a structured, repeatable engineering workflow with specialized agents for each phase of development.

---

## What It Does

Every task runs through a 5-stage pipeline:

| Stage | Agent | Gate |
|-------|-------|------|
| 1. Discovery & Requirements | Requirements Analyst | ✓ Human approval |
| 2. Analysis & Planning | Planner | Auto (or manual on scope change) |
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
| `/agent-os:start "<task>"` | Run a task through the complete 5-stage pipeline. |
| `/agent-os:status` | Show current pipeline state, active task, and pending approvals. |
| `/agent-os:approve` | Approve a pending gate and advance to the next stage. |
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

Agent OS does not ship or depend on these tools. They are third-party projects that fit the pipeline well; **review their code before installing them.**

| Tool | What it does | Agents | Requirements |
|------|--------------|--------|--------------|
| [CodeGraph](https://github.com/colbymchenry/codegraph) (`colbymchenry/codegraph`, npm `@colbymchenry/codegraph`) | Local graph of symbols, calls and imports (index in `.codegraph/`). Agents query it instead of grepping and reading files one by one. | planner, backend-developer, frontend-developer, qa-engineer | MCP server + CLI, **not a skill**. Subagents reach it through the `codegraph` CLI, so they need `Bash` (see below). `codegraph init` in every repo you want indexed. |
| [The AntiSlop](https://github.com/aplaceforallmystuff/the-antislop) (`aplaceforallmystuff/the-antislop`, MIT) | Detects and fixes AI-sounding writing patterns. | requirements-analyst, architect | Skill `antislop` installed in `~/.claude/skills/antislop/` (see below). |

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

agent-os looks up skills in `.claude/skills/`, `~/.claude/skills/` and its own plugin, not in other plugins. Its marketplace install (`/plugin install the-antislop@aplaceforallmystuff`) is therefore not visible to agent-os. Also, the repository keeps the skill in `skills/antislop/`, so cloning the whole repo into `~/.claude/skills/` does not produce a loadable skill. Copy the skill folder instead:

```bash
git clone https://github.com/aplaceforallmystuff/the-antislop.git /tmp/the-antislop
cp -r /tmp/the-antislop/skills/antislop ~/.claude/skills/antislop
```

Then assign it in `~/.agent-os/defaults.json` (all your projects):

```json
{
  "agents": {
    "requirements-analyst": { "skills": ["antislop"] },
    "architect": { "mode": "extend", "skills": ["antislop"] }
  }
}
```

It is not recommended for the developers: it targets prose, not code.

If a configured skill is not installed, skill-resolution shows `⚠ skill '<name>' not found — skipped` and the agent continues with its defaults. You can also assign skills with `/agent-os:skills` (menu → "Todos mis proyectos").

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
2. **After Architecture Design** — You approve the architecture before any code is written.

Additional approvals are requested automatically when:
- Critical QA issues are found
- Planning detects a scope expansion
- A destructive or irreversible operation is needed

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
