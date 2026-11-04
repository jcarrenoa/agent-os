---
description: Commit the changes of the active (or last finished) task, one local commit per repo, on a feature branch named by the project's git conventions. Never pushes. Can run as a post-QA command. With a tracker, run directly it also offers the commits comment, always with approval.
argument-hint: "[ISSUE-KEY] (e.g. VR-123; defaults to the task's tracker key, then its TASK-NNN ID)"
allowed-tools: [Bash, Read, Write, Edit, AskUserQuestion]
---

# Agent OS — Commit

Create one local commit per repo with the changes of the current task, following `config.git`. **Never run `git push`**, in any form, for any reason: the user pushes.

Never use `--no-verify`, `--amend`, `--no-gpg-sign`, `git stash`, `git reset --hard` or `git checkout -- <path>`. Never commit on a protected branch.

## Step 1: Read Config and Task

1. Read `.agent-os/config.json`. If it does not exist: tell the user to run `/agent-os:setup` first, and stop.
2. Read `.agent-os/state.json` → `active_task_id`, then `.agent-os/tasks/<id>.json`. If there is no active task, use the most recent task file with `status = "done"` and say so in the report. If there is none, stop and say so.
3. Repos:
   - `multi-repo`: each value of `config.workspace_paths`, a path relative to the workspace root (e.g. `reach-backend`). Its key is the repo's role (`backend`, `frontend`); the repo itself is always named by its path.
   - `single` or `monorepo`: one repo, the workspace root (key `"."`). In a `monorepo`, list every subtask in the body.
4. Conventions of each repo: the fields of `config.git`, overridden field by field by `config.git.repos.<repo path>`. Missing fields take these defaults:

   | Field | Default |
   |---|---|
   | `base_branch` | none (branch from the current HEAD) |
   | `branch_pattern` | `feature/{key}-{slug}` |
   | `commit_pattern` | `{type}: {summary}` |
   | `commit_pattern_no_key` | `{type}: {summary}` |
   | `commit_types` | `feat`, `fix`, `chore`, `refactor`, `docs`, `test` |
   | `commit_language` | `en` |
   | `protected_branches` | `main`, `master` |

5. Placeholders:
   - `{key}`: first match wins: `$ARGUMENTS` when it looks like an issue key (`[A-Z][A-Z0-9_]+-[0-9]+`); `task.external.key`; the task ID (`TASK-NNN`). When the key is the task ID (no issue key), commit subjects use `commit_pattern_no_key`.
   - `{task}`: the task ID.
   - `{type}`: `task.change_type`. If it is missing or not in `commit_types`, use the closest listed type (`refactor`, `docs`, `test` → `chore` when listed, otherwise the first type) and say so in the report.
   - `{summary}`: the task title as a short imperative sentence in `commit_language` (`Expand contact and group APIs`), with no final period. Shorten it so the whole subject line is at most 72 characters.
   - `{scope}`: the repo role (`backend`, `frontend`), or the module most of the changes are in for a single repo.
   - `{slug}`: the task title in kebab-case, ASCII only, at most 5 words (`expand-contact-group-apis`).
6. `<base>` of each repo: with `epic_branches: true` and `task.external.epic`, the epic branch (**Epic Branch** below; here only steps 1 and 2, nothing is switched or updated); otherwise `base_branch`.

## Epic Branch

