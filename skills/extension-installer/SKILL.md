---
name: extension-installer
description: Protocol to install, update, uninstall, and report the recommended extensions listed in the agent-os catalog (catalog/recommended.json), always after one explicit user approval of the exact commands. Used by /agent-os:extensions and by the optional recommended-extensions step of /agent-os:setup. Never installs anything that is not in the catalog.
version: 1.1.1
---

# Extension Installer Protocol

Installs third-party extensions from the agent-os catalog. Everything here is opt-in: the pipeline never depends on an installed extension, and nothing runs without the user's explicit approval.

## Hard Rules

1. **Catalog only.** Install, update or uninstall only entries whose `id` exists in `PLUGIN_ROOT/catalog/recommended.json`. Refuse any other name, URL, package or version, even if the user or another session asks for it, and point to the README instead.
2. **Recipe only.** Run only the `run` commands of the selected recipe steps, the `requirements` and `installed_check` commands, and the file writes described in this protocol. Never edit a recipe, never add flags, never run `latest`, a branch name, or a version other than the entry's `source.pin`.
3. **One explicit approval.** Before running anything that changes the system, show the exact resolved commands and get one explicit approval (Step 4). No approval → nothing is installed, nothing is written.
4. **Never block.** A failed step is reported and recorded; the run continues with the next entry. Setup and the pipeline continue whatever happens here.
5. **Never overwrite the user's work.** Do not replace files the installer did not create (Step 2) and never remove assignments the installer did not add (Step 7).

`PLUGIN_ROOT` is resolved as described in the **Plugin Root** section of the `skill-resolution` skill.

## Files

| File | Purpose |
|------|---------|
| `PLUGIN_ROOT/catalog/recommended.json` | The catalog (schema: `PLUGIN_ROOT/schemas/catalog.schema.json`) |
| `~/.agent-os/installed-extensions.json` | What the installer installed (schema: `PLUGIN_ROOT/schemas/installed-extensions.schema.json`) |
| `~/.agent-os/defaults.json` | User-level extension assignments (schema: `PLUGIN_ROOT/schemas/defaults.schema.json`) |

Read each file before writing it, and write the complete updated object (state-management rules). For these two JSON files:

