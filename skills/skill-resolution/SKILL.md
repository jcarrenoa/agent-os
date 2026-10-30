---
name: skill-resolution
description: Resolves which skills, docs, agents, and commands are active for a given pipeline stage, role, and agent, resolves the agent-os plugin root, defines what goes in each agent prompt, and verifies that agents applied their extensions. Use this skill before spawning any agent to determine what extensions are configured, and before reading any file bundled with the plugin (workflows, schemas, plugin skills). Called by every stage command (discover, plan, design, develop, qa) in the agent-os pipeline.
version: 3.5.0
---

# Skill Resolution Protocol

Determines the complete, ordered set of skills and docs an agent receives. Combines three configuration levels (plugin defaults, user defaults, project config) and five tiers ordered from generic to specific (plugin → context → stage → role → agent). Later entries override earlier ones on conflict.

## Plugin Root

Every file bundled with agent-os (workflows, schemas, plugin skills) is addressed with an absolute path built from the plugin root. Never read a plugin file through a path relative to the working directory: the working directory is the user's project, not the plugin.

The plugin root is: `${CLAUDE_PLUGIN_ROOT}`

Claude Code substitutes the plugin root variable inline when it loads skill, command, and agent content (documented in the Claude Code plugins reference, "Where each variable resolves"), so the line above already holds the absolute path of the installed plugin version. Use it as follows:

1. **Normal case**: the line above shows an absolute path. That is `PLUGIN_ROOT`.
2. **Fallback** (only if the line above is not an absolute path because it still starts with `${`, e.g. on an older Claude Code version): glob `~/.claude/plugins/**/skills/tech-adaptation/SKILL.md`. Keep only matches whose plugin root (the directory that contains `skills/`) contains `.claude-plugin/plugin.json` with `"name": "agent-os"`. If several remain, prefer those under `plugins/cache/` and take the most recently modified one. That root is `PLUGIN_ROOT`. If none remain, stop and tell the user the agent-os plugin root could not be found.

The calling command resolves `PLUGIN_ROOT` once per run and passes absolute paths to subagents. Subagents never search for plugin files on their own.

## Inputs

