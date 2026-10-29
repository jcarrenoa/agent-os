# Agent OS — Development Instructions

This is the agent-os plugin repository itself. These instructions apply when working on this codebase.

## What This Repository Is

This is a Claude Code plugin that provides a multi-agent software development framework. It is not a runnable application — it is a collection of Markdown instruction files, JSON schemas, and metadata consumed by Claude Code.

## Repository Structure

```
agent-os/
├── .claude-plugin/plugin.json              ← Plugin manifest (name, author)
├── .claude-plugin/marketplace.json         ← Local dev marketplace (source: "./")
├── agents/*.md                             ← Subagent definitions
├── commands/*.md                           ← Slash command definitions
├── skills/*/SKILL.md                       ← Model-invoked skills
│   ├── tech-adaptation/SKILL.md            ← Default stack skill (extend mode)
│   ├── skill-resolution/SKILL.md           ← Extension resolution protocol
│   └── approval-gates/SKILL.md             ← Gate presentation protocol
├── schemas/*.schema.json                   ← JSON schemas for state validation
│   ├── config.schema.json                  ← Project config (includes extensions)
│   └── defaults.schema.json               ← User ~/.agent-os/defaults.json shape
├── workflows/*.md                          ← Detailed stage workflow instructions
├── catalog/recommended.json                ← Optional recommended extensions (pinned recipes)
├── catalog/templates/*.md                  ← Wrapper skill templates written by the installer
└── marketplace/                            ← Marketplace distribution files
```

## File Authoring Rules

### Agents (`agents/*.md`)
- Frontmatter required: `name`, `description`, `tools`, `model`, `color`.
- `model`: use `opus` for analyst/planner/architect roles, `sonnet` for developer/QA roles.
- `tools`: list only what the agent actually needs. Read-only agents should not have `Write` or `Edit`.
- Instructions must be clear, step-by-step, and actionable. Claude follows them literally.

### Commands (`commands/*.md`)
- Frontmatter required: `description` (shown in /help).
- Optional: `argument-hint`, `allowed-tools`, `model`.
- Commands are the user-facing entry points. They orchestrate agents and manage state.
- Commands must always check for `.agent-os/config.json` before doing anything.
- Commands must always read state before writing it.

### Skills (`skills/<name>/SKILL.md`)
- Frontmatter required: `name`, `description`, `version`.
- The `description` field is what triggers the skill — it must be precise and include example phrases.
- Skills provide protocols and patterns, not direct actions.

### Schemas (`schemas/*.schema.json`)
- Must be valid JSON Schema Draft-07.
- Use `additionalProperties: false` on all object schemas.
- All required fields must be in the `required` array.
- Keep descriptions on every property.

### Workflows (`workflows/*.md`)
- Must define: Entry Conditions, Exit Conditions, The Agent's Job, State Updates.
- Entry and exit conditions must be checkable (boolean, not vague).
- State Updates section must be precise — exact field names and values.

## What NOT to Do

- Do not add a `CLAUDE.md` inside `.agent-os/` — that directory is created in target projects, not here.
- Do not create any runnable code (Python, TypeScript, etc.) in this repository — this plugin works entirely through Markdown instructions. The install/uninstall commands in `catalog/recommended.json` are data, not scripts: they run only through the `extension-installer` skill, after the user explicitly approves the exact commands.
- Do not duplicate instructions across files — reference the skills instead.
- Do not hardcode assumptions about the target project's tech stack in the plugin files.
- Do not hardcode external skill names or doc paths in agents or commands. Extensions are resolved dynamically via the `skill-resolution` skill. Agents receive resolved paths in their `## Extensions` prompt block.
  - **Documented exception — the recommended catalog.** `catalog/recommended.json` names specific third-party extensions. It is optional data: extension resolution works by name and never depends on it, and agents and commands still never reference those names. Rules for the catalog:
    - Every source is pinned to an exact version or full commit SHA, never `latest` or a branch. Bump `catalog_version` whenever a pin, recipe or template changes.
    - Only catalog entries can be installed, and only after the user approves the exact commands (`extension-installer` skill). Nothing outside a recipe is ever run or modified.
    - Before adding or re-pinning an entry, review the upstream code at that pin (install scripts, telemetry, network calls, side effects) and list the risks in the entry's `notes`.
    - Recipes never run upstream installers that change the user's Claude Code configuration (e.g. `codegraph install`).
- Do not run commands or agents with external effects (publish, push, deploy, merge, send) automatically. They must always go through an approval gate per the `approval-gates` skill.

## Versioning

When making changes:
- Update `plugin.json` if the plugin name or description changes.
- Update `marketplace/.claude-plugin/marketplace.json` if the git URL changes.
- Update the `version` field in skill frontmatter when a skill's behavior changes.
- Commit with descriptive messages that explain what changed and why.

## Testing the Plugin

To test this plugin against a real project:
1. The plugin must be installed or symlinked in the target project's Claude Code settings.
2. Navigate to a test project directory.
3. Run `/agent-os:setup` and verify `.agent-os/config.json` is created correctly.
4. Run `/agent-os:start "test task"` and verify the full pipeline runs.
5. There are no automated unit tests for this plugin — all testing is integration testing against real Claude Code sessions.

## Known Limitations

- Claude Code does not support "installation hooks" — there is no code that runs at install time. Onboarding is done via `/agent-os:setup`.
- The plugin cannot enforce schema validation automatically — agents must follow the state-management skill's protocol.
- Parallel agent spawning for multiple subtasks in development relies on Claude's ability to spawn parallel agents; behavior may vary.
- The marketplace URL in `marketplace.json` must be updated to the actual Git remote after the repository is published.
