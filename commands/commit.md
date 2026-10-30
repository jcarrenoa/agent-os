---
description: Commit the changes of the active (or last finished) task, one local commit per repo, on a feature branch named by the project's git conventions. Never pushes. Can run as a post-QA command.
argument-hint: "[ISSUE-KEY] (e.g. VR-123; defaults to the task's tracker key, then its TASK-NNN ID)"
allowed-tools: [Bash, Read, Write]
---

# Agent OS — Commit

Create one local commit per repo with the changes of the current task, following `config.git`. **Never run `git push`**, in any form, for any reason: the user pushes.

Never use `--no-verify`, `--amend`, `--no-gpg-sign`, `git stash`, `git reset --hard` or `git checkout -- <path>`. Never commit on a protected branch.

## Step 1: Read Config and Task

1. Read `.agent-os/config.json`. If it does not exist: tell the user to run `/agent-os:setup` first, and stop.
2. Read `.agent-os/state.json` → `active_task_id`, then `.agent-os/tasks/<id>.json`. If there is no active task, use the most recent task file with `status = "done"` and say so in the report. If there is none, stop and say so.
3. Repos:
   - `multi-repo`: each path in `config.workspace_paths` (key = role, e.g. `backend`), relative to the workspace root.
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

## Step 2: Commit Each Repo

For each repo, run every git command as `git -C <repo> ...`:

1. `rev-parse --is-inside-work-tree` fails → skip the repo, note "no es un repo git".
2. `status --porcelain=v1 --untracked-files=all` is empty → skip the repo, note "sin cambios".
3. **Branch.** Read `branch --show-current`. `<target>` = `branch_pattern` with the placeholders filled. "This task's branch" is a branch that matches `branch_pattern` with this `{key}` (any `{slug}`), or contains the key as a whole word after a `/` (`temp/VR-91-dev`). A protected branch matches an entry of `protected_branches` exactly or by its `*` glob (`release/*`).
   - This task's branch → commit there.
   - A branch that matches `branch_pattern` with another key (another task) → `switch -c <target>` from the current HEAD, and warn that it is stacked on that branch: its PR will carry that task's commits unless that branch is merged first.
   - `base_branch` → `switch -c <target>`. The uncommitted changes move with it, and the branch is born from `base_branch`.
   - Any other branch, protected or not → `switch -c <target>` from the current HEAD, and warn that the branch was not born from `base_branch` (when one is configured).
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
{ "repo": "backend", "branch": "feature/VR-91-expand-contact-group-apis", "hash": "a1b2c3d", "subject": "VR-91 | feat: Expand contact and group APIs", "files": ["src/Contacts/ContactController.cs"], "subtasks": ["SUB-01", "SUB-02"] }
```

`repo` is the repo path (`"."` for the root), `hash` the short hash (`rev-parse --short HEAD`), `files` the committed paths (`show --name-only --format= HEAD`). Change no other field. The tracker's Comment Commits (`tracker-sync` skill) reads this list.

## Step 4: Report

In `preferences.output_language`:

```markdown
### 📦 Commits por repo
| Repo | Rama | Commit | Asunto | Archivos | Nota |
|---|---|---|---|---|---|
| backend | feature/VR-91-expand-contact-group-apis | a1b2c3d | VR-91 \| feat: Expand contact and group APIs | 6 | — |
| frontend | — | — | — | 0 | sin cambios |

Nada se ha subido al remoto. Para publicar: `git -C <repo> push -u origin <rama>`.
```

Under the table, list every warning: stacked branches, branches not born from `base_branch`, files excluded as secrets or tooling, pre-task changes left out or mixed in, type fallbacks, and hook output.
