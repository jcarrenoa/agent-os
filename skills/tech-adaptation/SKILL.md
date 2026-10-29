---
name: tech-adaptation
description: Plugin default skill for the architect, backend-developer and frontend-developer agents. Added automatically when mode is "extend" (the default); mode "replace" omits it without affecting other extensions. Resolved by the skill-resolution skill to an absolute path under the plugin root and passed in the agent's Extensions block.
version: 1.2.0
---

# Tech Adaptation Protocol

## Step 1: Read the Stack

At the start of any development task, read `.agent-os/config.json` and extract:
- `tech_stack.backend`
- `tech_stack.frontend`
- `tech_stack.mobile`
- `tech_stack.database`
- `tech_stack.package_manager`
- `tech_stack.language`

If `config.json` does not exist, tell the user to run `/agent-os:setup` and stop.

## Step 2: Backend Patterns

Apply these patterns based on `tech_stack.backend`:

### `fastapi`
- Architecture: Hexagonal (Ports & Adapters) or Clean Architecture
- Language: Python 3.11+, type hints everywhere
- Async: Use `async/await` for all I/O operations
- Models: Pydantic v2 (`BaseModel`, `model_validator`, `field_validator`)
- Router structure: `app/routers/`, `app/services/`, `app/repositories/`, `app/domain/`
- Dependency injection via FastAPI `Depends()`
- Tests: pytest + pytest-asyncio, `httpx.AsyncClient` for integration tests
- Env config: `pydantic-settings`, `.env` files

### `nestjs`
- Architecture: Hexagonal/DDD with NestJS modules
- Language: TypeScript strict mode
- Structure: `src/modules/<domain>/`, each with controller, service, repository, dto, entity
- DI: NestJS IoC container, `@Injectable()`, `@Controller()`
- ORM: Prefer Prisma or TypeORM
- Tests: Jest, `@nestjs/testing`, `supertest` for e2e
- Validation: `class-validator`, `class-transformer`

### `springboot`
- Architecture: Hexagonal + DDD
- Language: Java 17+ or Kotlin
- Layers: `domain/`, `application/`, `infrastructure/`, `interfaces/`
- Spring annotations: `@Service`, `@Repository`, `@RestController`, `@Entity`
- ORM: Spring Data JPA
- Tests: JUnit 5, Mockito, `@SpringBootTest`, `@WebMvcTest`
- Config: `application.yml`

### `dotnet`
- Architecture: Clean Architecture (Domain → Application → Infrastructure → Presentation)
- Language: C# 12, nullable reference types enabled
- CQRS: MediatR for commands and queries
- ORM: Entity Framework Core
- Validation: FluentValidation
- Tests: xUnit, Moq, `WebApplicationFactory`
- Config: `appsettings.json`, `IOptions<T>`

### `go`
- Architecture: Hexagonal / Standard Go patterns
- Language: Go 1.21+, idiomatic error handling
- Structure: `internal/`, `pkg/`, `cmd/`
- Interface-based design: define interfaces in the consumer package
- No frameworks preferred; use `net/http` stdlib or `chi`/`fiber`
- ORM: `pgx` (PostgreSQL) or `database/sql`
- Tests: `testing` stdlib, table-driven tests, `testify`

### `express`
- Architecture: MVC with service layer
- Language: TypeScript strict mode
- Structure: `src/routes/`, `src/controllers/`, `src/services/`, `src/models/`
- Validation: `zod` or `joi`
- ORM: Prisma (preferred) or Sequelize
- Tests: Jest + supertest

### `laravel`
- Architecture: MVC with service/repository pattern
- Language: PHP 8.2+
- Eloquent ORM with repository pattern
- Laravel conventions: `app/Models/`, `app/Http/Controllers/`, `app/Services/`
- Tests: PHPUnit + Pest

### `rails`
- Architecture: MVC, Rails conventions
- Language: Ruby 3.x
- Active Record ORM
- Tests: RSpec or Minitest

## Step 3: Frontend Patterns

Apply these patterns based on `tech_stack.frontend`:

### `react`
- Architecture: MVVM (ViewModel = hooks, View = components, Model = domain types)
- Language: TypeScript strict mode
- State: Zustand (global) + React Query/TanStack Query (server state)
- Components: Functional only, no class components
- Styling: Tailwind CSS (preferred) or CSS Modules
- Forms: React Hook Form + Zod
- Tests: Vitest + Testing Library

### `next`
- All React patterns above, plus:
- App Router (not Pages Router for new projects)
- Server Components by default, `"use client"` only when needed
- Server Actions for mutations
- Route handlers for API endpoints

### `vue`
- Architecture: MVVM, Composition API
- Language: TypeScript with `<script setup>`
- State: Pinia
- Routing: Vue Router 4
- Styling: Tailwind CSS or CSS Modules
- Tests: Vitest + Vue Testing Library

### `nuxt`
- All Vue patterns above, plus:
- Nuxt 3 auto-imports
- Server routes for API (`server/api/`)
- `useFetch` / `useAsyncData` for data fetching

### `angular`
- Architecture: Modules with Standalone Components (Angular 17+)
- Language: TypeScript strict mode
- State: NgRx Signals or NGRX Store
- DI: Angular IoC container
- Forms: Reactive Forms with typed form groups
- Tests: Jest + Angular Testing Library

### `flutter`
- Architecture: BLoC (preferred) or Riverpod
- Language: Dart 3, null safety
- Structure: `lib/features/<feature>/`, `lib/core/`, `lib/shared/`
- BLoC: `flutter_bloc`, events/states as sealed classes
- Navigation: go_router
- DI: `get_it`
- Tests: `flutter_test`, `bloc_test`

### `svelte`
- Architecture: Component-based, Svelte stores for state
- Language: TypeScript
- SvelteKit for routing and SSR
- Tests: Vitest + Testing Library

## Step 4: Database Patterns

Apply these based on `tech_stack.database`:

### `postgresql`
- Use parameterized queries, never string interpolation
- Migrations: project-standard tool (Alembic, Flyway, Prisma Migrate, EF Migrations, golang-migrate)
- Indexes: declare indexes for all foreign keys and frequent query fields

### `mongodb`
- Schema validation via application layer (Pydantic, Mongoose, etc.)
- Use aggregation pipeline for complex queries
- Index strategy based on query patterns

### `supabase`
- Use Supabase client SDK
- Row Level Security (RLS) policies required for all user-scoped tables
- Prefer Supabase Edge Functions for server-side logic

## Step 5: Apply Conventions from CLAUDE.md

After applying stack-specific patterns, read the workspace root `CLAUDE.md` (if present) and, when you work in a `working_directory` (monorepo / multi-repo), also `[working_directory]/CLAUDE.md`. Any conventions, restrictions, or patterns defined there **override** the defaults from this skill; on conflict between the two, the subrepo's `CLAUDE.md` wins.

## Non-Negotiable Rules (all stacks)

- Validate input at system boundaries (HTTP handlers, queue consumers, form submissions)
- Never expose internal error details to API clients
- Use environment variables for secrets and config — never hardcode
- Write tests for new business logic
- Follow the existing code style of the project — do not introduce new patterns that conflict with existing ones
