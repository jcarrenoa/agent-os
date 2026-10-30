---
name: skills-menu
description: Interactive menu for configuring agent-os extensions (skills, docs, agents, commands). Invoked by /agent-os:skills with no arguments and by /agent-os:setup for the extension configuration step. Guides the user through selecting a destination (by agent or by stage/role), selecting elements, setting mode, and choosing the save level.
version: 1.2.0
---

# Skills Menu Protocol

This protocol presents an interactive menu for configuring agent-os extensions. Follow every step in order. Never combine two questions into one. Never present more than 4 options in a single AskUserQuestion call.

## Prerequisites

Before starting the menu:
1. Read `.agent-os/config.json` (project-level extensions).
2. Read `~/.agent-os/defaults.json` if it exists (user-level extensions).
3. Glob `.claude/skills/*/SKILL.md` and `~/.claude/skills/*/SKILL.md` — read the `name` and `description` frontmatter of each (first 6 lines). Build a list: `available_skills[]`.
4. Glob `.claude/agents/*.md` and `~/.claude/agents/*.md` — read `name` and `description`. Build `available_agents[]`.
5. Glob `.claude/commands/*.md` and `~/.claude/commands/*.md` — read `description` (it may be missing: a command without frontmatter is still listed, by file name). Build `available_commands[]`.
5b. If `config.workspace_type` is `monorepo` or `multi-repo`: repeat steps 3–5 for `<workspace_paths.backend>/.claude/` and `<workspace_paths.frontend>/.claude/`, and tag each element with its origin (`backend`, `frontend`). Elements from a subrepo are only offered for destinations of that role (or QA).
6. Collect docs already referenced in extensions (from config.json + defaults.json). Build `available_docs[]`.

---

## Entrypoint

The calling command may restrict which initial options appear. The full set is:

- `"Por agente (Recomendado)"` — Assign skills/docs directly to a specific agent.
- `"Por etapa y rol"` — Assign to all agents in a stage or role.
- `"Ver configuración actual"` — Show the combined config with provenance (same as `list`).
- `"Quitar asignaciones"` — Remove already-assigned skills/docs from a destination.
- `"Omitir por ahora"` — Exit the menu without saving (shown only from setup context).

The calling command passes the subset to show. Always keep the same order as above; omit those not in the subset.

---

## Step A: Action

```
AskUserQuestion:
  question: "¿Qué quieres configurar?"
  header: "Extensiones"
  options: [the subset provided by the calling command]
  multiSelect: false
```

- If **"Ver configuración actual"**: call the `skill-resolution` skill for all stages/agents and present a combined summary. Then ask: "¿Configurar algo más?" → "Sí, volver al menú" | "No, listo". If yes, restart Step A.
- If **"Omitir por ahora"**: exit the menu immediately.
- If **"Quitar asignaciones"**: follow Steps B–C to choose the destination, then jump to **Step D-Remove**.
- Otherwise: continue to Step B.

---

## Step B: Choose Destination

### B1 — Por agente

```
AskUserQuestion:
  question: "¿A qué grupo de agentes quieres asignar?"
  header: "Grupo"
  options:
    - "Análisis — requirements-analyst, planner, approver"
    - "Diseño — architect"
    - "Desarrollo — backend-developer, frontend-developer"
    - "QA — qa-engineer"
  multiSelect: false
```

- **"Diseño"** → destination = `agent:architect`. Skip Step B1b.
- **"QA"** → destination = `agent:qa-engineer`. Skip Step B1b.
- **"Análisis"** or **"Desarrollo"** → continue to Step B1b.

**Step B1b — Select specific agent** (only for Análisis and Desarrollo):

For Análisis:
```
AskUserQuestion:
  question: "¿A cuál de los agentes de análisis?"
  header: "Agente"
  options:
    - "requirements-analyst — Captura de requisitos (Stage 1)"
    - "planner — Planificación de subtasks (Stage 2)"
    - "approver — Revisa requisitos y plan (opcional, config.approver)"
  multiSelect: false
```

For Desarrollo:
```
AskUserQuestion:
  question: "¿A cuál de los agentes de desarrollo?"
  header: "Agente"
  options:
    - "backend-developer — Implementa el backend"
    - "frontend-developer — Implementa el frontend"
  multiSelect: false
```

