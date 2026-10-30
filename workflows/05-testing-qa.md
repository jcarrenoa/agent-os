# Workflow: Stage 5 — Testing & QA

## Purpose

Independently verify that the implementation correctly satisfies all approved requirements, with no regressions and acceptable code quality.

## Entry Conditions

- `stages.development.status = "done"`.
- `task.files_modified` is populated.
- `state.json.current_stage = "qa"` and `stage_status = "in_progress"`.

## Exit Conditions

- [ ] The test suite has been run and results recorded.
- [ ] All `must` requirements are verified as covered.
- [ ] No critical issues remain (or user has explicitly accepted them).
- [ ] A QA report is recorded in the task file.

## The Agent's Job

Before spawning the agent, the calling command follows the `skill-resolution` skill for stage `qa` and passes the result under `## Extensions`. If the resolution includes extra Agents, they are spawned in parallel and their findings are merged. Post-QA Commands are run after APPROVE. Every command requires an approval gate unless it is explicitly configured with `auto: true` (tagged `[auto]`).

The `qa-engineer` agent independently reviews:
1. Requirements coverage: is each requirement implemented and tested?
2. Test suite: does it pass?
3. Code quality: correctness, security, edge cases in modified files.
4. Architecture compliance: does the implementation match Stage 3's design?
5. Regressions: are existing tests still passing?

The agent returns a QA report with a recommendation: `APPROVE` or `REQUEST_CHANGES`.

## Recommendation Criteria

### APPROVE when:
- All `must` requirements are covered.
- Test suite passes (or pre-existing failures exist and are documented).
- No critical security issues.
- No critical regressions.

### REQUEST_CHANGES when:
- Any `must` requirement is not implemented.
- Critical security issues found (hardcoded secrets, SQL injection, missing auth).
- Critical regressions (existing tests broken by new code).
- Data migration risks not handled.

Minor and major issues should be documented but do not block approval.

## Decision Tree After QA Report

```
QA Recommendation?
├── APPROVE
│   └── Close task as done. Show completion summary.
│
└── REQUEST_CHANGES
    ├── Unattended approver mode → QA Retries (approval-gates skill):
    │   re-run development with the issues up to max_revisions, then
    │   critical left → blocked · only major/minor → close with unresolved_findings
    └── Otherwise: present issues to user. Ask:
        ├── "Fix now" → rollback development stage, re-run develop + qa
        ├── "Accept and close" → user accepts risk, close with QA notes
        └── "New task for fixes" → close current task, advise user to create follow-up
```

## State Updates

1. When starting: `task.stages.qa.status = "in_progress"`, `started_at = now`.
2. After agent report:
   - Update `task.test_results` from agent output.
   - `task.stages.qa.issues` = every issue of the report (`id`, `severity` critical|major|minor, `location`, `description`).
   - `task.stages.qa.output_summary = "[N tests passed, N failed. N issues found.]"`.
   
3. On APPROVE + task close:
   - `task.stages.qa.status = "done"`, `completed_at = now`, `approved_at = now`.
   - Add `"qa"` to `stages_completed`.
   - `task.status = "done"`, `task.completed_at = now`.
   - `state.json.current_stage = "idle"`, `stage_status = "done"`, `active_task_id = null`.

4. On REQUEST_CHANGES + user chooses "Fix now":
   - Keep `task.stages.qa.status = "awaiting_approval"` (for reference).
   - Reset `task.stages.development.status = "not_started"`.
   - Remove `"development"` from `stages_completed`.
   - `state.json.current_stage = "development"`, `stage_status = "not_started"`.

5. On REQUEST_CHANGES + user chooses "Accept and close":
   - `task.stages.qa.status = "done"`, with `output_summary` noting accepted issues.
   - `task.status = "done"`, `task.completed_at = now`.
   - Close task as above.
