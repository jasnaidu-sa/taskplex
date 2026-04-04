# taskplex: Planning & Implementation
<!-- Loaded by orchestrator after initialization. Self-contained. -->
<!-- v3: 3-route architecture (Standard/Team/Blueprint) with universal planning agent -->
<!-- v4: Added progress visibility, inline artifact presentation, pre-spawn status messages -->
<!-- v5: Collapsed to 3 routes (Light/Standard/Blueprint). Standard absorbs Team (multi-agent + mandatory critic). -->

**Policy reference**: `~/.claude/taskplex/policy.json` for iteration limits.

## ⚠️ User Visibility Rules (HARD RULE — all routes)

**The user must see what's happening at all times.** The chat window is where the user works — not files on disk.

1. **Before every agent spawn**: Tell the user what agent is being spawned, what it will do, and roughly how long it takes.
2. **After every agent returns**: Present a **structured working summary** in the conversation — not the full artifact, not a 3-line terse summary.
3. **Never say "review file X"** — present the content where the user is reading.
4. **Never leave the user waiting with no information** for more than 30 seconds.

### Presentation Detail by Phase

| Phase | What to show in chat | Detail Level |
|-------|---------------------|:---:|
| **Design (Sub-phase B)** | Full context synthesis, questions, approaches, section approvals | High |
| **Architecture (Blueprint)** | Component breakdown per wave, key decisions per section, intent coverage matrix | High |
| **Spec/Planning** | Section summaries: what each section does, files involved, key decisions. Not full spec. | Medium |
| **Implementation** | Per-worker status: "Worker 1 complete: auth module, 4 files, build passing" | Low |
| **QA** | Journey results table, verification verdict, bugs found | Medium |
| **Validation** | Gate verdicts (PASS/FAIL per gate), summary of findings | Low-Medium |

### Structured Summary Format (for architecture + planning)

When presenting the architecture or spec after an agent returns, use this format:

```
{Wave/Section Name}:
  - {Component}: {what it does}. {N} files.
    Key decisions: {1-2 critical choices}
  - {Component}: {what it does}. {N} files.
    Key decisions: {1-2 critical choices}

Intent Coverage:
  AC-1.1 {description} → {Wave}: {Component} ✓
  AC-1.2 {description} → {Wave}: {Component} ✓
  AC-2.1 {description} → NOT MAPPED ⚠️
  Coverage: {mapped}/{total} ACs ({percentage}%)

{If gaps}: "Warning: {N} acceptance criteria not mapped to any section. 
Address before proceeding."

Review each section — any changes?
```

This is ~20-30 lines for a typical architecture. Enough to review and give feedback. The full artifact is on disk for reference; the chat shows the working summary.

**Manifest schema**: `~/.claude/taskplex/manifest-schema.json` — field definitions.

## Execution Readiness Gates (AUTHORITATIVE)

The ONLY gates that control transition from planning → implementation:

1. `planSource.userAcknowledged` = `true` — user approved the plan via AskUserQuestion
2. `workflowState.standardPlanning.executionAuthorized` = `true` — set after user approval

**⚠️ `planningChecklist` is a VESTIGIAL v1 field.** It is NOT referenced by any v2+ phase doc, hook, or gate. Do NOT block on `planningChecklist.implementationPlanReady` — it is not an authoritative gate. If it exists in the manifest, ignore it for phase transition decisions.

After the user approves the plan (via Pre-Implementation Acknowledgment), set `executionAuthorized = true`, update `manifest.phase = "implementation"`, and proceed immediately. Do NOT wait for a separate `/taskplex` invocation.

---

## Design Phase Transition

When entering planning, ensure the design phase gate is cleared:
- Set `manifest.designPhase = "planning-active"` if not already set
- For Blueprint initiative mode, this should already be set from the initiative approval phase
- For non-initiative modes, set it when the brief is approved and planning begins

This clears all `tp-design-gate` write restrictions.

---

## Route Dispatch

The route was set during init from user flags. Read `manifest.executionMode`:

| Route | When | What happens |
|-------|------|--------------|
| `light` | `--light` flag | Orchestrator writes minimal spec → single implementation agent → self-review |
| `standard` | Default (no flag, or `--standard`/`--team`) | Planning agent writes spec + sections → spec critic → 1-3 implementation agents (multi-agent parallel) → mandatory tactical critic |
| `blueprint` | `--blueprint`, `--architect`, or `--deep` flag | Opus architect → strategic + tactical critics → multi-agent implementation + worktrees + waves |

**Legacy mapping**: If `manifest.executionMode` contains old values, map them: `single→standard`, `parallel→standard`, `team→standard`, `architect→blueprint`, `prd→blueprint`.

If the architect determines the scope warrants wave decomposition: read the "Blueprint: Initiative Mode" section below. The architect controls this decision — there is no separate `initiativeMode` flag.

---

## ⚠️ Execution Continuity Rule (HARD RULE — all routes)

**The user's last checkpoint is Pre-Implementation Acknowledgment (Phase A.3).** Once the user approves the plan and implementation begins, the orchestrator runs to completion without stopping to ask.

**DO NOT:**
- ❌ Ask "should I continue?" between agent dispatches
- ❌ Pause after one agent returns to check in with the user
- ❌ Stop after a few tasks and ask to proceed
- ❌ Present intermediate results and wait for approval before the next agent
- ❌ Run implementation across multiple user turns when it can complete in one

**DO:**
- ✅ Dispatch all independent agents in a **single message** (parallel Agent tool calls)
- ✅ When an agent returns, immediately dispatch the next dependent agent if any
- ✅ Run all agents, merge results, then proceed to QA/validation — uninterrupted
- ✅ Only stop for: blocking failure, iteration limit reached, or agent escalation that needs user input

**For Standard/Blueprint routes (multi-agent) specifically:**
- All independent workers dispatch in **one message** using parallel tool calls
- Each worker MUST use `isolation: "worktree"` (Blueprint) or shared workspace with file ownership (Standard)
- After all workers return, merge and proceed to build check → QA → validation — no pause