The calling command provides:
- **stage**: `discovery` | `planning` | `design` | `development` | `qa`
- **role** (optional): `backend` | `frontend` — provide when applicable. For the architect, see [Architect: Multiple Roles](#architect-multiple-roles).
- **agent** (required): the exact agent name being spawned (`requirements-analyst`, `planner`, `approver`, `architect`, `backend-developer`, `frontend-developer`, `qa-engineer`). For the `approver`, **stage** is the stage under review (`discovery` for targets `requirements` and `questions`, `planning` for `planning` and `scope-change`, `design` for `design`), so it receives that stage's entries plus its own `agents.approver` entries. For `design` it gets the context, stage and agent tiers only (no role tier and no plugin defaults: it reviews, it does not design).
- **workspace root**: current working directory (where `.agent-os/` lives)

## Step 1: Read Configuration Sources

Read these files (if they exist):

1. `~/.agent-os/defaults.json` — **user level**
2. `.agent-os/config.json` → `extensions` field — **project level**
3. `.agent-os/config.json` → `workspace_type`, `workspace_paths`, `focus` — used in Step 3.

If neither 1 nor 2 has relevant data: proceed with plugin defaults only.

## Step 2: Build the Merged Effective Config

Build the list in five tiers, from least to most specific. Within each tier, user-level entries come first, then project-level entries.

### Tier 1: `plugin` (built-in defaults)

Plugin defaults exist only for agents that design or implement code: `architect`, `backend-developer`, `frontend-developer`. For them, the plugin default is the `tech-adaptation` skill, added only when the resolved mode is `extend`. Other agents have no plugin defaults.

### Tier 2: `context` (global docs, all agents)
```
docs += user.context, project.context
```

### Tier 3: `stages.<stage>` (all agents in this stage)
```
skills += user.stages.<stage>.skills, project.stages.<stage>.skills
docs   += user.stages.<stage>.docs,   project.stages.<stage>.docs
```

### Tier 4: `stages.<stage>.<role>` (agents in this role, if role given)
Apply only when a role is provided AND the stage has per-role config (design, development).
```
skills += user.stages.<stage>.<role>.skills, project.stages.<stage>.<role>.skills
docs   += user.stages.<stage>.<role>.docs,   project.stages.<stage>.<role>.docs
```

### Tier 5: `agents.<agent>` (this specific agent only)
```
skills += user.agents.<agent>.skills, project.agents.<agent>.skills
docs   += user.agents.<agent>.docs,   project.agents.<agent>.docs
```

**Duplicates**: if the same skill name or doc path appears more than once, keep only its **last** occurrence (the most specific tier and level). The final list is ordered generic → specific.

### QA extras (`qa-engineer` only)
```
extra_agents   += user.stages.qa.agents,   project.stages.qa.agents,
                  user.agents.qa-engineer.agents, project.agents.qa-engineer.agents
extra_commands += user.stages.qa.commands, project.stages.qa.commands,
                  user.agents.qa-engineer.commands, project.agents.qa-engineer.commands
```

Each `commands` entry is either a string (`"notify-slack"`) or an object (`{ "name": "notify-slack", "auto": true }`). Normalize every entry to `{ name, auto }`, with `auto = false` for strings and for objects without `auto`. If a command name appears more than once, the last occurrence (most specific) decides `auto`.

### Mode resolution

`mode` controls only Tier 1 (plugin defaults). It never removes context, stage, role, or agent entries.

- `extend` (default): add the plugin defaults (Tier 1).
- `replace`: do not add the plugin defaults. Every other tier is kept.

Check in order (first non-null value wins):
1. `project.agents.<agent>.mode`
2. `user.agents.<agent>.mode`
3. `project.stages.<stage>.<role>.mode` (if role given)
4. `user.stages.<stage>.<role>.mode` (if role given)
5. Default: `"extend"`

### Scaffold resolution (development stage only)
1. `project.stages.development.<role>.scaffold_if_new`
2. `user.stages.development.<role>.scaffold_if_new`
3. Default: `null`

## Architect: Multiple Roles

The architect designs for every active role, so it is resolved per role instead of with a combined mode.

1. Active roles come from `config.focus`: `fullstack` → `backend`, `frontend`; `backend` → `backend`; `frontend` → `frontend`. Roles that are not active are ignored even if they have configuration.
2. **Shared section**: Tiers 2, 3 and 5 (context, `stages.design`, `agents.architect`), resolved once.
3. **One section per active role**: the mode for that role (resolved with role = that role), Tier 1 if that mode is `extend`, and Tier 4 (`stages.design.<role>`).
4. Never merge the role sections into one list or invent a combined mode. When designing the part of the solution that belongs to a role, the architect applies, in order: that role's plugin defaults → shared context and stage entries → that role's entries → shared agent entries.

## Step 3: Resolve File Paths

### Project search locations

The "project" locations for skills, agents, and commands depend on the workspace:

- `single`, or no role given and not QA: `.claude/{skills,commands,agents}` at the workspace root.
- `monorepo` / `multi-repo` with a role given: first `<workspace_paths.<role>>/.claude/{skills,commands,agents}`, then the workspace root `.claude/{skills,commands,agents}`. The subrepo wins.
- `monorepo` / `multi-repo` for QA (no role): the workspace root first, then each active role's `<workspace_paths.<role>>/.claude/...` (backend before frontend).

For the architect, use the role's search locations when resolving that role's section, and the root-only locations for the shared section.

### Skills

For each name in the merged `skills` list:

1. **Prefixed plugin skill** (`<plugin>:<skill>`, the same name Claude Code uses to invoke a plugin skill): look only inside that plugin.
   - `agent-os:<skill>` → `PLUGIN_ROOT/skills/<skill>/SKILL.md`.
   - Any other `<plugin>` → find its root with [Locating Another Plugin](#locating-another-plugin), then use the first existing file among `<root>/skills/<skill>/SKILL.md`, `<root>/<dir>/<skill>/SKILL.md` for each directory listed in the `skills` field of `<root>/.claude-plugin/plugin.json`, and `<root>/SKILL.md` when `<skill>` equals `<plugin>` (single-skill plugin).
2. **Unprefixed name**: search in order, take the first match:
   a. Project locations (see above): `<location>/.claude/skills/<name>/SKILL.md`
   b. `~/.claude/skills/<name>/SKILL.md` (user)
   c. `PLUGIN_ROOT/skills/<name>/SKILL.md` (plugin)
3. **Not found**: add warning `⚠ skill '<name>' not found — skipped`, continue. For a prefixed name whose plugin is not installed, use `⚠ skill '<plugin>:<skill>' not found (plugin '<plugin>' is not installed) — skipped`. Never stop the pipeline for a missing skill.

The plugin default `tech-adaptation` always resolves to `PLUGIN_ROOT/skills/tech-adaptation/SKILL.md`.

#### Locating Another Plugin

There is no `${CLAUDE_PLUGIN_ROOT}` for other plugins, so their root is looked up on disk. `<plugin>` is the plugin's **manifest name** (`name` in its `.claude-plugin/plugin.json`), which is what Claude Code namespaces skills under. It can differ from the marketplace entry name used for installation and cache folders, so always confirm the manifest name.

The plugins root is `~/.claude/plugins`, unless the `CLAUDE_CODE_PLUGIN_CACHE_DIR` environment variable points elsewhere.

1. **Install registry**: read `<plugins root>/installed_plugins.json`. Claude Code records each install there with its `scope`, `installPath` and `version`, keyed by plugin id `<entry-name>@<marketplace>`. Do not depend on any other part of its layout: collect every record that has an `installPath`, and keep those whose `<installPath>/.claude-plugin/plugin.json` has `"name": "<plugin>"` (if a manifest has no `name`, compare the entry name before `@`).
   - Ignore records whose `installPath` does not exist or contains an `.orphaned_at` marker (a superseded version waiting for cleanup).
   - If several remain, prefer a record whose scope applies to this workspace (a `project` or `local` install for the workspace root) over a `user` install, then the most recent `version` / most recently modified directory.
2. **Fallback glob** (no registry, or no match in it): glob `<plugins root>/cache/*/*/*/.claude-plugin/plugin.json`, `<plugins root>/synced/**/.claude-plugin/plugin.json`, `~/.claude/skills/*/.claude-plugin/plugin.json` and `.claude/skills/*/.claude-plugin/plugin.json`. Keep matches whose `name` is `<plugin>`, drop version directories with an `.orphaned_at` marker, and take the most recently modified one. The plugin root is the directory that contains `.claude-plugin/`.
3. **Not found**: emit the warning from step 3 above and continue.

Resolve each plugin root once per run and reuse it for every skill of that plugin.

### Docs

For each path in the merged `docs` list:
- `~/...` → expand to home directory
- Relative path → resolve against the workspace root (never against a `workspace_paths` subdirectory)
- Absolute path → use as-is

Verify each file exists. If not found: add warning `⚠ doc '<path>' not found — skipped`. Output every doc as an absolute path.

### Extra Agents (QA only)

For each agent name: check the project locations (`<location>/.claude/agents/<name>.md`), then `~/.claude/agents/<name>.md`. Not found → warning.

### Commands (QA only)

For each command name: a prefixed `agent-os:<command>` is a built-in command, `PLUGIN_ROOT/commands/<command>.md`. Otherwise check the project locations (`<location>/.claude/commands/<name>.md`), then `~/.claude/commands/<name>.md`. Not found → warning. To run a command, read its file and follow it.

**Every post-QA command requires an approval gate by default.** Tag it `[gate]` unless its normalized entry has `auto: true`, in which case tag it `[auto]`. Never infer `auto` from the command's description, frontmatter, or name — a command without frontmatter is still gated.

### Scaffold (development stage only)

If `scaffold_if_new` is non-null and `config.project_type = "new"`: resolve the skill name using the same lookup order. If not found: warning, skip.

## Step 4: Produce Output

All paths in the output are absolute.

```
EXTENSION RESOLUTION
====================
Stage:  [stage]
Role:   [role or "—"]
Agent:  [agent name]
Mode:   [extend | replace]
Order:  generic → specific. Later entries override earlier ones on conflict; the project's CLAUDE.md overrides all.

Skills (application order):
  [/absolute/path/to/tech-adaptation/SKILL.md]  [tier: plugin]  [level: plugin]
  [/absolute/path/to/SKILL.md]   [tier: stage|role|agent]  [level: user|project]

Docs (read order):
  [/absolute/path/to/file.md]    [tier: context|stage|role|agent]  [level: user|project]

[QA only:]
Extra agents:     [name ...]
Post-QA commands: [name [gate] | name [auto] ...]

Scaffold: [/absolute/path/to/SKILL.md | none]

Warnings:
  ⚠ skill 'x' not found — skipped
====================
```

For the architect, replace the single `Mode` / `Skills` / `Docs` lists with:

```
Order:  generic → specific. Later entries override earlier ones on conflict; the project's CLAUDE.md overrides all.

### Shared
Skills: [...]  (tiers: stage, agent)
Docs:   [...]  (tiers: context, stage, agent)

### Role: backend
Mode:   [extend | replace]
Skills: [...]  (tiers: plugin, role)
Docs:   [...]  (tier: role)

### Role: frontend
Mode:   [extend | replace]
Skills: [...]
Docs:   [...]
```

If nothing is configured at any level and there are no warnings:

```
EXTENSION RESOLUTION
====================
Stage: [stage] | Agent: [agent]
No extensions configured. Applying plugin defaults.
Skills: [/absolute/path/to/tech-adaptation/SKILL.md]  [tier: plugin]  [level: plugin]   ← only for architect / developers
The project's CLAUDE.md overrides all.
====================
```

## Step 5: Build the Agent Prompt

The agent must read its inputs from disk, so the prompt carries **references, not content**. It contains only:

- The `## Extensions` block exactly as produced in Step 4.
- Paths: `.agent-os/config.json`, the task file, the ADR files that apply (`.agent-os/decisions/DEC-NNN.json`), and `working_directory` when applicable.
- Development only: the subtask IDs assigned to this run, in execution order, each with its title on one line.
- Discovery only: the user's task description, verbatim (not for the `approver`: it reads `description` from the task file).
- Approver only: `Review target: requirements | planning | design | questions | scope-change` and, except for `questions`, the absolute path of the workflow of the stage under review.
- Revision runs only: the revision line from the `approval-gates` skill (Revision Runs), naming one source: the review round, or one change request `CR-<k>`. Approver findings and the user's change requests are recorded in the task file and read from there.
- Discovery answers runs only: `Answers: read open_questions in the task file.` (Clarifying Questions Protocol in `01-discovery.md`). Never the answers themselves.
- Development QA revisions only (unattended mode): `QA revision <n>: fix every critical and major issue in stages.qa.issues of the task file.`
- Decisions the agent cannot find in any file, stated briefly.

Never copy into the prompt the content of the design, ADRs, requirements, subtask details beyond their titles, config values, rules from any `CLAUDE.md`, or the content of skills and docs. A prompt that already contains them gives the agent a reason to skip its files and its extensions.

Never read test plans, checklists or expected-results files from the workspace to pass hints to agents.

## Step 6: Verify Extensions Applied

Every agent-os agent ends its report with an `EXTENSIONS APPLIED` section listing each Skills and Docs path it read, with one line on how it applied it (or `not applicable: <reason>`). After each agent returns:

1. Compare that section with the resolution's Skills and Docs paths (for the architect, the paths of every section). When there was no `## Extensions` block, the expected path is the plugin default (tech-adaptation) for the architect and developers, and nothing for the rest.
2. If the section is missing, or a resolved path is not in it, tell the user `⚠ <agent> did not confirm applying: <missing paths | "EXTENSIONS APPLIED section missing">` and append the same text to `stages.<stage>.extension_warnings` in the task file.
3. Do not block the stage and do not re-run the agent automatically. A path marked `not applicable` counts as read; mention it in the stage summary.

Extra QA agents from extensions are not agent-os agents: do not verify their reports.

## Security Rules

1. Post-QA commands tagged `[gate]` are NEVER run automatically. The calling command presents an approval gate (approval-gates skill) before executing each one. Only commands explicitly configured with `auto: true` run without approval.
2. Skills that require user input: the calling command must supply that input from the task file, because subagents cannot prompt the user.
3. `scaffold_if_new` applies only when `project_type = "new"`.
4. If a resolved skill contradicts the project's `CLAUDE.md`, `CLAUDE.md` wins. In `monorepo` / `multi-repo`, a subrepo's `CLAUDE.md` wins over the root `CLAUDE.md` for that role. Mention the conflict once in the output.
