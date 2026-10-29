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
| `/agent-os:setup` | Initialize agent-os for this project. Detects and confirms tech stack. |
| `/agent-os:start "<task>"` | Run a task through the complete 5-stage pipeline. |
| `/agent-os:status` | Show current pipeline state, active task, and pending approvals. |
| `/agent-os:approve` | Approve a pending gate and advance to the next stage. |
| `/agent-os:rollback <stage>` | Roll back to a previous stage. |
| `/agent-os:discover "<task>"` | Run Stage 1 (Discovery) only. |
| `/agent-os:plan` | Run Stage 2 (Planning) only. |
| `/agent-os:design` | Run Stage 3 (Design) only. |
| `/agent-os:develop` | Run Stage 4 (Development) only. |
| `/agent-os:qa` | Run Stage 5 (QA) only. |

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