The user trusted the plan when they approved it. Implementation executes that plan. The next user interaction is at completion (git/PR) or if something breaks.

---

## Memplex Context Assembly (before every agent spawn)

**If `manifest.memplexAvailable === true`**, the orchestrator assembles memplex context before spawning any agent. Agents cannot call MCP tools — they start from zero unless the orchestrator hydrates them.

**Assembly process** (per agent, before spawn):
1. Identify the agent's primary files (from spec, worker brief, or file ownership)
2. For each primary file (max 3):
   - `file_intelligence(file)` — coupled files, known issues, change patterns
   - `get_error_resolution(file)` — previously solved errors
3. `search_knowledge` scoped to the agent's section/task — patterns, decisions, corrections
4. Format results into a "Known Context" block and include in the agent's prompt

**Assembly template** (include in agent prompt after the spec/brief reference):
```markdown
## Known Context (from project memory)

### File Intelligence
- **{file}**: Coupled with {list}. Known issues: {list or "none"}.

### Error Resolutions
- **{error pattern}** in {file}:{line} — Resolution: {fix description}

### Relevant Knowledge
- {Pattern, decision, or convention relevant to this work}

### Corrections
- {Past corrections about this area}
```

**Budget**: Max 5-9 MCP calls per agent spawn (3 file_intelligence + 3 get_error_resolution + 1 search_knowledge + optional get_file_coupling). Should take < 5 seconds.

**If `manifest.memplexAvailable === false`**: Skip entirely. The "Known Context" block is omitted from the prompt. No placeholder, no note. The agent receives spec, brief, and conventions only.

---

## Verification Commands Assembly (before every implementation agent spawn)

**MANDATORY**: Every implementation agent handoff MUST include a `verification` block with the exact commands to run for self-verification. The orchestrator copies these from `manifest.buildCommands`.

```markdown
## Verification (MANDATORY — run before reporting completion)

Commands:
- Typecheck: {manifest.buildCommands.typecheck}
- Lint: {manifest.buildCommands.lint}
- Test: {manifest.buildCommands.test}
{If manifest.buildCommands.custom:}
- {name}: {command}

Run ALL commands. Fix failures. Do NOT report STATUS: completed if any command fails.
```

If `manifest.buildCommands` is not yet resolved (still in init phase): use project defaults based on detected type (package.json → npm, Cargo.toml → cargo, etc.).

---

## Frontend Parity Rule (MANDATORY for architect prompts)

Before spawning the architect (or planning agent), check `manifest.frontendParity.required`. If true:

1. Include in the architect prompt: "This project has frontends at: {paths}. EVERY new API endpoint MUST have a corresponding frontend task (screen/page/component). Acceptance criteria MUST include user-facing ACs in Given/When/Then format for each endpoint that has a user surface."

2. The architect's spec MUST NOT contain API endpoints without corresponding frontend work UNLESS the endpoint is explicitly marked as `internal/background` (e.g., webhook handlers, scheduled jobs, agent-only endpoints).

3. The orchestrator MUST verify the spec before spawning workers: count API endpoints vs frontend tasks. If ratio < 0.5 (fewer than half the endpoints have frontend work), flag and add frontend tasks.

---

## Light Route

### Phase A: Minimal Planning

The orchestrator writes a minimal spec directly (no planning agent spawn). This is for clear tasks with known scope.

1. **Write spec.md** inline: The orchestrator reads brief.md and writes a minimal spec with:
   - Files to modify/create
   - Key changes per file
   - Acceptance criteria from brief

Set `manifest.planFile = "spec.md"`.

### Phase A.3: Pre-Implementation Acknowledgment (MANDATORY)

Same gate as other routes — present spec to user, get approval. But the spec is shorter.

1. Present the minimal spec in the conversation.
2. Ask: **Proceed** / **Discuss changes**
3. Set `planSource.userAcknowledged = true` and `workflowState.standardPlanning.executionAuthorized = true`.

### Phase A.4: Refine Task List (MANDATORY)

**Step 1 — Delete the placeholder** "Implementation" task.

**Step 2 — Create specific tasks:**
```
TaskCreate: "Implement — {summary of spec, N files}"
TaskCreate: "Self-review — verify changes match brief"
TaskCreate: "Build gate — typecheck + lint"
```

### Phase B: Single-Agent Implementation

**⚠️ Set `manifest.implementationDelegated = true`** immediately.

1. **Lean profile (simple tasks)**: Implement directly inline.
   - Compaction guard monitors context growth
   - If compaction guard triggers: delegate to implementation agent

2. **Standard profile**: Spawn implementation agent.
   **Before spawning**: Run Memplex Context Assembly for the spec's primary files.
   > Spawn implementation-agent (sonnet) from ~/.claude/agents/core/implementation-agent.md
     Context: "Read your spec at .claude-task/{taskId}/spec.md. Implement the plan." + Known Context block (if memplex available)
     max_turns: 25
     Writes: source code changes, deferred items
     Returns: "STATUS: completed|blocked. FILES_MODIFIED: [...]. BUILD: pass|fail."

3. **Self-review** (no agent spawn — orchestrator checks inline):
   - Do modified files compile? (run build commands)
   - Do changes match the brief's ACs?
   - Any orphaned endpoints? (if frontend parity required)
   Quick 30-second check. If issues found, fix inline.

4. **Build gate**: Run typecheck + lint + tests. Build-fix rounds per policy `limits.buildFixRounds`.

5. **Convention scan**: Read conventions.json or CONVENTIONS.md. Grep modified files for violations. Fix inline. Budget: 3-6 Grep calls.

7. **Update documentation** (if needed — same rules as Standard route).

**Git notes**: No incremental commits. One commit at completion.

7. **Proceed to QA**: Read `~/.claude/taskplex/phases/qa.md`

---

## Standard Route

### Phase A: Planning Agent with Section Assignment

