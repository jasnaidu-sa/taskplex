# /taskplex - Adaptive Workflow with Agent Orchestration

**Command**: `/taskplex [flags] [task description]`

Single entry point for all development tasks.

## Flags

**Route** (if omitted, user is prompted to choose):
- `--standard` or `--full` — Full design, single implementation agent *(default)*
- `--team` or `--parallel` — Full design, multi-agent execution (1-3 parallel agents)
- `--blueprint` or `--architect` or `--deep` — Opus architect + critics + multi-agent + worktrees

**Modifiers:**
- `--prd` — Blueprint at initiative scale: feature decomposition + wave execution + cross-feature closure
- `--light` — Reduced design depth (fewer questions, minimal brief)
- `--plan PLAN-{id}` — Use existing plan file
- `--skip-design` — No design phase, straight to execution (logs degradation)

## Examples

```
/taskplex                                 # interactive: asks task + route
/tp add user authentication               # asks route choice, then begins
/tp --light fix the login button          # standard route, light design depth
/tp --team refactor the API layer         # full design, multi-agent, no menu
/tp --blueprint redesign the data pipeline # opus architect + multi-agent + worktrees
/tp --prd Q3 feature roadmap             # blueprint at initiative scale
```

---

## MANDATORY EXECUTION PROTOCOL

**STOP. You are now entering a structured workflow. You MUST follow these steps in exact order. Do NOT skip ahead. Do NOT write any source code until the design phase is complete and approved.**

**The `tp-design-gate` hook WILL BLOCK your Edit/Write calls** if you try to write artifacts before reaching the required design sub-phase. The `tp-heartbeat` hook tracks every file edit. The `tp-pre-commit` hook blocks commits without validation. These are enforced — you cannot bypass them.

### STEP 0.5: Load Skill Evolutions (if any)

Check for `~/.claude/skills/taskplex/evolutions.json`. If it exists, read it and load active evolutions into your context. These are improvements learned from past task failures and corrections — apply them throughout this task.

If no evolutions.json exists: skip. This is expected for new installations.

### STEP 1: Read the initialization phase file (MANDATORY)

You MUST read this file IN FULL before doing anything else:

```
Read: ~/.claude/taskplex/phases/init.md
```

This file contains Steps 0-5 that you must follow sequentially:
- Step 0: Parse task description and flags, select route
- Step 1: Create `.claude-task/{taskId}/manifest.json`
- Step 2: Create session file (MANDATORY for hooks to work)
- Step 3: Load context, check for resume, detect PRD/plan references
- Step 4: Detect project type, load conventions
- Sub-phase A: Convention check (scan codebase, ask questions in full mode)
- Sub-phase B: Intent/user journey exploration (MUST interact with user)
- Step 5: Quality profile selection, branch creation

**HARD RULES from init.md that you MUST follow:**
1. In BOTH modes: You MUST gather context from invocation + docs + scan BEFORE the first question
2. In BOTH modes: You MUST synthesize context and confirm understanding with the user (sets `contextConfirmed`)
3. In BOTH modes: You MUST NOT ask questions the documentation already answers
4. In FULL design depth: You MUST resolve all ambiguities through adaptive questioning (sets `ambiguitiesResolved`)
5. In FULL design depth: You MUST propose 2-3 approaches and get user selection BEFORE writing brief.md
6. In FULL design depth: You MUST present design section-by-section and get user approval BEFORE writing brief.md
7. You MUST update `manifest.designPhase` after completing each sub-phase
8. You MUST update `manifest.designInteraction` flags and counters as you progress

**The design gate hook checks `contextConfirmed`, `ambiguitiesResolved`, `approachSelected`, and `sectionsApproved`. Question counters are tracked for observability but are NOT gate criteria. The amount of interaction adapts to context density — rich input needs less ceremony, vague input needs more.**

### STEP 2: Read the planning & execution phase file

After init is complete and brief is written/approved:

```
Read: ~/.claude/taskplex/phases/planning.md
```

This file defines three routes (Standard, Team, Blueprint) with:
- Planning agent spawn
- Spec critic review
- Pre-implementation user acknowledgment (MANDATORY — last user checkpoint before implementation)
- Implementation agent dispatch
- Build checks

**For Team/Blueprint routes**:
- The orchestrator MUST delegate to agents — the implementation gate hook blocks orchestrator source edits until `manifest.implementationDelegated = true`
- Blueprint agents MUST use `isolation: "worktree"` — no exceptions
- Independent agents MUST be dispatched in a single message (parallel Agent tool calls)
- **Execution Continuity Rule**: Once the user approves the plan, implementation runs non-stop to completion. Do NOT pause between agents, do NOT ask "should I continue?". The only reason to stop is a blocking failure.

