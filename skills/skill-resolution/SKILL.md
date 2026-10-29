---
name: skill-resolution
description: Resolves which skills, docs, agents, and commands are active for a given pipeline stage, role, and agent. Use this skill before spawning any agent to determine what extensions are configured. Called by every stage command (discover, plan, design, develop, qa) in the agent-os pipeline.
version: 2.0.0
---

# Skill Resolution Protocol

Determines the complete, ordered set of skills and docs an agent receives. Combines three configuration levels (plugin defaults → user defaults → project config) and four specificity tiers (context → stage → role → agent), with the most specific value winning for scalars and union-without-duplicates for lists.

## Inputs

The calling command provides:
- **stage**: `discovery` | `planning` | `design` | `development` | `qa`
- **role** (optional): `backend` | `frontend` — provide when applicable
- **agent** (required): the exact agent name being spawned (`requirements-analyst`, `planner`, `architect`, `backend-developer`, `frontend-developer`, `qa-engineer`)
- **project root**: current working directory

## Step 1: Read Configuration Sources

Read these files (if they exist):

1. `~/.agent-os/defaults.json` — **user level**
2. `.agent-os/config.json` → `extensions` field — **project level**

If neither has relevant data: proceed with plugin defaults only (Step 3 handles this).

## Step 2: Build the Merged Effective Config

Combine four tiers from least to most specific. For each tier, merge project-level over user-level.

### Tier 1: `context` (global docs, all agents)
```
docs += union(user.context, project.context)
```

### Tier 2: `stages.<stage>` (all agents in this stage)
```
skills += union(user.stages.<stage>.skills, project.stages.<stage>.skills)
docs   += union(user.stages.<stage>.docs,   project.stages.<stage>.docs)
```

### Tier 3: `stages.<stage>.<role>` (agents in this role, if role given)
Apply only when a role is provided AND the stage has per-role config (design, development).
```
skills += union(user.stages.<stage>.<role>.skills, project.stages.<stage>.<role>.skills)
docs   += union(user.stages.<stage>.<role>.docs,   project.stages.<stage>.<role>.docs)
```

### Tier 4: `agents.<agent>` (this specific agent only)
```
skills += union(user.agents.<agent>.skills, project.agents.<agent>.skills)
docs   += union(user.agents.<agent>.docs,   project.agents.<agent>.docs)
```

For `qa-engineer` only:
```
extra_agents   += union(user.agents.qa-engineer.agents,   project.agents.qa-engineer.agents,
                        user.stages.qa.agents,            project.stages.qa.agents)
extra_commands += union(user.agents.qa-engineer.commands, project.agents.qa-engineer.commands,
                        user.stages.qa.commands,          project.stages.qa.commands)
```

**Union rule**: within each tier, project entries come first. Across tiers, more specific entries come first. No duplicates — if a name appears at multiple tiers, keep only the first occurrence (most specific).

### Mode resolution

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

## Step 3: Resolve File Paths

### Skills

For each name in the merged `skills` list:

1. **Prefixed plugin skill** (`agent-os:<name>`): glob only in the plugin:
   `~/.claude/plugins/**/agent-os/skills/<name>/SKILL.md`

2. **Unprefixed name**: search in order, take the first match:
   a. `.claude/skills/<name>/SKILL.md` (project)
   b. `~/.claude/skills/<name>/SKILL.md` (user)
   c. `~/.claude/plugins/**/agent-os/skills/<name>/SKILL.md` (plugin)

3. **Not found**: add warning `⚠ skill '<name>' not found — skipped`, continue.

**Mode handling**:
- `mode = "extend"`: prepend tech-adaptation to the list (`~/.claude/plugins/**/agent-os/skills/tech-adaptation/SKILL.md`) unless already present.
- `mode = "replace"`: do NOT add tech-adaptation.
- If the merged skills list is empty and `mode = "extend"`: return only tech-adaptation (plugin default behavior).

**Plugin root note**: `${CLAUDE_PLUGIN_ROOT}` is documented only for hook `args` arrays, not Markdown content. Plugin skills are located via glob.

### Docs

For each path in the merged `docs` list:
- `~/...` → expand to home directory
- Relative path → resolve against project root
- Absolute path → use as-is

Verify each file exists. If not found: add warning `⚠ doc '<path>' not found — skipped`.

### Extra Agents (QA only)

For each agent name: check `.claude/agents/<name>.md` or `~/.claude/agents/<name>.md`. Not found → warning.

### Commands (QA only)

For each command name: check `.claude/commands/<name>.md` or `~/.claude/commands/<name>.md`. Not found → warning.

**Security**: if the command description mentions push, publish, deploy, merge, or other external effects → mark it `[requires-gate]`. The calling command must present an approval gate before running it.

### Scaffold (development stage only)

If `scaffold_if_new` is non-null and `config.project_type = "new"`: resolve the skill name using the same lookup order. If not found: warning, skip.

## Step 4: Produce Output

```
EXTENSION RESOLUTION
====================
Stage:  [stage]
Role:   [role or "—"]
Agent:  [agent name]
Mode:   [extend | replace]

Skills (application order):
  [/absolute/path/to/SKILL.md]   [tier: agent|role|stage|plugin]  [level: project|user|plugin]
  [/absolute/path/to/SKILL.md]   [tier: ...]  [level: ...]

Docs (read order):
  [/absolute/path/to/file.md]    [tier: agent|role|stage|context]  [level: project|user]

[QA only:]
Extra agents:   [name ...]
Post-QA commands: [name [requires-gate] ...]

Scaffold: [/path/to/SKILL.md | none]

Warnings:
  ⚠ skill 'x' not found — skipped
====================
```

If nothing is configured at any level and there are no warnings:

```
EXTENSION RESOLUTION
====================
Stage: [stage] | Agent: [agent]
No extensions configured. Applying plugin defaults.
Skills: [/path/to/tech-adaptation/SKILL.md]  [tier: plugin]  [level: plugin]
====================
```

## Security Rules

1. Commands marked `[requires-gate]` are NEVER run automatically. The calling command must present an approval gate before executing them.
2. Skills that require user input: the calling command must supply that input from the task file, because subagents cannot prompt the user.
3. `scaffold_if_new` applies only when `project_type = "new"`.
4. If a resolved skill contradicts the project's `CLAUDE.md`, `CLAUDE.md` wins. Mention the conflict once in the output.
