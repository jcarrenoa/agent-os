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
- [ ] No open ambiguities remain: every entry in `task.open_questions` has a `resolution`, or is flagged as open in the output after two question rounds.
- [ ] Requirements are testable (each can be verified pass/fail).

If any exit condition is not met, the requirements-analyst agent must ask additional questions before completing the stage.

## The Agent's Job

Before spawning the agent, the calling command follows the `skill-resolution` skill for stage `discovery` and passes the result to the agent under `## Extensions`. The agent reads any listed skills and docs before starting.

The `requirements-analyst` agent:
1. Reads `config.json` for project context.
2. Reads `CLAUDE.md` and `README.md` for domain knowledge.
3. Analyzes the task description.
4. If something that changes the requirements is ambiguous: returns a `QUESTIONS` block and stops (Clarifying Questions Protocol).
5. Otherwise, or on an answers run: produces the structured requirements document.
6. Returns it to the command for review.

## Clarifying Questions Protocol

The analyst is a subagent and cannot talk to the user. Questions always go through the calling command:

1. The analyst returns only this block (no requirements), at most 8 questions:
   ```
   QUESTIONS
   - Q-1: [question]
     Why: [what changes in the requirements depending on the answer]
     Options: [proposed default] | [alternative] | [alternative]
   ```
   IDs continue the numbering of `task.open_questions` (the first round starts at Q-1).
2. The command appends each question to `task.open_questions` with `resolution = null`, `resolved_by = null`.
3. The command gets the answers:
   - **unattended** approver mode: the approver answers them (Clarifying Questions in the `approval-gates` skill), `resolved_by = "approver"`.
   - **every other case**: ask the user with AskUserQuestion, at most 4 questions per call, one question per entry, its `Options` as the options (2–4; the first marked "(Recomendado)"; the user can always write another answer). Record the answer verbatim with `resolved_by = "human"`. If the user explicitly leaves the decision to the agent ("decide tú", "lo que propongas"), record the proposed default with `resolved_by = "default"`.
   - Set `resolved_at = now` for each answered question.
4. The command re-runs the analyst with the same inputs plus the line `Answers: read open_questions in the task file.` (references only, never the answers in the prompt).
5. The analyst may return a second `QUESTIONS` block at most once. After two rounds it must produce the requirements; anything still unclear stays in `open_questions` with `resolution = null` and is flagged in its output.

The analyst never answers its own questions. It records an assumption in `assumptions` only for details that do not change behavior the user would notice; anything else is a question.

## What the Requirements-Analyst Must NOT Do

- Propose technical solutions or architectural approaches.
- Make assumptions about behavior — ask instead (Clarifying Questions Protocol).
- Add requirements that were not implied by the user's request.
- Proceed to planning (that is the planner's job).

## Output Format

The agent returns a structured requirements block:

```
REQUIREMENTS OUTPUT
===================
Task: [title]
Summary: [2-3 sentences]
Change type: [feat | fix | chore | refactor | docs | test]

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
   - Set `task.change_type` from the agent's Change Type.
   - Populate `task.assumptions` and `task.out_of_scope`. `task.open_questions` is written only by the Clarifying Questions Protocol; questions the analyst flags as still open keep `resolution = null`.
   - On a revision run, replace these fields (except `open_questions`) with the new version.
   - If the agent returned a `QUESTIONS` block instead: Clarifying Questions Protocol, then re-run it.
3. Approver review (only when `config.approver.enabled` and `"requirements"` is in `config.approver.gates`): follow **Approver Review** in the `approval-gates` skill. It writes `task.stages.discovery.review`, and in autonomous or unattended mode may approve (step 5 with `approved_by = "approver"`), re-run the agent, or (unattended) block the stage.
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