### STEP 3: Read the QA phase file

After implementation is complete:

```
Read: ~/.claude/taskplex/phases/qa.md
```

This phase tests the implementation by actually using it — browser walkthrough for UI apps, running commands for CLIs, calling endpoints for APIs. It references journeys from `product/brief.md` if one exists. Bugs found are fixed in up to 3 rounds before proceeding.

### STEP 4: Read the validation phase file

After QA is complete:

```
Read: ~/.claude/taskplex/phases/validation.md
```

This file defines the full validation pipeline:
- Artifact validation, traceability, build checks
- Security review, closure, code review
- Hardening (standard/enterprise profiles)
- Compliance (final gate — nothing passes without this)
- Git integration, PR creation
- Post-completion summary

### Policy and gate references

- **Policy**: `~/.claude/taskplex/policy.json` — quality profiles, gates, limits
- **Gates**: `~/.claude/taskplex/gates.md` — canonical gate names, verdict enums, execution order
- **Artifacts**: `~/.claude/taskplex/artifact-contract.md` — required artifacts by profile
- **Manifest schema**: `~/.claude/taskplex/manifest-schema.json` — field definitions
- **Handoff contract**: `~/.claude/taskplex/handoff-contract.md` — agent transition format

---

## PROGRESS CHECKLIST (MANDATORY — output at top of EVERY message)

**You MUST output this checklist at the top of EVERY response throughout the entire task lifecycle.** Update it as you progress. This is your state tracker — it prevents you from skipping steps or losing your place. The current phase is marked with `📍`. The current step within that phase is marked with `➡️`.

### Full Design Depth Checklist

```
═══ TASKPLEX PROGRESS ═══

📍 PHASE 1: INITIALIZATION
  {✅|➡️|⬜} 1.0 Parse task description & select route
  {✅|➡️|⬜} 1.1 Create task manifest + directory
  {✅|➡️|⬜} 1.2 Create session file
  {✅|➡️|⬜} 1.3 Load context, check resume, detect PRD
  {✅|➡️|⬜} 1.4 Detect project type, load conventions

📍 PHASE 2: DESIGN — Convention Check (Sub-phase A)
  {✅|➡️|⬜} 2.1 Automated convention scan
  {✅|➡️|⬜} 2.2 Convention questions (2-4, one per turn) [full only]

📍 PHASE 3: DESIGN — Intent Exploration (Sub-phase B)
  {✅|➡️|⬜} 3.1 Synthesize + confirm — gather context (invocation+docs+scan), confirm with user [contextConfirmed]
  {✅|➡️|⬜} 3.2 Targeted questions — adaptive (0-5), resolve remaining gaps [ambiguitiesResolved]
  {✅|➡️|⬜} 3.3 User journey mapping — before/after, interactions, edge cases
  {✅|➡️|⬜} 3.4 Scope boundaries — in/out/deferred, confirm with user
  {✅|➡️|⬜} 3.5 Propose approaches — 2-3 options with trade-offs, user selects [approachSelected]
  {✅|➡️|⬜} 3.6 Section-by-section design approval — present each, get approval [sectionsApproved]
  {✅|➡️|⬜} 3.7 Write brief.md

📍 PHASE 4: PLANNING
  {✅|➡️|⬜} 4.1 Write spec.md (+ sections.json for team/blueprint)
  {✅|➡️|⬜} 4.2 Spec critic review
  {✅|➡️|⬜} 4.3 Pre-implementation acknowledgment (user reviews full spec inline)
  {✅|➡️|⬜} 4.4 Refine task list — replace placeholders with actual plan tasks

📍 PHASE 5: IMPLEMENTATION (tasks refined at 4.4 based on route + spec)
  {✅|➡️|⬜} 5.1 Implementation (refined per route — see task list)
  {✅|➡️|⬜} 5.2 Coherence check — verify code matches spec (haiku, fast)
  {✅|➡️|⬜} 5.3 Build gate (typecheck + lint + tests)
  {✅|➡️|⬜} 5.4 Update documentation (if applicable)

📍 PHASE 5.5: QA
  {✅|➡️|⬜} 5.5.1 Detect product type & QA strategy
  {✅|➡️|⬜} 5.5.2 Smoke test
  {✅|➡️|⬜} 5.5.3 Journey walkthrough
  {✅|➡️|⬜} 5.5.4 Edge case probing
  {✅|➡️|⬜} 5.5.5 Bug fix loop (0/3 rounds)
  {✅|➡️|⬜} 5.5.6 QA report

📍 PHASE 6: VALIDATION
  {✅|➡️|⬜} 6.0 Artifact validation (blocking pre-check)
  {✅|➡️|⬜} 6.0.5 Traceability gate [standard + enterprise]
  {✅|➡️|⬜} 6.1 Build validation (typecheck + lint + tests)
  {✅|➡️|⬜} 6.1b Convention compliance checks
  {✅|➡️|⬜} 6.2 Security review (security-reviewer)
  {✅|➡️|⬜} 6.3 Closure — requirements verification (closure-agent)
  {✅|➡️|⬜} 6.4 Code review (code-reviewer) [standard + enterprise]
  {✅|➡️|⬜} 6.5 Conditional reviewers (database, e2e, user-workflow) [if triggered]
  {✅|➡️|⬜} 6.6 Build-fixer [if any reviewer found issues]
  {✅|➡️|⬜} 6.7 Hardening (hardening-reviewer) [standard + enterprise]
  {✅|➡️|⬜} 6.8 Compliance — final gate (compliance-agent)
  {✅|➡️|⬜} 6.9 Write validation-gate.json

📍 PHASE 7: COMPLETION (part of validation phase in manifest)
  {✅|➡️|⬜} 7.1 Git commit
  {✅|➡️|⬜} 7.2 Push/PR (if requested)
  {✅|➡️|⬜} 7.3 Task summary
  {✅|➡️|⬜} 7.4 Brief validation (if product/brief.md exists)
```

