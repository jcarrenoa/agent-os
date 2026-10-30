---
description: Approval console for the active task. Shows the pending gate (requirements, plan or architecture), the approver verdict, and the approval history, and lets you approve, request changes, or see the full requirements, plan or ADRs.
allowed-tools: [Bash, Read, Write, Glob, Grep, AskUserQuestion]
---

# Agent OS — Review

You are showing the approval console for the active task. Write everything in `preferences.output_language`, as formatted Markdown (headings + tables, like the `/agent-os:setup` summary). The templates below show the Spanish labels; translate them for English.

This command shows **references and summaries**, not pasted artifacts: the full requirements, plan or ADRs are shown only with "Ver detalle".

## Step 1: Read State

1. Check `.agent-os/config.json`. If missing: tell the user to run `/agent-os:setup` and stop.
2. Read `config.json`, `.agent-os/state.json`, and the active task file `.agent-os/tasks/[active_task_id].json`.
3. If `active_task_id` is null: say there is no active task, list the last tasks in `.agent-os/tasks/` with their status, and stop.

## Step 2: Show the Console

```markdown
## 🗂️ Agent OS — Revisión · [TASK-NNN] [title]

**Etapa:** [current_stage] · **Estado:** [stage_status] · **Approver:** [desactivado | advisory · gates: requirements, planning | autonomous · gates: ... · max_revisions: N | unattended · gates: ... · max_revisions: N]

### ⏳ Gate pendiente — [Requisitos | Plan | Arquitectura]
| | |
|---|---|
| Gate | `[pending_approval.gate]` |
| Etapa | [current_stage] → [next_stage] |
| Solicitado | [requested_at] |
| Resumen | [stages.<stage>.output_summary] |
| Artefacto | `.agent-os/tasks/[TASK-NNN].json` [+ for design: `.agent-os/decisions/DEC-NNN.json`, one per decision] |
| Contenido | [requirements: N REQ / N US (must · should · could), N supuestos, N preguntas abiertas · planning: N subtareas (N backend, N frontend), complejidad, N riesgos · design: one line per ADR "DEC-NNN — title"] |
| Aprobación | [Humana · or "approver (autonomous / unattended)" for a gate in `config.approver.gates` (design only in unattended) · "Humana (pediste cambios)" when the stage has a change request] |
```

Then, if `stages.<stage>.review` exists, the approver verdict block from **Presenting the Approver Verdict** in the `approval-gates` skill. For `design` there is a review only in unattended mode.

Then the history of the task:

```markdown
### 📜 Historial de aprobaciones
| Etapa | Estado | Aprobado por | Fecha | Veredicto approver | Ronda | Rev. automáticas | Cambios pedidos | Sin resolver |
|---|---|---|---|---|---|---|---|---|
| Discovery | done | approver | 2026-10-29 10:12 | APPROVE | 2 | 1 | 0 | — |
| Planning | awaiting_approval | — | — | REQUEST_CHANGES | 1 | 0 | 1 (CR-1) | — |
| Design | not_started | — | — | — | — | — | — | — |
```

One row per stage in `discovery`, `planning`, `design` (and `qa` when it has `auto_revisions` or `unresolved_findings`), from the task file (`status`, `approved_by`, `approved_at`, `review.verdict`, `review.round`, `auto_revisions`, `change_requests` IDs, `unresolved_findings`). Use `—` for missing values.

If `task.approver_decisions` is not empty, add the **Approver Decisions Report** table from the `approval-gates` skill after the history.

## Step 3: Actions

Offer actions only when `state.stage_status = "awaiting_approval"` and `pending_approval.gate` is `requirements-approved`, `plan-approved` or `architecture-approved`.
- No pending gate: the console ends after Step 2. Add one line with the next command for the current stage (`/agent-os:start`, or the stage command).
- Any other gate (e.g. QA): show it, and point to the command that handles it (`/agent-os:qa`, `/agent-os:approve`). No actions.
- `stage_status = "blocked"` (e.g. unattended mode stopped on a blocker): show `state.json.blocker` and the unresolved IDs with their findings, and point to `/agent-os:rollback` or the stage command. No actions.

```
AskUserQuestion:
  question: "¿Qué hago con el gate [Requisitos | Plan | Arquitectura]?"
  header: "Revisión"
  options:
    - "Aprobar" — "Avanza a [next_stage]. Queda registrado como aprobado por ti."
    - "Pedir cambios" — "Escribes los cambios y se relanza [producing agent] con ellos y los hallazgos del approver."
    - "Ver detalle" — "Muestra completos [los requisitos / el plan / los ADR]."
    - "Cancelar" — "No cambia nada."
  multiSelect: false
```

An empty or unclear answer is "Cancelar". Only an explicit "Aprobar" approves.

### Aprobar

Follow **How to Record an Approval** in the `approval-gates` skill with `approved_by = "human"`: the same state writes as `/agent-os:approve` (its Step 3). The approver's verdict does not block it. Then tell the user what is next, as in Step 4 of `/agent-os:approve`.

### Pedir cambios

1. Ask the user, in a normal message, to describe the changes. Wait for the reply. An empty reply cancels.
2. Record the request **first**, before reading or judging anything else: append it as `CR-<k>` to `stages.<stage>.change_requests` with the text verbatim and the `cited_findings` copied (How to Handle Rejection or Change Requests in the `approval-gates` skill).
3. Always re-run the producing agent. Do not check the requirements, plan or ADRs yourself to decide whether the request is already covered, and do not ask the user again instead of re-running: if it is covered, the agent says so in its Revision Notes and the gate is shown again with them.
   - the producing agent is `requirements-analyst` (requirements), `planner` (plan) or `architect` (architecture, resolved with the **Architect: Multiple Roles** rules of `skill-resolution`);
   - resolve its extensions, re-run it with the human revision line (`change request CR-<k>`; references only), and write its new output to the task file as its stage workflow describes;
   - run the **Approver Review** again when the approver reviews that gate. Because the stage now has a change request, the gate is requested again from the human in every mode.
4. Show the console again (Step 2) for the new gate, with the agent's Revision Notes for `CR-<k>`.

### Ver detalle

Show the full artifact from the files, then ask the Step 3 question again:

- **Requisitos**: tables for user stories (`ID`, `Prioridad`, `Historia`, `Criterios de aceptación`) or functional requirements (`ID`, `Prioridad`, `Requisito`), then non-functional requirements, constraints, assumptions, out of scope, and the clarifying questions (`ID`, `Pregunta`, `Respuesta`, `Respondida por`: human / approver / default, or "sin resolver").
- **Plan**: a subtasks table (`ID`, `Título`, `Rol`, `Requisitos`, `Depende de`, `Criterio`), then complexity, risks (`Riesgo`, `Mitigación`), external dependencies and scope change.
- **Arquitectura**: for each ID in `task.decisions`, read `.agent-os/decisions/DEC-NNN.json` and show `title`, `status`, `context`, `decision`, `rationale`, `alternatives` and `consequences`, plus `stages.design.output_summary`.

### Cancelar

Change nothing. Remind the user that `/agent-os:review` and `/agent-os:approve` are available.

## Rules

- This command never approves anything without the explicit "Aprobar" choice, and never sets `approved_by = "approver"`: approver approvals happen only in the Approver Review of the stage commands.
- Every gate shown here, the architecture included, is decided by a human.
- "Pedir cambios" always records a `CR-<k>` and re-runs the agent (Step 3, Pedir cambios).
- Read every state file before writing it (state-management skill).