Set destination = `agent:<selected-agent-name>`.

---

### B2 — Por etapa y rol

**Step B2a — Select stage:**
```
AskUserQuestion:
  question: "¿A qué etapa?"
  header: "Etapa"
  options:
    - "Discovery / Planning — Etapas de análisis"
    - "Design — Diseño y arquitectura"
    - "Development — Desarrollo"
    - "QA — Pruebas y calidad"
  multiSelect: false
```

- If **"Discovery / Planning"** → ask Step B2b.
- If **"Design"** or **"Development"** → ask Step B2c.
- If **"QA"** → destination = `stage:qa`. No role question.

**Step B2b — Discovery or Planning** (only if "Discovery / Planning" was chosen):
```
AskUserQuestion:
  question: "¿Discovery o Planning?"
  header: "Etapa"
  options:
    - "Discovery (Stage 1) — Captura de requisitos"
    - "Planning (Stage 2) — Plan de implementación"
  multiSelect: false
```

Set destination = `stage:discovery` or `stage:planning`.

**Step B2c — Role** (only for Design and Development):

Before presenting options, read `config.focus`:
- `"backend"`: the project has no frontend agent. For development, set destination = `stage:development:backend` automatically and skip this question. For design, show the question but omit "Solo Frontend".
- `"frontend"`: the project has no backend agent. For development, set destination = `stage:development:frontend` automatically and skip this question. For design, show the question but omit "Solo Backend".
- `"fullstack"` (default): show all three options.

```
AskUserQuestion:
  question: "¿Alcance de la asignación?"
  header: "Rol"
  options:
    - "Toda la etapa — aplica a backend y frontend"  ← omit if focus is backend or frontend
    - "Solo Backend"  ← omit if focus = "frontend"
    - "Solo Frontend"  ← omit if focus = "backend"
  multiSelect: false
```

Set destination = `stage:design`, `stage:design:backend`, `stage:design:frontend`, `stage:development:backend`, or `stage:development:frontend`.

---

## Step C: Determine if Destination Implements Code

A destination "implements code" when it is one of:
- `agent:architect`, `agent:backend-developer`, `agent:frontend-developer`
- `stage:design`, `stage:design:backend`, `stage:design:frontend`
- `stage:development:backend`, `stage:development:frontend`

Store this as `implements_code = true|false`. Used in Step E.

Determine if destination is **QA** (`agent:qa-engineer` or `stage:qa`). Store as `is_qa = true|false`. Used in Step D to show agents/commands.

---

## Step D: Select Elements

### Build element lists for this destination

- **skills_to_show**: `available_skills[]`, excluding any already assigned to this destination (those are marked ✓)
- **docs_to_show**: `available_docs[]`, excluding already-assigned ones (marked ✓)
- **agents_to_show**: `available_agents[]` (only if `is_qa = true`)
- **commands_to_show**: `available_commands[]` (only if `is_qa = true`)

Get currently assigned elements for the destination from the combined config (project + user levels). Mark each with ✓.

Build a flat list `all_elements[]` in this order:
1. Skills (marked ✓ or not)
2. Docs (marked ✓ or not)
3. Agents (only if is_qa; marked ✓ or not)
4. Commands (only if is_qa; marked ✓ or not)

### Paginated selection

Split `all_elements[]` into batches of 3. Show one batch at a time.

For each batch:
```
AskUserQuestion:
  question: "Selecciona elementos para [destination] (✓ = ya asignado). Página [N/total]:"
  header: "Elementos"
  options:
    - "[✓] <name> — <description>" (if already assigned)
    - "<name> — <description>" (if not assigned)
    - (up to 3 items)
    - "Ver más…" (only as 4th option, only if there are more batches)
  multiSelect: true
```

- Record selected items (exclude "Ver más…" from the selection).
- If "Ver más…" was selected OR if the user clicked "Ver más…" as the only non-selection: show the next batch, accumulating selections.
- After the last batch (no "Ver más…"): finalize `selected_elements[]`.

Items selected that had ✓ (already assigned) → no change (keep assigned).
Items selected that did NOT have ✓ → add to assignment.
Items that had ✓ but were NOT selected → remove from assignment (deassign).