### Light Design Depth Checklist

```
═══ TASKPLEX PROGRESS (light) ═══

📍 PHASE 1: INITIALIZATION
  {✅|➡️|⬜} 1.0-1.4 Task setup + context

📍 PHASE 2: DESIGN (light)
  {✅|➡️|⬜} 2.1 Convention scan (auto)
  {✅|➡️|⬜} 2.2 Synthesize + confirm [contextConfirmed] + fill gaps (adaptive, 0-2 questions)
  {✅|➡️|⬜} 2.3 Write brief.md

📍 PHASE 3-5: (same as full — planning, implementation)
📍 PHASE 5.5: QA (same as full)
📍 PHASE 6-7: (same as full — validation, completion)
```

### Checklist Rules

- You MUST be on the step marked ➡️ — do NOT jump ahead
- Each design step (Phase 3) is a SEPARATE user interaction turn — do NOT combine
- After user answers, do targeted research THEN ask the next question
- Do NOT propose approaches (3.5) until Steps 3.1-3.4 are complete
- Do NOT discuss implementation details (UI patterns, components) until Step 3.5+
- Questions should build on previous answers — each answer drives the next research
- Mark completed steps with ✅ immediately after they pass
- If a step is skipped (e.g., light mode skips convention questions), mark as `⏭️`

### Persisting Checklist to Manifest (MANDATORY — survives compaction)

**On EVERY step transition**, you MUST write the checklist state to `manifest.phaseChecklist`. This is what the hooks read — if you only display it in chat, it's lost on compaction.

**Schema** for `manifest.phaseChecklist`:
```json
{
  "initialization": {
    "1.0 Parse task & select route": "done|active|pending",
    "1.1 Create manifest": "done|active|pending",
    "1.2 Create session file": "done|active|pending",
    "1.3 Load context & check resume": "done|active|pending",
    "1.4 Detect project & conventions": "done|active|pending"
  },
  "conventionCheck": {
    "2.1 Automated convention scan": "done|active|pending|skipped",
    "2.2 Convention questions": "done|active|pending|skipped"
  },
  "intentExploration": {
    "3.1 Synthesize + confirm": "done|active|pending",
    "3.2 Targeted questions (adaptive)": "done|active|pending|skipped",
    "3.3 User journey mapping": "done|active|pending",
    "3.4 Scope boundaries": "done|active|pending",
    "3.5 Propose approaches": "done|active|pending",
    "3.6 Section-by-section approval": "done|active|pending",
    "3.7 Write brief.md": "done|active|pending"
  },
  "planning": {
    "4.1 Write spec.md": "done|active|pending",
    "4.2 Spec critic review": "done|active|pending",
    "4.3 Pre-implementation acknowledgment": "done|active|pending"
  },
  "implementation": {
    "5.1 Implementation": "done|active|pending",
    "5.2 Build check": "done|active|pending"
  },
  "qa": {
    "5.5.1 Detect product type & QA strategy": "done|active|pending|skipped",
    "5.5.2 Smoke test": "done|active|pending|skipped",
    "5.5.3 Journey walkthrough": "done|active|pending|skipped",
    "5.5.4 Edge case probing": "done|active|pending|skipped",
    "5.5.5 Bug fix loop": "done|active|pending|skipped",
    "5.5.6 QA report": "done|active|pending|skipped"
  },
  "validation": {
    "6.0 Artifact validation": "done|active|pending",
    "6.0.5 Traceability gate": "done|active|pending|skipped",
    "6.1 Build validation": "done|active|pending",
    "6.1b Convention compliance": "done|active|pending|skipped",
    "6.2 Security review": "done|active|pending",
    "6.3 Closure": "done|active|pending",
    "6.4 Code review": "done|active|pending|skipped",
    "6.5 Conditional reviewers": "done|active|pending|skipped",
    "6.6 Build-fixer": "done|active|pending|skipped",
    "6.7 Hardening": "done|active|pending|skipped",
    "6.8 Compliance": "done|active|pending",
    "6.9 Write validation-gate.json": "done|active|pending"
  },
  "completion": {
    "7.1 Git commit": "done|active|pending",
    "7.2 Push/PR": "done|active|pending|skipped",
    "7.3 Task summary": "done|active|pending"
  }
}
```