**Pre-spawn status** (tell the user what's happening):
> "Spawning the planning agent to design the implementation. This typically takes 3-8 minutes.
> The agent will read the brief, analyze the target area, write a detailed spec, and assign sections for parallel execution."

**Before spawning**: Run Memplex Context Assembly (see section above) for the target area files from brief.md.

Spawn the planning agent in multi-agent mode. The planning agent identifies independent sections and writes file ownership.

> Spawn planning-agent (sonnet) from ~/.claude/agents/core/planning-agent.md
  Context: brief.md path, taskId, designDepth, qualityProfile, mode: "multi-agent"
  Writes: .claude-task/{taskId}/spec.md, .claude-task/{taskId}/sections.json, .claude-task/{taskId}/file-ownership.json, .claude-task/{taskId}/conventions-snapshot.json
  Returns: "PLANNING COMPLETE. Spec: {path}. Sections: {N}. Files affected: {N}. Key decisions: {bullets}"

Set `manifest.planFile = "spec.md"`.

### Phase A.1: Research (conditional)

**Skip if**: All technologies already in codebase, or cm has recent research.

**Research triggers** (any of these):
- New dependency not in codebase
- External API integration
- Version migration
- Unfamiliar pattern (planning agent flagged uncertainty)
- User explicitly requested research

If triggered:
> "Spawning researcher to investigate {topics}. This typically takes 2-5 minutes."

> Spawn researcher (sonnet) from ~/.claude/agents/core/researcher.md
  Context: brief.md, spec.md, package.json deps, CONVENTIONS.md, research questions
  Writes: .claude-task/{taskId}/research/*.md
  Returns: "RESEARCH COMPLETE. Questions: N. Key findings: {bullets}"

After researcher returns: **present key findings inline** (don't just say "research complete"). Summarize the findings that affect the plan.

### Phase A.2: Spec Critic (mandatory)

After planning agent returns, spawn a spec reviewer:

> Spawn closure-agent (haiku) from ~/.claude/agents/core/closure-agent.md
  Context: spec.md, sections.json, brief.md, CONVENTIONS.md — review spec + section assignments against brief requirements
  Returns: "Verdict: APPROVED|NEEDS_REVISION"

Track in `manifest.iterationCounts.reviewRounds.specCritic`.

- If NEEDS_REVISION: feed reviewer feedback back to planning agent (re-spawn with feedback context). Max 2 rounds per policy `limits.reviewResubmissions`.
- If APPROVED: proceed to Pre-Implementation Acknowledgment.

### Phase A.3: Pre-Implementation Acknowledgment (MANDATORY)

**Present a structured working summary — not the full spec, not a 3-line summary.**

1. Read `.claude-task/{taskId}/spec.md` and `.claude-task/{taskId}/brief.md`

2. **Build intent traceability matrix:**
   - Extract all ACs from brief.md
   - Map each AC to the spec section that addresses it
   - Flag any unmapped ACs

3. **Present in the conversation** using the structured summary format:

   For each section/worker:
   - What it does (1 sentence)
   - Files involved (list)
   - Key decisions (1-2 per section)
   - Which ACs it addresses

   Then show intent coverage:
   ```
   Intent Coverage:
     AC-1.1 User login → Section 1: Auth ✓
     AC-1.2 Bcrypt hashing → Section 1: Auth ✓
     AC-2.1 Dashboard stats → Section 2: Dashboard ✓
     Coverage: 3/3 ACs mapped (100%)
   ```

4. After presenting, ask:
   > "Plan: {N} sections, {M} files. Intent coverage: {mapped}/{total} ACs.
   > {If gaps: 'Warning: {unmapped ACs} not addressed.'}
   >
   > **Proceed** / **Discuss changes** / **Revise sections**"

**Do NOT** dump the full spec.md. The structured summary IS the review surface.
**Do NOT** say "review spec.md" and point to a file.

Write intent matrix to `.claude-task/{taskId}/intent-traceability.md`.

Set `planSource.userAcknowledged = true` and `workflowState.standardPlanning.executionAuthorized = true`.

### Phase A.3b: Verification Test Plan (Standard + Blueprint — skip for Light)

**Before implementation begins**, the verification agent reads the spec and pre-commits what it will test. This creates a "sprint contract" — the implementation agent knows exactly what the verification agent will check, leading to better implementation.

> Spawn verification-agent (sonnet) from $TASKPLEX_HOME/agents/core/verification-agent.md
  Context: spec.md, brief.md — MODE: "test-plan" (not "verify")
  Writes: .claude-task/{taskId}/test-plan.md
  Returns: "Test plan: N happy path checks, P adversarial probes planned"

The verification agent produces a **test plan** (not test execution):

```markdown
# Verification Test Plan

## Happy Path Checks (will run after implementation)
1. POST /api/register with valid data → expect 201 + user object
2. POST /api/login with valid credentials → expect 200 + JWT
3. Auth middleware rejects expired token → expect 401

## Adversarial Probes (will run after implementation)
1. [Boundary] POST /api/register with empty email → expect 400
2. [Idempotency] POST /api/register same email twice → expect 409 or graceful no-op
3. [Injection] POST /api/register with SQL in email → expect 400, no DB corruption
4. [Auth] Access /api/protected without token → expect 401

## Commands I Will Run
- curl POST /api/register with valid/invalid payloads
- curl POST /api/login with valid/expired/missing credentials
- curl GET /api/protected with/without token
```

**Include the test plan in the implementation agent's context.** The implementation agent sees what will be tested and can build accordingly. This is not "teaching to the test" — it's making the acceptance criteria concrete and executable.

### Phase A.4: Refine Task List (MANDATORY — STOP and do this NOW)

**You MUST refine the task list immediately after the user approves the plan.** Do NOT proceed to implementation until the task list reflects the actual plan. The generic "Implementation" placeholder must be replaced with specific tasks.

**Do this RIGHT NOW using TaskUpdate and TaskCreate:**

**Step 1 — Delete the placeholder** "Implementation" task (TaskUpdate with status: deleted).

**Step 2 — Create specific implementation tasks:**
```
TaskCreate: "Worker 1: {section title} — {file count} files"
TaskCreate: "Worker 2: {section title} — {file count} files"
TaskCreate: "Worker 3: {section title} — {file count} files" (if applicable)
TaskCreate: "Coherence check — verify each worker matches spec"
TaskCreate: "Merge workers + build gate"
TaskCreate: "Tactical critic review"
TaskCreate: "Update documentation"
```

For Blueprint route, call TaskCreate for each:
```
TaskCreate: "Present architecture to user"
TaskCreate: "Worker 1: {section title} (worktree)"
TaskCreate: "Worker 2: {section title} (worktree)"
TaskCreate: "Worker 3: {section title} (worktree)" (if applicable)
TaskCreate: "Coherence check — verify each worker matches spec"
TaskCreate: "Merge worktrees + build gate"
TaskCreate: "Tactical critic review"
TaskCreate: "Update documentation"
```

**Step 3 — Refine QA and Validation placeholders:**
```
TaskUpdate: "QA" → "QA — {method} ({product type})" (e.g., "QA — browser walkthrough (web app)")
TaskUpdate: "Validation" → "Validation — {profile} profile ({N} gates)"
```

**Step 4 — Add documentation task detail** (based on what the spec modifies):
- API endpoints changed? → "Update API documentation"
- New feature? → "Update README with feature description"
- Config/env changes? → "Update setup instructions"
- DB schema changes? → "Update migration notes"
- CHANGELOG exists? → "Add CHANGELOG entry"
- If nothing needs docs: delete the docs task

**The user must see the refined task list before implementation begins.** This is their view into what's about to happen.

### Phase B: Multi-Agent Implementation

**⚠️ HARD RULES (Standard multi-agent implementation)**:
1. The orchestrator MUST NOT edit source code directly — delegate to agents
2. Independent agents MUST be dispatched in a **single message** (parallel Agent tool calls)
3. Do NOT pause between agent dispatches — see Execution Continuity Rule above
4. Run straight through: dispatch all agents → merge → build check → tactical critic → QA → validation
5. The `tp-design-gate` hook blocks orchestrator source edits until `manifest.implementationDelegated = true`

**Handoff contract**: `~/.claude/taskplex/handoff-contract.md` — structured format for all agent transitions.

1. **Read sections.json** for independent section assignments.

2. **Spawn 1-3 implementation agents** (one per independent section).
   **Dispatch ALL independent agents in a single message using parallel Agent tool calls.**
   **After all agents are dispatched**, set `manifest.implementationDelegated = true`.

   Before spawning, write handoff records. Then spawn all at once:

   Before spawning each agent, write an assignment handoff to `manifest.workerHandoffs[]`:
   ```json
   {
     "workerId": "worker-{section-slug}",
     "direction": "orchestrator→worker",
     "fromAgent": "orchestrator",
     "toAgent": "implementation-agent:worker-{section-slug}",
     "phase": "implementation",
     "context": {
       "currentState": "Planning complete, spec approved",
       "relevantFiles": [{ "path": "spec.md#section-N", "role": "Implementation spec for this section" }],
       "dependencies": "List any cross-section dependencies",
       "constraints": "Max 25 turns. Follow CONVENTIONS.md."
     },
     "deliverable": {
       "description": "Implement Section N: {title}",
       "acceptanceCriteria": ["AC from spec for this section"],
       "evidenceRequired": "Build passes, all acceptance criteria met with file:line citations"
     },
     "timestamp": "ISO"
   }
   ```

   **Before spawning each agent**: Run Memplex Context Assembly for that section's primary files.

   > Spawn implementation-agent (sonnet) from ~/.claude/agents/core/implementation-agent.md
     Context: "Read spec at .claude-task/{taskId}/spec.md, section: {section-id}. Read file-ownership at .claude-task/{taskId}/file-ownership.json." + Known Context block (if memplex available)
     max_turns: 25
     Writes: source code changes, deferred items, workers/{workerId}.json
     Returns: "STATUS: completed|blocked. FILES_MODIFIED: [...]. BUILD: pass|fail."

   **Worker sizing**: Each agent owns at most 10-12 files. Split if larger.

   On each agent return:
   - Read `workers/{workerId}.json` for structured handoff data
   - Append completion handoff to `manifest.workerHandoffs[]`
   - Append to `manifest.implementationAgents[]`
   - Merge FILES_MODIFIED into `manifest.modifiedFiles`
   - If blocked: read escalation report, append to `manifest.escalations[]`, continue with remaining agents

**Git notes**: No incremental commits. One commit at completion.

3. **Implementation coherence check** (per worker, fast):
   For each completed worker, spawn a quick coherence check:
   > Spawn closure-agent (haiku) — "Does worker {N}'s output match its spec section?"
   - If DRIFT on any worker: one revision round for that worker, then proceed
   - Workers can be coherence-checked in parallel (one haiku call each)

4. **Build gate** (after all agents return + coherence verified): Run typecheck + lint + tests. If failures, spawn build-fixer (max rounds per policy). This catches integration issues between workers before moving to critic review.

5. **Tactical critic review** (MANDATORY — see Critic Review section below).

6. **Update documentation** (same rules as before — update docs based on modified files).

7. **Proceed to QA → Full Validation**: Read `~/.claude/taskplex/phases/qa.md`, then `~/.claude/taskplex/phases/validation.md`. Full validation runs once after all workers complete and build gate passes.

---

## Critic Review (MANDATORY for Standard and Blueprint routes)

### Standard Route: Tactical Critic
After implementation completes and build gate passes, spawn a tactical critic agent (sonnet):
- Reviews all modified files
- Checks: patterns followed, edge cases handled, tests sufficient, frontend parity
- Returns: APPROVED or REVISE with specific file-level feedback
- If REVISE: orchestrator fixes issues, re-runs critic (max 2 cycles)

> Spawn tactical-critic (sonnet) from ~/.claude/agents/core/tactical-critic.md
  Context: spec.md, manifest.modifiedFiles, brief.md, CONVENTIONS.md
  Writes: .claude-task/{taskId}/reviews/tactical-review.md
  Returns: "Verdict: APPROVED|REVISE. {N} issues found."

Track in `manifest.iterationCounts.reviewRounds.tacticalCritic`.

### Blueprint Route: Strategic + Tactical Critics
- **Strategic critic** runs after architect produces spec (before implementation)
- **Tactical critic** runs after each wave's implementation
- Both must approve before proceeding

### Light Route: Self-Review
No agent spawn. Orchestrator checks:
1. Do modified files compile?
2. Do changes match the brief's ACs?
3. Any orphaned endpoints? (if frontend parity required)
Quick 30-second check. If issues found, fix inline.

---

## Context Management Philosophy

All task state lives ON DISK, not in the orchestrator's context window.

- `manifest.json` — authoritative task state (phase, progress, wave status)
- `brief.md` — approved design (survives any compaction)
- `spec.md` — implementation plan (survives any compaction)
- `workers/*.md` — agent briefs (survive compaction)
- `progress.md` — rendered from manifest.progressNotes by heartbeat hook
- `checkpoints/` — pre-compaction snapshots

The orchestrator's context window is TRANSIENT. Everything important is written to disk.
After compaction, the orchestrator reads manifest.json and resumes. No information is lost
IF the manifest is kept current.

**This means**: The context window is NOT a constraint on task size. A 17-feature, 4-wave
blueprint can execute in a single /tp invocation because the orchestrator reads from disk
at each phase transition, not from memory of previous phases.

Agents receive their context via prompts (assembled from spec + worker briefs), not from
the orchestrator's memory. Agent results are written to disk, not held in context.

---

## Mandatory Manifest Updates (ALL ROUTES)

The orchestrator MUST update the manifest at these points. These are NOT optional.
The heartbeat hook will warn if updates are missed.

### After each design sub-phase completes:
```json
manifest.phaseChecklist.{subphase} = "completed"
manifest.progressNotes.push({
  "phase": "{phase}",
  "note": "{what was completed}",
  "timestamp": "ISO"
})
```

### After each wave completes (Blueprint):
```json
manifest.waveProgress.{waveId}.status = "completed"
manifest.waveProgress.{waveId}.validation = { passed: true/false, details: "..." }
manifest.waveProgress.{waveId}.filesModified = [...]
manifest.progressNotes.push({
  "phase": "implementation",
  "note": "Wave {N} complete: {summary}. Validation: {passed/failed}.",
  "timestamp": "ISO"
})
```

### After each agent returns:
```json
manifest.progressNotes.push({
  "phase": "implementation",
  "note": "Agent {id} complete: {summary}",
  "timestamp": "ISO"
})
```

### After validation:
```json
manifest.phaseChecklist.validation = "completed"
manifest.progressNotes.push({
  "phase": "validation",
  "note": "All gates passed. Journey verification: {X}/{Y}. Smell test: {passed}.",
  "timestamp": "ISO"
})
```

Failure to update these fields means recovery after compaction will have stale state.

---

## Blueprint Route

### Phase A: Architecture Decisions

1. **Architecture Decision Presentation**: Analyze brief + codebase, present key decisions to user via `AskUserQuestion`:
   - State each decision clearly with 2-4 options and trade-offs
   - Convention conflict → 3-option pattern: Follow / Deviate / Update
   - Budget: 1-3 `AskUserQuestion` calls
   - **Interaction-first**: First question within 30 seconds. Do minimal research before the first architecture question.

2. **Architect Design Review**:
   The architect agent's design loop includes strategic and tactical review internally. Critics are NOT separate agent spawns — they are part of the architect's reasoning process.

   **Pre-spawn status**:
   > "Spawning the architect agent (opus). This typically takes 5-15 minutes for complex features.
   > The architect will analyze the codebase, design the architecture, create a spec, and write
   > worker briefs for each implementation section.
   > You can check progress: `cat .claude-task/{taskId}/architecture.md`"

   **Before spawning**: Run Memplex Context Assembly for the target area files from brief.md.

   > Spawn architect (opus) from ~/.claude/agents/core/architect.md
     Context: brief.md, architecture decisions, CONVENTIONS.md, CLAUDE.md, .schema/ docs, qualityProfile + Known Context block (if memplex available)
     Writes: .claude-task/{taskId}/architecture.md, .claude-task/{taskId}/spec.md, .claude-task/{taskId}/file-ownership.json, .claude-task/{taskId}/workers/worker-{n}-brief.md
     Returns: "Architecture designed. {N} components, {M} files affected."

   Set `manifest.planFile = "spec.md"`.

   **Path guardrails**: Architect writes ONLY to `.claude-task/{taskId}/`. Never to source paths.

3. **Build intent traceability matrix + present architecture to user**

   After the architect returns:
   
   **Step 3a — Intent traceability contract:**
   a. Read `.claude-task/{taskId}/brief.md` — extract ALL acceptance criteria (AC-N.M)
   b. Read `.claude-task/{taskId}/architecture.md` and `spec.md`
   c. For each AC, find which wave/component/section addresses it
   d. Build the matrix:
      ```
      Intent Coverage:
        AC-1.1 User can login → Wave 0: Auth module ✓
        AC-1.2 Bcrypt hashing → Wave 0: Auth module ✓
        AC-2.1 Real-time stats → Wave 1: Dashboard ✓
        AC-3.1 Rate limiting → NOT MAPPED ⚠️
        Coverage: 3/4 ACs mapped (75%)
      ```
   e. If any AC is unmapped: flag it BEFORE presenting to user
   f. Write matrix to `.claude-task/{taskId}/intent-traceability.md`

   **Step 3b — Present structured summary:**
   Use the format from User Visibility Rules. For each wave:
   - Component name, what it does, file count
   - Key decisions (1-2 per component)
   - Which ACs it addresses
   
   Then show the intent coverage matrix.
   
   Then ask:
   > "Architecture: {N} components across {W} waves. Intent coverage: {mapped}/{total} ACs.
   > {If gaps: 'Warning: {unmapped ACs} not addressed — add to a wave or confirm deferred.'}
   >
   > **Proceed** / **Revisit architecture** / **Modify sections** / **Cancel**"

   **Do NOT** dump the full architecture.md. Present the structured working summary.
   **Do NOT** say "review architecture.md" — the summary IS the review surface.

### Phase B: Pre-Implementation Research (conditional)

**Skip if**: All technologies already in codebase, or cm has recent research.

**Research triggers**: New dependency, external API, version migration, unfamiliar pattern, explicit request.

If triggered:
> Spawn researcher (sonnet) from ~/.claude/agents/core/researcher.md
  Context: brief.md, package.json deps, CONVENTIONS.md, research questions
  Writes: .claude-task/{taskId}/research/*.md
  Returns: "RESEARCH COMPLETE. Questions: N. Key findings: {bullets}"

### Phase C: Multi-Agent Implementation

**⚠️ HARD RULES (Blueprint implementation)**:
1. The orchestrator MUST NOT edit source code directly — ALL implementation goes through agents
2. Every agent MUST use `isolation: "worktree"` — no exceptions
3. Independent agents MUST be dispatched in a **single message** (parallel Agent tool calls)
4. Do NOT pause between agent dispatches — see Execution Continuity Rule above
5. Run straight through: dispatch all agents → merge results → build check → QA → validation
6. The `tp-design-gate` hook blocks orchestrator source edits until `manifest.implementationDelegated = true`

4. **Spawn implementation agents** (fan-out, non-stop):

   **Independent sections** — dispatch ALL in one message using parallel Agent tool calls:
   ```
   // In a SINGLE response, call Agent multiple times:
   Agent({ prompt: "worker-1 brief...", isolation: "worktree" })
   Agent({ prompt: "worker-2 brief...", isolation: "worktree" })
   Agent({ prompt: "worker-3 brief...", isolation: "worktree" })
   ```

   **Dependent sections** — dispatch sequentially, but immediately (no user check-in between):
   ```
   Agent({ prompt: "worker-1 (foundation)...", isolation: "worktree" })
   // worker-1 returns → immediately dispatch worker-2
   Agent({ prompt: "worker-2 (depends on worker-1)...", isolation: "worktree" })
   ```

   **Before spawning**: Run Memplex Context Assembly for each worker's primary files (from worker brief).

   Each agent:
   > Spawn implementation-agent (sonnet) from ~/.claude/agents/core/implementation-agent.md
     Context: "Read your brief at .claude-task/{taskId}/workers/worker-{n}-brief.md. Implement the plan." + Known Context block (if memplex available)
     max_turns: 30
     **isolation: "worktree"** ← MANDATORY for Blueprint
     Writes: source code changes, deferred items
     Returns: "STATUS: completed|blocked. FILES_MODIFIED: [...]. BUILD: pass|fail."

   **After all agents are dispatched**, set `manifest.implementationDelegated = true`.

   **Post-worker coherence check + merge** (after all workers return):
   1. **Coherence check per worker** (before merge): For each completed worker, spawn closure-agent (haiku) to verify output matches its worker brief. If DRIFT: one revision round in the worktree, then proceed. Check workers in parallel.
   2. Merge each worker branch: `git merge --no-ff worktree-{worker}`
   3. If conflict: attempt auto-resolve, else report to user
   4. Worker commits: `git commit -m "wip(worker-{N}): {section}"`

5. **Build gate** (after merge + coherence verified): Run typecheck + lint + tests on merged result. If failures, spawn build-fixer (max rounds per policy). This catches integration issues between workers before moving to QA.

6. **Update documentation** (same as Standard route step 6 — update docs based on modified files).

7. **Proceed to QA → Full Validation**: Read `~/.claude/taskplex/phases/qa.md`, then `~/.claude/taskplex/phases/validation.md`. Full validation (security, closure, code review, hardening, compliance) runs once after all workers are merged and build gate passes.

---

## Blueprint: Initiative Mode

**Activated when**: The architect determines the scope warrants wave decomposition (based on feature count, complexity, and cross-cutting concerns). `--prd` is an alias for `--blueprint` — the architect always evaluates whether initiative mode is appropriate.

Initiative mode is Blueprint at scale — the same opus architect + critics + multi-agent execution, but with feature decomposition, wave-based execution, and cross-feature closure.

### Initiative Phase 0.5: Product Brief

**HARD RULE:** By the time initiative mode begins, the user has ALREADY been through the full design interaction in init.md. The brief.md has been written from that approved design.

Run the Product Brief agent to formalize the approved brief into initiative format:

> Read ~/.claude/agents/core/product-brief.md
  Mode: Initiative (full brief with component-grouped user stories)

The product-brief agent:
1. Reads the already-approved brief.md
2. Restructures into initiative-format brief (component-grouped user stories, testable success criteria)
3. Asks MVP vs Full scope decision + 1-2 initiative-specific questions (feature granularity, wave strategy)
4. Finalizes the brief

After brief is confirmed, set `manifest.designInteraction.prdBriefConfirmed = true`.

### Initiative Phase 1: Bootstrap

**→ Set `manifest.designPhase = "prd-bootstrap"` before spawning.**

> Spawn prd-bootstrap (opus) from ~/.claude/agents/core/prd-bootstrap.md
  Context: user description, product brief, INTENT.md, CONVENTIONS.md, CLAUDE.md, .schema/ overviews
  Writes: .claude-task/PRD-{timestamp}/prd.md, .claude-task/PRD-{timestamp}/prd-state.json
  Returns: "PRD bootstrapped: {N} features across {W} waves"

### Initiative Phase 1.5: Strategic Critic Reviews

**→ Set `manifest.designPhase = "prd-critic"` before spawning.**

> Spawn strategic-critic (opus) from ~/.claude/agents/core/strategic-critic.md
  Context: prd.md, INTENT.md, CONVENTIONS.md — review the PRD as a whole
  Writes: .claude-task/PRD-{id}/strategic-review.md
  Returns: "Verdict: APPROVED|NEEDS_REVISION"

- If NEEDS_REVISION: update prd.md per feedback, re-review (max per policy `limits.reviewResubmissions`)
- If REJECTED (deadlock after limit): mark PRD as `blocked:critic-deadlock`, notify user
- If APPROVED: continue

### Initiative Phase 2: Tactical Critic Reviews

> Spawn tactical-critic (sonnet) from ~/.claude/agents/core/tactical-critic.md
  Context: prd.md, CONVENTIONS.md, .schema/ docs — review per-feature specs
  Writes: .claude-task/PRD-{id}/tactical-review.md
  Returns: "Verdict: APPROVED|NEEDS_REVISION"

- Same escalation rules as Phase 1.5

### Initiative Phase 2b: User Reviews Initiative Plan

**→ Set `manifest.designPhase = "prd-approval"` before presenting to user.**

**HARD RULE — STOP AND PRESENT TO THE USER. DO NOT AUTO-PROCEED.**

Present the critic-reviewed plan to the user:
1. **Initiative Overview**: Problem statement, target users, success criteria
2. **Feature List**: Each feature with ID, name, complexity, dependencies, description
3. **Wave Plan**: Which features execute in which wave, and why
4. **Risk Assessment**: Key risks, mitigations, assumptions
5. **Critic Feedback Summary**: What the critics flagged and what was changed

Then ask:
- `AskUserQuestion`: **Approve and proceed** / **Modify features** / **Cancel**

After user approves: Set `manifest.designInteraction.prdUserReviewed = true`.

### Initiative Phase 2c: Choose Execution Mode

`AskUserQuestion`: "How should features be executed?"

**Autonomous mode** (Default for one-shot execution):
- Critic APPROVED = auto-proceed (no user confirmation between waves)
- All waves execute back-to-back without stopping
- Drift from spec is logged but not prompted

**Interactive mode**:
- Single confirmation per WAVE (not per feature)
- After the user confirms a wave, all features in that wave execute non-stop

**→ After user confirms, set `manifest.designInteraction.prdExecutionModeChosen = true` and `manifest.designPhase = "planning-active"`**

### Initiative Phase 2d: Branch Strategy + Wave Progress Init

If `manifest.git.available === true` and `manifest.git.config.createBranch === true`:
- Each wave gets its own branch: `feat/prd-{id}-wave{N}`
- Wave 0 branch created from `manifest.git.baseBranch`
- Subsequent wave branches created from the merged result of the previous wave

**Initialize `manifest.waveProgress`** at this point (MANDATORY):
```json
"waveProgress": {
  "wave-1": { "name": "Wave 1 Name", "status": "pending", "features": ["F1","F2","F3"], "validation": null, "filesModified": [] },
  "wave-2": { "name": "Wave 2 Name", "status": "pending", "features": ["F4","F5","F6"], "validation": null, "filesModified": [] }
}
```

## Blueprint Execution: One-Shot Wave Loop

After design approval (strategic critic APPROVED, user confirms), the blueprint enters
a **continuous execution loop** that runs ALL waves to completion without stopping.

### The Loop

```
FOR each wave in architect's spec:
  1. READ manifest.json (current state — NOT from memory)
  2. READ spec.md wave section (agent briefs for this wave)
  3. UPDATE manifest.waveProgress.{waveId}.status = "in-progress"
  4. SPAWN parallel agents for this wave (worktrees)
  5. COLLECT agent results (summaries only — details on disk)
  6. MERGE agent changes into main tree
  7. UPDATE manifest:
     - waveProgress.{waveId}.status = "completed"
     - progressNotes.push(wave summary)
     - modifiedFiles += new files
  8. VALIDATE this wave:
     - Build checks (cargo check / npm run build)
     - Tests (cargo test / npm test)
     - Journey verification (new endpoints → frontend consumers?)
     - Cross-wave check (does this wave's output integrate with previous waves?)
     - If FAIL: fix inline (up to 2 retries). If still fails: pause and inform user.
     - If PASS: continue to next wave
  9. CHECKPOINT: manifest is already on disk (heartbeat updates it continuously)
END FOR
```

### Rules

1. **No per-wave approval**: User approved the plan once. Waves execute continuously.
2. **No per-wave route switching**: Blueprint rigor throughout. Every wave gets the same
   validation depth.
3. **No stopping between waves**: After Wave N validation passes, Wave N+1 starts immediately.
4. **Only stop for**: Validation failure after 2 retries, OR agent escalation, OR user interrupt.
5. **Read from disk at each wave start**: The orchestrator reads manifest.json and spec.md
   at the start of each wave. This ensures correct state even if compaction occurred.

### Wave Validation Gate (MANDATORY)

Every wave gets full validation:

1. **Build check**: All crates/packages compile
2. **Test check**: All tests pass (including new tests from this wave)
3. **Journey check**: New API endpoints have frontend consumers (if frontendParity.required)
4. **Cross-wave check**:
   - Wave N's frontend code correctly calls Wave N-1's API endpoints
   - Shared types are consistent across waves
   - No broken imports between wave boundaries
5. **Regression check**: Previous waves' tests still pass

If any check fails:
- Attempt inline fix (orchestrator or spawned fixer agent)
- Re-run failed checks
- Max 2 fix attempts per wave
- If still failing after 2 attempts: STOP, inform user with specific failure details

### Cross-Wave Validation

After each wave (starting from Wave 2), verify integration with previous waves:

```
Wave 2 validation:
  - Do mobile screens (Wave 2) call the API endpoints added in Wave 1?
  - Grep mobile code for Wave 1 endpoint paths
  - If gaps found: add to fix list

Wave 3 validation:
  - Do dashboard pages (Wave 3) call the same endpoints as mobile (Wave 2)?
  - Are shared types consistent?
  - If gaps found: add to fix list
```

### Wave Agent Dispatch

For each wave:

1. **Wave 0** (sequential — foundational features):
   For each feature in wave 0, dispatch immediately one after another:
   - Update prd-state.json: feature status = `in-progress`
   - Check attempt count (max per policy `limits.prdFeatureAttemptsAutonomous` in autonomous mode)
   - Each feature gets architect-level worker brief + implementation agent dispatch
   > Spawn implementation-agent (sonnet) from ~/.claude/agents/core/implementation-agent.md
     Context: assembled payload, feature requirements, max_turns: 30
     **isolation: "worktree"** ← MANDATORY
     Writes: source code changes, deferred items
     Returns: "STATUS: completed|blocked. FILES_MODIFIED: [...]. BUILD: pass|fail."
   - On return: update manifest + prd-state.json, **immediately dispatch next feature**
   - If failed: mark dependent features as `blocked`, continue to next — do NOT stop to ask

2. **Wave 1+** (parallel — independent features):
   - Dispatch ALL features in the wave in a **single message** using parallel Agent tool calls
   - Every agent MUST use `isolation: "worktree"`
   - After all return: merge, update state, run wave validation, proceed to next wave immediately

### Wave Merge

After each wave completes:
1. List completed feature branches
2. Apply merge strategy (from `manifest.git.config.mergeStrategy`, default `squash`)
3. If conflicts:
   > Spawn merge-resolver (sonnet) from ~/.claude/agents/core/merge-resolver.md
     Context: source branch, target branch, conflict files, feature descriptions
     Returns: "Merge resolved: {N} files. Typecheck: PASS/FAIL"
   - If unresolvable: mark as `blocked:merge-conflict`, notify user

4. Update prd-state.json: wave status = `completed`

### Final Validation (after all waves)

After the last wave completes and passes validation:

1. **Full journey verification**: Trace EVERY acceptance criterion from the brief
   to working code across ALL waves
2. **Product smell test**:
   - Can a user sign up and use every feature?
   - Does every endpoint have a consumer?
   - Would I ship this today?
3. **Tactical critic**: Spawns critic agent to review the ENTIRE changeset
   (not just last wave)
4. **Cross-wave integration test**: Run full test suite one final time
5. **Full Validation Pipeline** — runs ONCE after all waves complete. This is where
   security review, closure, code review, hardening, and compliance run.
   See `~/.claude/taskplex/phases/validation.md`.
6. Generate completion report
7. If git available: create comprehensive PR covering the entire initiative
8. Present to user
9. Clean up: archive prd-state.json

### Wave Progress in Manifest

Initialize at planning time (Phase 2d):
```json
"waveProgress": {
  "wave-1": { "name": "Backend Hardening", "status": "pending", "features": ["F1","F2","F3","F4","F5"], "validation": null, "filesModified": [] },
  "wave-2": { "name": "Mobile Completion", "status": "pending", "features": ["F6a","F6","F7","F8","F9","F10"], "validation": null, "filesModified": [] },
  "wave-3": { "name": "Dashboard Completion", "status": "pending", "features": ["F11","F12","F13","F14"], "validation": null, "filesModified": [] },
  "wave-4": { "name": "CI/CD", "status": "pending", "features": ["F15","F16"], "validation": null, "filesModified": [] }
}
```

Update after each wave completes:
```json
"waveProgress": {
  "wave-1": { "status": "completed", "validation": { "build": "pass", "tests": "pass", "journeys": "8/8", "crossWave": "n/a" }, "filesModified": ["core/src/vps.rs", "..."] },
  "wave-2": { "status": "in-progress", "..." : "..." }
}
```

### Initiative prd-state.json Schema

```json
{
  "prdId": "PRD-{timestamp}",
  "title": "Initiative Title",
  "status": "in-progress|completed|paused|blocked",
  "executionMode": "interactive|autonomous",
  "createdAt": "ISO timestamp",
  "features": {
    "F0": {
      "title": "Feature title",
      "complexity": 3,
      "wave": 0,
      "status": "pending|in-progress|completed|failed|blocked",
      "dependencies": [],
      "worker": "worktree-branch-name|null",
      "phase": "planning|spec-review|implementing|code-review|validating|done",
      "startedAt": "ISO|null",
      "completedAt": "ISO|null",
      "blockedBy": "merge-conflict|critic-deadlock|dependency-F{N}|null",
      "notifications": []
    }
  },
  "waves": {
    "0": { "status": "pending|in-progress|completed", "features": ["F0"] },
    "1": { "status": "pending|in-progress|completed", "features": ["F1", "F2"] }
  },
  "currentWave": 0,
  "mergeResults": {},
  "finalReport": null
}
```

Note: The per-feature `route` field has been removed. All features use Blueprint execution.

### Initiative Failure Handling

- **Feature failure**: Mark feature as `failed`, mark dependents as `blocked` (not `skipped` — blocked is resumable)
- **Merge conflict**: Try merge-resolver. If unresolvable, mark as `blocked:merge-conflict`
- **Critic deadlock**: After 2 revision rounds, mark as `blocked:critic-deadlock`
- **User can resume**: `blocked` features can be retried. `skipped` is terminal.

### Initiative Resume Flow

On session start (handled by session-start hook):
1. Scan `.claude-task/PRD-*` directories for `prd-state.json`
2. If found with `status: 'in-progress'` or `status: 'paused'`:
   - Display status to user
   - User runs `/tp --prd` to resume
3. Resume picks up from last incomplete wave/feature

---

## External PRD Handling

If `manifest.planSource.origin === "external-file"`: the task has an external PRD that didn't go through the lifecycle. The PRD is **requirements input**, not a validated plan. The Blueprint route MUST still run the full pipeline — brief generation, architecture decisions, critics, all standard artifacts.

---

## Next

Read the QA phase: `~/.claude/taskplex/phases/qa.md`
After QA, read the validation pipeline: `~/.claude/taskplex/phases/validation.md`