If `all_elements[]` is empty: tell the user "No se encontraron skills ni docs disponibles." and offer "Other" (automatic from AskUserQuestion) to enter a path or name manually. Accept the input and add it to `selected_elements[]`.

---

## Step D-Remove (Quitar asignaciones)

Same as Step D but:
- Only show items that are currently assigned to this destination (those with ✓).
- Items selected → remove from assignment.
- Items not selected → keep.

If no items are assigned to this destination: tell the user and exit.

---

## Step E: Mode (only if implements_code = true)

```
AskUserQuestion:
  question: "¿Cómo aplicar las skills en [destination]?"
  header: "Modo"
  options:
    - "Extend (Recomendado) — se añade tech-adaptation (default del plugin) antes de las skills configuradas"
    - "Replace — no se añade tech-adaptation; las skills de contexto, etapa, rol y agente se mantienen"
  multiSelect: false
```

Store result as `selected_mode`.

---

## Step E2: Post-QA Command Approval (only if is_qa = true and commands were selected)

Every post-QA command requires an approval gate by default. Ask only to opt specific commands out:

```
AskUserQuestion:
  question: "¿Qué comandos post-QA pueden ejecutarse SIN pedir aprobación? (por defecto todos piden aprobación)"
  header: "Aprobación"
  options:
    - "<command> — <description or 'sin descripción'>" (one per selected command, up to 3 per page; paginate as in Step D)
  multiSelect: true
```

Store the selected names as `auto_commands[]`. Unselected commands stay gated. Never pre-select a command, and never infer `auto` from its description or name.

---

## Step F: Save Level

```
AskUserQuestion:
  question: "¿Dónde guardar esta configuración?"
  header: "Nivel"
  options:
    - "Este proyecto (.agent-os/config.json) (Recomendado)"
    - "Todos mis proyectos (~/.agent-os/defaults.json)"
  multiSelect: false
```

Store as `save_target = "project" | "global"`.

---

## Step G: Preview and Confirm

Build the merged result for this destination (what the agent would actually receive):

```
─────────────────────────────────────────
Vista previa — [destination]
─────────────────────────────────────────
Orden: genérico → específico. Lo posterior prevalece en caso de conflicto; el CLAUDE.md del proyecto prevalece sobre todo.

Skills (en orden de aplicación):
  [/path/to/SKILL.md]   (plugin · default)
  [/path/to/SKILL.md]   (etapa · proyecto)
  [/path/to/SKILL.md]   (agente · proyecto)

Docs:
  [/path/to/doc.md]     (agente · proyecto)

Modo: extend | replace

[Si is_qa:]
Agentes extra: [...]
Comandos post-QA: [nombre [requiere aprobación] | nombre [auto] ...]
─────────────────────────────────────────
```

```
AskUserQuestion:
  question: "¿Guardar esta configuración?"
  header: "Confirmar"
  options:
    - "Guardar"
    - "Editar selección — volver al paso de elementos"
    - "Cancelar — descartar cambios"
  multiSelect: false
```

- **"Editar selección"**: go back to Step D.
- **"Cancelar"**: discard, skip to Step H.
- **"Guardar"**: write to the target file (see below).

### Write logic

**For `save_target = "project"`**: Read `.agent-os/config.json`, merge `extensions` (do not replace), write back.
- Destination `agent:<name>` → `extensions.agents.<name>`
- Destination `stage:<stage>` → `extensions.stages.<stage>`
- Destination `stage:<stage>:<role>` → `extensions.stages.<stage>.<role>`

**For `save_target = "global"`**: Read `~/.agent-os/defaults.json` (create if missing), merge, write back. Same mapping.

**Merge rules**:
- Lists: union without duplicates. New entries are appended at the end of their list.
- `commands`: write commands in `auto_commands[]` as `{ "name": "<command>", "auto": true }` and every other command as a plain string (gated). If a command already exists in the list, replace its entry with the new form.
- `mode`: set to `selected_mode` if this is a code-implementing destination.
- `scaffold_if_new`: preserve existing value (do not change from the menu).

---

## Step H: Continue?

```
AskUserQuestion:
  question: "¿Configurar otro destino?"
  header: "Continuar"
  options:
    - "Sí, configurar otro"
    - "No, he terminado"
  multiSelect: false
```

If "Sí": restart from Step A.
If "No": exit the menu.