**How to update**: After completing a step, use Bash with a node one-liner to read manifest, update the relevant step status, and write back. Example:
```bash
node -e "
const fs = require('fs');
const p = '.claude-task/{taskId}/manifest.json';
const m = JSON.parse(fs.readFileSync(p,'utf8'));
if (!m.phaseChecklist) m.phaseChecklist = {}; // init if missing
m.phaseChecklist.intentExploration = m.phaseChecklist.intentExploration || {};
m.phaseChecklist.intentExploration['3.1 Synthesize + confirm'] = 'done';
m.phaseChecklist.intentExploration['3.2 Targeted questions (adaptive)'] = 'active';
fs.writeFileSync(p, JSON.stringify(m, null, 2));
"
```

**Hook integration**:
- `tp-session-start.mjs` reads `manifest.phaseChecklist` and renders it into `additionalContext` after compaction — so you see it immediately
- `tp-session-start.mjs` also instructs you to recreate TaskCreate tasks from the checklist for task tracking
- `tp-pre-compact.mjs` includes `phaseChecklist` in the checkpoint snapshot
- `tp-heartbeat.mjs` already persists the manifest on every edit — the checklist is saved automatically

---

## MID-TASK CHANGES (interrupts, scope changes, questions)

The user may interrupt at any time during a task — to add scope, change direction, ask questions, or pause. Handle it naturally:

1. **The user can redirect you at any time.** The workflow is a guide, not a prison. If the user says "also add X" or "change Y to Z", do it.
2. **Always update the task list to reflect changes.** If scope expands, `TaskCreate` for the new work. If a task changes, `TaskUpdate` the description. If work is done, mark it completed. The task list must always reflect reality.
3. **Always update the manifest and artifacts to reflect changes.** If the user changes scope, update brief.md. If they change direction, update the spec. If they add requirements, update the manifest. Keep `.claude-task/` current.
4. **Side questions don't advance the workflow.** If the user asks "how does X work?" — answer it, then continue from where you were. Don't update the checklist or phase state.
5. **If the user says "pause" or "stop"** — save state to manifest, note where you are in progressNotes, and stop. The session-start hook will pick up where you left off next time.

---

## ANTI-PATTERNS (things you must NEVER do)

1. **NEVER start coding without reading init.md first** — the hooks will block you
2. **NEVER skip user interaction** — even with rich context, you must confirm understanding with the user
3. **NEVER write brief.md before contextConfirmed is set** — the hook checks flags, not counters
4. **NEVER commit without passing validation** — the pre-commit hook blocks this
5. **NEVER "wing it" by guessing what the workflow should be** — READ THE PHASE FILES
6. **NEVER batch all design questions into one message** — one question per turn, wait for answer
7. **NEVER front-load 10+ minutes of research before asking the first question** — engage user within 30 seconds
8. **NEVER ask questions the documentation already answers** — gather context first, ask about gaps
9. **NEVER force questions for ceremony** — if context is sufficient, fewer questions is better
10. **NEVER implement directly in team/blueprint mode** — delegate to agents, the implementation gate will block you
11. **NEVER ask "should I continue?" during implementation** — the user approved the plan, execute it non-stop
12. **NEVER pause between agent dispatches** — dispatch all independent agents in one message (parallel calls)
13. **NEVER skip worktrees in blueprint mode** — every implementation agent MUST use `isolation: "worktree"`
14. **NEVER loop through features one-at-a-time asking for confirmation** — dispatch wave, run to completion

## RECOVERY

If you are resuming from compaction or a new session:
1. Read `manifest.json` from `.claude-task/{taskId}/` for full task state
2. Check `manifest.phase` and `manifest.designPhase` to know where you are
3. Check `manifest.phaseChecklist` for step-level progress
4. Recreate TaskCreate tasks from the checklist for remaining pending/active steps
5. Continue from the step marked `active` — do NOT restart the workflow