Used here and by `/agent-os:start` when a task is created. Only with `config.git.epic_branches: true` (or the repo's override) and `task.external.epic`. It decides the branch the task's branch is born from: the rule is to use it when it is clear and to ask when it is not, never to guess.

1. **Known**: `config.git.epic_branch_map[<epic key>]`, when it exists as a local or remote branch in the repos → that branch.
2. **Found**: list the branches of each repo (`branch -a --format=%(refname:short)`, without `origin/`, deduplicated). A branch is a match when its name, without a leading `feature/`, equals the epic key or the epic summary in kebab-case ASCII (`Contactos` → `contactos`). Exactly one match, the same in every repo that has one → that branch, said in one line: "Rama de la épica <summary> (<key>): `<branch>`".
3. **Ask** in any other case: no match, several, or different ones per repo. AskUserQuestion, header "Épica", in `output_language`: "¿Desde qué rama sale <task key>? Es de la épica <summary> (<epic key>).". Options: up to 3 candidates, ordered by how close their name is to the summary or the key and then by most recent commit, among the branches that are not protected, are not `base_branch` and do not match `branch_pattern` with any key, each with its last commit date; and "`<base_branch>` (sin rama de épica)". The user can type another name. Save the answer in `config.git.epic_branch_map[<epic key>]` (read the file first, change only that key), unless it is `base_branch`, so it is not asked again for the next story of the epic.
4. **Update and check out** (only from `/agent-os:start`, before the git baseline is recorded). For each repo, read-only first: `fetch` the remote, then compare the epic branch with `origin/<base_branch>`. Show one table (`Repo | Rama actual | Rama de la épica | Le faltan de <base_branch> | Cambios sin commitear`) and ask once, header "Ramas": "¿Preparo los repos en `<epic branch>` actualizada con `<base_branch>`? Solo en local, nunca hace push.": "Sí, preparar (Recomendado)" / "No, seguir como están".
   - Yes, per repo: if the epic branch only exists remotely, create it tracking `origin/<branch>`. `switch` to it; if git refuses because of uncommitted changes, stop for this repo and report it, never forcing. Then update it from `origin/<base_branch>`: `merge --ff-only` when possible; otherwise `merge --no-edit origin/<base_branch>`, and if it conflicts, `merge --abort` and report that the merge must be done by hand. Never `reset`, `rebase` or `push`.
   - Report per repo what was done, and that the epic branch was updated only locally: to share the update, `git -C <repo> push`.
   - No: continue on the current branches. `/agent-os:commit` warns later if the task's branch is not born from the epic branch.

Without an epic, or with `epic_branches` off, `<base>` is `base_branch` and nothing here applies.

## Step 2: Commit Each Repo

For each repo, run every git command as `git -C <repo> ...`:

1. `rev-parse --is-inside-work-tree` fails → skip the repo, note "no es un repo git".
2. `status --porcelain=v1 --untracked-files=all` is empty → skip the repo, note "sin cambios".
3. **Branch.** Read `branch --show-current`. `<target>` = `branch_pattern` with the placeholders filled. "This task's branch" is a branch that matches `branch_pattern` with this `{key}` (any `{slug}`), or contains the key as a whole word after a `/` (`temp/VR-91-dev`). A protected branch matches an entry of `protected_branches` exactly or by its `*` glob (`release/*`).
   - This task's branch → commit there.
   - `<base>` (Step 1.6: the epic branch, or `base_branch`) → `switch -c <target>`. The uncommitted changes move with it, and the branch is born from `<base>`. Checked before the next rule, because an epic branch can be named like a key (`VR-12`).
   - A branch that matches `branch_pattern` with another key (another task) → `switch -c <target>` from the current HEAD, and warn that it is stacked on that branch: its PR will carry that task's commits unless that branch is merged first.
   - Any other branch, protected or not → `switch -c <target>` from the current HEAD, and warn that the branch was not born from `<base>` (when one is configured).
   - If `<target>` already exists: `switch <target>` instead of creating it. If git refuses because it would overwrite local changes, stop for this repo and report it. Never force the switch.
   - Detached HEAD → create `<target>` from it and warn.
4. **Stage.** With `task.git_baseline.<repo>`, stage only what the task changed. For each path in the status output:
   - not in `git_baseline.<repo>.dirty` → stage it (`add -- <path>`; a deleted path with `add -A -- <path>`);
   - in `dirty` with a different `git hash-object -- <path>` (or deleted now, or present now when the baseline blob was `null`) → stage it, and warn that it mixes work from before the task;
   - in `dirty` with the same blob → leave it unstaged and list it as "cambios previos a la tarea, no incluidos".

   Without a baseline: `add -A`, and warn that the commit may include uncommitted work from before the task.
5. **Exclude.** Unstage (`restore --staged -- <path>`) and list every staged path that is:
   - a secret: `.env`, `.env.*`, `*.pem`, `*.key`, `*.pfx`, `*.p12`, a file named `secrets*` or `credentials*`;
   - local settings: `appsettings.*.json` other than `appsettings.json`;
   - tooling: anything under `.agent-os/`, `.codegraph/`, `node_modules/`, `bin/`, `obj/`.

   If nothing is left staged, skip the repo and report it.
6. **Commit** with `commit -F <tempfile>` (a temp file outside the repo) and this message in `commit_language`:

   ```
   <subject from commit_pattern or commit_pattern_no_key>

   - SUB-01 <subtask title>
   - SUB-02 <subtask title>

   Agent OS <TASK-NNN>
   ```

   Body lines: the subtasks of this repo's role (`backend` → `assigned_to = "backend-developer"`, `frontend` → `"frontend-developer"`); every subtask for a `single` or `monorepo` repo. Omit the list when the task has no subtasks.

   If the commit fails (a hook rejects it, signing fails): leave the changes staged, keep the hook output for the report, and continue with the next repo. Do not retry with other flags.

## Step 3: Record the Commits

Read the task file again and set `commits` to the entries of earlier runs (keep those whose `hash` differs) plus one per commit created now:

```json
{ "repo": "reach-backend", "branch": "feature/VR-91-expand-contact-group-apis", "hash": "a1b2c3d", "subject": "VR-91 | feat: Expand contact and group APIs", "files": ["src/Contacts/ContactController.cs"], "subtasks": ["SUB-01", "SUB-02"] }
```

`repo` is the repo path, the same value as the `git_baseline` key (`"reach-backend"`; `"."` for the root), never the role, `hash` the short hash (`rev-parse --short HEAD`), `files` the committed paths (`show --name-only --format= HEAD`), `subtasks` the IDs of the subtasks listed in that commit's body (Step 2.6; `[]` when the task has none). Change no other field. The tracker's Comment Commits (`tracker-sync` skill) reads this list. If the task file cannot be written, say so in the report and list the hashes: those commits cannot be commented until they are recorded.

## Step 4: Report

In `preferences.output_language`:

```markdown
### 📦 Commits por repo
| Repo | Rama | Commit | Asunto | Archivos | Nota |
|---|---|---|---|---|---|
| reach-backend | feature/VR-91-expand-contact-group-apis | a1b2c3d | VR-91 \| feat: Expand contact and group APIs | 6 | — |
| reach-frontend | — | — | — | 0 | sin cambios |

Nada se ha subido al remoto. Para publicar: `git -C <repo> push -u origin <rama>`.
```

Under the table, list every warning: stacked branches, branches not born from `base_branch`, files excluded as secrets or tooling, pre-task changes left out or mixed in, type fallbacks, and hook output.

## Step 5: Tracker Comment

Only with `config.tracker`. Do these in order:

1. Read `.agent-os/state.json` and the task file again. If `current_stage = "qa"`, `active_task_id` is this task and `stages.qa.status = "done"`, this run is a post-QA command: stop here, and do not show or offer any comment. The calling command runs Comment Commits right after its post-QA commands.
2. Otherwise, read `${CLAUDE_PLUGIN_ROOT}/skills/tracker-sync/SKILL.md`. If that path is not absolute (it still starts with `${`), resolve the plugin root with the **Plugin Root** section of the `skill-resolution` skill first.
3. Run **Comment Commits** of that skill as Manual, for this task, also when this run created no commit: earlier commits may still be pending. It always asks before writing, and a failure is recorded for `/agent-os:tracker sync` without affecting the commits already made. Only with the `comment` permission (**Permissions** in that skill); without it, nothing is offered.
