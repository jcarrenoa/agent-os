# Workflow: Stage 1 — Discovery & Requirements

## Purpose

Understand what needs to be built before any planning or implementation begins. Produce a complete, unambiguous set of requirements that all subsequent stages will build on.

## Entry Conditions

- `.agent-os/config.json` exists and is valid.
- A task description has been provided by the user.
- `state.json.current_stage = "discovery"` and `stage_status = "in_progress"`.

## Exit Conditions (all must be true before requesting approval)

- [ ] At least one functional requirement (`must`) is captured: a `REQ-NNN` with `requirements_format = "req"`, or a `US-NNN` story with at least one acceptance criterion with `"user-stories"`.
- [ ] At least one non-functional requirement is captured (or explicitly noted as none).
- [ ] Out-of-scope items are explicitly listed.
- [ ] No open ambiguities remain (all clarifying questions answered).
- [ ] Requirements are testable (each can be verified pass/fail).

If any exit condition is not met, the requirements-analyst agent must ask additional questions before completing the stage.

## The Agent's Job

Before spawning the agent, the calling command follows the `skill-resolution` skill for stage `discovery` and passes the result to the agent under `## Extensions`. The agent reads any listed skills and docs before starting.

The `requirements-analyst` agent:
1. Reads `config.json` for project context.
2. Reads `CLAUDE.md` and `README.md` for domain knowledge.
3. Analyzes the task description.
4. Identifies ambiguities and asks the minimum set of clarifying questions.
5. **Waits for user answers.**
6. Produces the structured requirements document.
7. Returns it to the command for review.

## What the Requirements-Analyst Must NOT Do

- Propose technical solutions or architectural approaches.
- Make assumptions about behavior — ask instead.
- Add requirements that were not implied by the user's request.
- Proceed to planning (that is the planner's job).

## Output Format

The agent returns a structured requirements block:

```
REQUIREMENTS OUTPUT
===================
Task: [title]
Summary: [2-3 sentences]

FUNCTIONAL REQUIREMENTS                  (requirements_format = "req", the default)
  REQ-001 [must]: [description]
  REQ-002 [should]: [description]
  REQ-003 [could]: [description]

USER STORIES                             (requirements_format = "user-stories", instead of FUNCTIONAL REQUIREMENTS)
  US-001 [must]: As a [role], I want [capability], so that [benefit].
    US-001-AC1: Given [context], when [action], then [result].

NON-FUNCTIONAL REQUIREMENTS
  REQ-004 [must]: [description]

CONSTRAINTS
  REQ-005 [must]: [description]

ASSUMPTIONS
  - [assumption made instead of asking, or "none"]

OUT OF SCOPE
  - [explicit out-of-scope item]

RECOMMENDED SUBTASK AREAS (for planner)
  1. [area]
  2. [area]
===================
```

## State Updates

During this stage, the command updates:

1. When starting: `task.stages.discovery.status = "in_progress"`, `started_at = now`.
2. When agent completes:
   - `task.requirements_format` = `config.preferences.requirements_format` (missing = `"req"`).
   - Populate `task.requirements` (functional with `"req"`; non-functional and constraints always) and, with `"user-stories"`, `task.user_stories` (`id`, `role`, `capability`, `benefit`, `priority`, `acceptance_criteria[{ id, text }]`).
   - Populate `task.assumptions`, `task.out_of_scope` and `task.open_questions`.
   - On a revision run, replace these fields with the new version.
3. Approver review (only when `config.approver.enabled` and `"requirements"` is in `config.approver.gates`): follow **Approver Review** in the `approval-gates` skill. It writes `task.stages.discovery.review`, and in autonomous mode may approve (step 5 with `approved_by = "approver"`) or re-run the agent.
4. When requesting approval:
   - `task.stages.discovery.status = "awaiting_approval"`, `completed_at = now`
   - `task.stages.discovery.output_summary = "[N requirements / N stories captured. Scope: ...]"`
   - `state.json.stage_status = "awaiting_approval"`
   - `state.json.pending_approval = { gate: "requirements-approved", context: "...", next_stage: "planning", approver_verdict: [verdict or null] }`

5. When approved:
   - `task.stages.discovery.status = "done"`, `approved_at = now`, `approved_by = "human" | "approver"`
   - `state.json.stages_completed` gets `"discovery"` appended
   - `state.json.current_stage = "planning"`, `stage_status = "not_started"`
   - `state.json.pending_approval = null`

## Approval Criteria

The user should approve when:
- Requirements accurately describe what they want to build.
- Scope is clearly bounded (no unintended inclusions or exclusions).
- All `must` requirements are present.

The user should reject when:
- A key requirement is missing.
- The scope is too broad or too narrow.
- A requirement is ambiguous or untestable.
