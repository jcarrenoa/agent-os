---
name: repo-discovery
description: This skill should be used when agent-os needs to inspect an unknown repository before starting any work. Use it at the beginning of /agent-os:setup and at the start of any Discovery stage. It provides a structured protocol for understanding the project without assumptions.
version: 1.0.0
---

# Repo Discovery Protocol

Use this protocol to inspect any repository before acting on it.

## Step 1: Identity

Read these files if they exist (use Glob to check):
- `CLAUDE.md`, `README.md`, `README.rst`
- `package.json` (name, description, scripts)
- `pyproject.toml` (name, description)
- `go.mod` (module name)
- `pom.xml` (artifactId)
- `*.csproj` or `*.sln` (AssemblyName)

Extract: project name, description, project type (new/existing/legacy).

## Step 2: Technology Detection

Check for these files to detect the tech stack:

### Frontend
| File | Technology |
|------|-----------|
| `package.json` with `"react"` in dependencies | React or Next.js |
| `package.json` with `"next"` in dependencies | Next.js |
| `package.json` with `"vue"` in dependencies | Vue or Nuxt |
| `package.json` with `"@angular/core"` | Angular |
| `pubspec.yaml` | Flutter |
| `package.json` with `"svelte"` | Svelte |

### Backend
| File | Technology |
|------|-----------|
| `pyproject.toml` or `requirements.txt` with `fastapi` | FastAPI |
| `pyproject.toml` or `requirements.txt` with `django` | Django |
| `package.json` with `@nestjs/core` | NestJS |
| `package.json` with `express` | Express |
| `go.mod` | Go |
| `pom.xml` with `spring-boot` | Spring Boot |
| `*.csproj` with `Microsoft.NET` | .NET |
| `Gemfile` with `rails` | Rails |
| `composer.json` with `laravel` | Laravel |

### Database
Look for these in config files, `.env.example`, or `docker-compose.yml`:
- `postgres`, `postgresql`, `pg` → PostgreSQL
- `mysql` → MySQL
- `mongo`, `mongodb` → MongoDB
- `sqlite` → SQLite
- `supabase` → Supabase
- `firestore`, `firebase` → Firestore

### Package Manager
| File | Manager |
|------|---------|
| `bun.lockb` | bun |
| `pnpm-lock.yaml` | pnpm |
| `yarn.lock` | yarn |
| `package-lock.json` | npm |
| `uv.lock` | uv |
| `poetry.lock` | poetry |
| `requirements.txt` | pip |
| `go.mod` | go mod |
| `pom.xml` | maven |
| `build.gradle` | gradle |

## Step 3: Development Commands

Check `package.json` scripts for:
- `test`, `test:unit`, `test:e2e` → test command
- `dev`, `start:dev`, `serve` → dev command
- `build`, `compile` → build command
- `lint`, `lint:fix` → lint command

Check `Makefile` for similar targets.

Check `pyproject.toml` for `[tool.pytest]`, `[tool.ruff]`.

## Step 4: Repository Health

Run these shell commands:
```bash
git status
git log --oneline -5
git branch
```

Check for:
- Existing `.gitignore` patterns
- Uncommitted changes (flag if present)
- Current branch name
- Whether it's a monorepo (multiple `package.json` files at different levels)

## Step 5: Existing Instructions

Read in this order (stop when found):
1. `CLAUDE.md`
2. `.claude/CLAUDE.md`
3. `docs/ARCHITECTURE.md`
4. `docs/CONTRIBUTING.md`

These override any assumptions.

## Step 6: Existing Tests

Run `Glob "**/*.test.*"` and `Glob "**/*.spec.*"` to count existing test files.
Check for test directories: `__tests__`, `tests`, `test`, `spec`.

## Output Format

After running discovery, summarize findings as:

```
Project: [name]
Type: [new|existing|legacy]
---
Frontend:    [technology or none]
Backend:     [technology or none]  
Mobile:      [technology or none]
Database:    [technology or none]
Package mgr: [manager]
---
Commands:
  Test:  [command or not found]
  Dev:   [command or not found]
  Build: [command or not found]
---
Git: [clean|dirty] on branch [branch]
Tests: [N test files found]
CLAUDE.md: [found|not found]
```

## Important Rules

- Never assume a tech stack. Always verify by checking files.
- If conflicting signals are found (e.g. both `fastapi` and `nestjs`), list all and ask the user to clarify.
- If the project has both a `frontend/` and `backend/` directory, it is a monorepo — detect stacks for each directory independently.
- Do not modify any files during discovery.