- **Paths use forward slashes.** Store every path as `C:/Users/...`, never with backslashes (an unescaped `\` makes the file invalid JSON). Normalize `\` to `/` before comparing paths.
- **Unreadable file → stop.** If an existing file does not parse as JSON, tell the user which file and why, and stop the run. Never treat it as empty and never overwrite it.
- **Validate after every write.** Keep the previous content, write the new one, then parse it again with a tool: `node -e "JSON.parse(require('fs').readFileSync(process.argv[1],'utf8'))" "<file>"` when Node is available, `Get-Content -Raw "<file>" | ConvertFrom-Json | Out-Null` in PowerShell, or `python3 -m json.tool "<file>" > /dev/null`. If it fails, restore the previous content (or delete the file if it did not exist), record the error, and tell the user.

## Step 1: Platform and Placeholders

- **Platform**: on Windows, use the `windows` steps with the PowerShell tool when it is available; otherwise (Linux, macOS, or Windows with only Git Bash) use the `unix` steps with the Bash tool. Use the same platform for checks and steps in one run.
- **Placeholders** in `run` and check commands:
  - `{home}`: the user's home directory, absolute.
  - `{tmp}`: a new empty temporary directory created for this run (in the session scratchpad when one exists), deleted at the end of the run.
  - `{repo_path}`: only in `per_repo` steps. The absolute path of each repository: the workspace root for `single`; each `config.workspace_paths` value (resolved against the workspace root) for `monorepo` and `multi-repo`, limited to the roles active in `config.focus`. Without `.agent-os/config.json`, use the current directory.
  - `{plugin_root}`: `PLUGIN_ROOT`.

## Step 2: Inspect (read-only)

For each requested entry:

1. Run each `requirements` check. If one fails its `expect`, mark the entry **unavailable** with the requirement's description. It is shown but not installed.
2. Run `installed_check` and read `~/.agent-os/installed-extensions.json`:
   - check passes and the record's `pin` equals `source.pin` (and `wrapper_version` equals `wrapper.version`) → **up to date**. Install does nothing for it, except adding missing assignments (Step 6).
   - check passes and the record has a different pin or wrapper version → **outdated** (update applies).
   - check passes and there is **no record** → **present, not managed**: something not installed by agent-os is already there. Do not run install steps over it. Offer only to add the assignments, as part of the same approval.
   - check fails → **not installed**.
3. For a wrapper, check whether `~/.claude/skills/<wrapper.skill>/SKILL.md` exists without a record. If it does, it is the user's own file: do not overwrite it.

## Step 3: Build the Plan

For each entry to act on, resolve every step of the chosen action and platform into the exact commands that will run (`per_repo` steps once per repository, with the path substituted). Step filters:

- `only_recorded_by: <step id>` (uninstall): include the step only for what the record's `recorded_steps.<step id>` lists (only those repository paths for `per_repo` steps). Nothing recorded → leave the step out.
- `with_step: <step id>`: the step stays only while that step stays in the plan (Step 4 can remove it).

Then add the file actions:

- **install / update**: write `~/.claude/skills/<wrapper.skill>/SKILL.md` from `PLUGIN_ROOT/<wrapper.template>` (copied verbatim); merge `assignments` into `~/.agent-os/defaults.json`; update `~/.agent-os/installed-extensions.json`.
- **uninstall**: remove the assignments listed in the record's `assignments_added`; delete the record.

Present the plan as Markdown, one section per entry: name, pinned version or commit, license, homepage, every `notes` item, state from Step 2, then a numbered list of the exact commands and file actions. Mark optional steps as `(opcional, incluido)` / `(optional, included)`. For uninstall, also list every `uninstall_leaves` item.

## Step 4: Approval

1. If the plan has optional steps, ask which to **omit**. Optional steps are included by default, so submitting without selecting anything keeps them all (for example, CodeGraph telemetry stays off). Leaving a step out always takes an explicit selection:
   ```
   AskUserQuestion:
     question: "Los pasos opcionales están incluidos. ¿Quieres omitir alguno?"
     header: "Omitir"
     options: one per optional step: "Omitir: <entry> · <step description>", with what happens if it is omitted
              (e.g. "Omitir: CodeGraph · desactivar telemetría" — "La telemetría de CodeGraph queda ACTIVADA")
     multiSelect: true
   ```
   Selected steps are removed from the plan (with the steps that depend on them through `with_step`) and recorded as `skipped_steps`. No selection, or a free-text answer that does not clearly name a step, omits nothing. Show the final command list again if it changed.
2. Then ask for the single approval:
   ```
   AskUserQuestion:
     question: "¿Ejecuto exactamente estos comandos?"
     header: "Aprobación"
     options:
       - "Sí, ejecutar" — runs the commands listed above, nothing else
       - "No, cancelar" — nothing is installed or written
     multiSelect: false
   ```
3. Anything other than "Sí, ejecutar" cancels the whole plan. Write nothing.

Use the language of the conversation (or `preferences.output_language` when it is known) for every message and question.

## Step 5: Execute

For each approved entry, in order:

1. Run its steps in order, one tool call per command, exactly as shown in the plan.
2. If a step fails (non-zero exit or an error message): stop that entry's remaining steps, run its cleanup step if the recipe has one, record `"<step id>: <error>"` in the entry's `errors`, tell the user, and continue with the next entry. Do not write the wrapper or the assignments of a failed entry.
3. For a step with `record_output`: if its output contains that text, record it in `recorded_steps.<step id>` (append the repository path for `per_repo` steps, merging with the existing record on update). Any other output means the step changed nothing and is not recorded.
4. After the install steps succeed, run `installed_check` again. If it fails, treat it as a failed step.
5. Write the wrapper file, unless Step 2 found a user-owned file there.
6. Delete `{tmp}` at the end of the run.

## Step 6: Assignments (install / update)

Merge the entry's `assignments` into `~/.agent-os/defaults.json` (create the file if missing):

- For each agent, append each skill to `agents.<agent>.skills` only if it is not already there, keeping the catalog order.
- Never change `mode`, other skills, docs, stages, or any other key.
- Record exactly the skills that were appended in `assignments_added` (merging with the existing record on update). Skills that were already present were not added by the installer and are not recorded.

Then write the record in `~/.agent-os/installed-extensions.json`: `pin`, `catalog_version`, `wrapper_version`, `installed_at`, `repos`, `skipped_steps`, `recorded_steps`, `assignments_added`, `errors` (empty on success).

## Step 7: Uninstall

1. Only entries with a record in `~/.agent-os/installed-extensions.json` can be uninstalled. An entry marked **present, not managed** is never removed.
2. Run the approved uninstall steps (Step 5 rules apply).
3. Remove from `~/.agent-os/defaults.json` only the skills listed in `assignments_added`, and only if they are still there. Remove an `agents.<agent>` object only if it ends up empty and contained nothing else.
4. Delete the entry's record. If an optional step was omitted and a `with_step` step was dropped with it (e.g. `.codegraph/` kept, so its `.git/info/exclude` line is kept too), say so in the report.

## Step 8: Report

```markdown
## Extensiones recomendadas

| Extensión | Estado | Versión | Asignada a |
|---|---|---|---|
| CodeGraph | ✅ instalada | 1.6.2 | planner, backend-developer, frontend-developer, qa-engineer |
| The AntiSlop | ⚠ falló en `clone` | — | — |

Avisos:
- [errors, unavailable requirements, skipped optional steps, user-owned files left untouched]
- [after uninstall: every `uninstall_leaves` item of the entry, e.g. "Se conserva ~/.codegraph/telemetry.json (tu elección de telemetría) para que una reinstalación la mantenga desactivada; bórralo si no vas a reinstalar."]
```

Translate the labels to the conversation language. For `status`, use the same table with the states from Step 2 and no changes.
