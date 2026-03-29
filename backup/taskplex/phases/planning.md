# taskplex: Planning & Implementation
<!-- Loaded by orchestrator after initialization. Self-contained. -->
<!-- v3: 3-route architecture (Standard/Team/Blueprint) with universal planning agent -->
<!-- v4: Added progress visibility, inline artifact presentation, pre-spawn status messages -->

**Policy reference**: `~/.claude/taskplex/policy.json` for iteration limits.

## ⚠️ User Visibility Rules (HARD RULE — all routes)

**The user must see what's happening at all times.**

1. **Before every agent spawn**: Tell the user what agent is being spawned, what it will do, and roughly how long it takes.
2. **After every agent returns**: Read the agent's output artifact and **present it in the conversation**. Do NOT say "review file X" — output the content inline where the user is reading.
3. **Present artifacts section by section** for long outputs (spec, architecture). Let the user digest each part.
4. **Never leave the user waiting with no information** for more than 30 seconds. If an agent is running, the pre-spawn message covers this. If you're doing research, say what you're looking at.

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
| `standard` | Default | Planning agent writes spec → spec critic → 1 implementation agent |
| `team` | `--team` or `--parallel` flag | Planning agent writes spec + sections → spec critic → 1-3 implementation agents |
| `blueprint` | `--blueprint`, `--architect`, or `--deep` flag | Opus architect → critics → multi-agent implementation + worktrees |

**Legacy mapping**: If `manifest.executionMode` contains old values, map them: `single→standard`, `parallel→team`, `architect→blueprint`, `prd→blueprint` (with `initiativeMode: true`).

If `manifest.initiativeMode === true`: read the "Blueprint: Initiative Mode" section below.

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

**For Team/Blueprint routes specifically:**
- All independent workers dispatch in **one message** using parallel tool calls
- Each worker MUST use `isolation: "worktree"` (Blueprint) or shared workspace with file ownership (Team)
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

## Standard Route

### Phase A: Planning Agent

**Pre-spawn status** (tell the user what's happening):
> "Spawning the planning agent to design the implementation. This typically takes 3-8 minutes.
> The agent will read the brief, analyze the target area, and write a detailed spec."

**Before spawning**: Run Memplex Context Assembly (see section above) for the target area files from brief.md.

> Spawn planning-agent (sonnet) from ~/.claude/agents/core/planning-agent.md
  Context: brief.md path, taskId, designDepth, qualityProfile + Known Context block (if memplex available)
  Writes: .claude-task/{taskId}/spec.md, .claude-task/{taskId}/conventions-snapshot.json
  Returns: "PLANNING COMPLETE. Spec: {path}. Files affected: {N}. Key decisions: {bullets}"

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
  Context: spec.md, brief.md, CONVENTIONS.md — review spec against brief requirements, not code
  Returns: "Verdict: APPROVED|NEEDS_REVISION"

Track in `manifest.iterationCounts.reviewRounds.specCritic`.

- If NEEDS_REVISION: feed reviewer feedback back to planning agent (re-spawn with feedback context). Max 2 rounds per policy `limits.reviewResubmissions`.
- If APPROVED: proceed to Pre-Implementation Acknowledgment.

### Phase A.3: Pre-Implementation Acknowledgment (MANDATORY)

**Present the spec to the user — do NOT just summarize it.** The user should see what they're approving.

1. Read `.claude-task/{taskId}/spec.md`
2. **Present the full spec in the conversation**, section by section:
   - Implementation approach and rationale
   - Files to create/modify (with purpose of each)
   - Key technical decisions
   - Acceptance criteria mapping
   - Testing strategy
3. After presenting, ask:
   > "This is the implementation plan. Review the approach, files, and decisions."
   >
   > **Proceed** / **Discuss changes** / **Revise specific sections**"

**Do NOT** say "review spec.md" and point to a file. The spec content must be visible in the conversation where the user is reading.

Set `planSource.userAcknowledged = true` and `workflowState.standardPlanning.executionAuthorized = true`.

### Phase A.4: Refine Task List (MANDATORY — after user approves plan)

Now that the plan is concrete, replace the placeholder tasks from init Step 2b with specific tasks based on the actual spec, route, and scope. Use `TaskUpdate` to update existing placeholders and `TaskCreate` for new tasks.

**For Standard route:**
- Update "Implementation" → "Implementation — {N} files per spec"
- Add: "Build check — typecheck + lint + tests"
- Add: "Update documentation — README, API docs, migration notes (if applicable)"

**For Team route:**
- Update "Implementation" → "Dispatch {N} workers — {section summaries}"
- Add one task per worker section: "Worker {N}: {section title} — {file count} files"
- Add: "Merge workers + build gate"
- Add: "Update documentation"

**For Blueprint route:**
- Update "Implementation" → "Architecture review + {N} worktree workers"
- Add: "Present architecture to user"
- Add one task per worker: "Worker {N}: {section title} (worktree)"
- Add: "Merge worktrees + build gate"
- Add: "Update documentation"

**Refine QA and Validation placeholders:**
- Update "QA" → "QA — {method} ({product type})" (e.g., "QA — browser walkthrough (web app)")
- Update "Validation" → "Validation — {profile} profile ({N} gates)"

**Documentation task detail** — based on what the spec modifies:
- API endpoints changed? → "Update API documentation"
- New feature? → "Update README with feature description"
- Config/env changes? → "Update setup instructions"
- DB schema changes? → "Update migration notes"
- CHANGELOG exists? → "Add CHANGELOG entry"
- If nothing needs docs: skip the docs task

### Phase B: Implementation

**⚠️ Set `manifest.implementationDelegated = true`** immediately for Standard route (orchestrator may implement inline for lean, or delegates to a single agent). The implementation gate hook allows orchestrator edits when this flag is set.

1. **Lean profile (simple tasks)**: Implement directly inline.
   - Compaction guard monitors context growth
   - If compaction guard triggers: delegate to implementation agent

2. **Standard+ profile**: Spawn implementation agent.
   **Before spawning**: Run Memplex Context Assembly for the spec's primary files.
   > Spawn implementation-agent (sonnet) from ~/.claude/agents/core/implementation-agent.md
     Context: "Read your spec at .claude-task/{taskId}/spec.md. Implement the plan." + Known Context block (if memplex available)
     max_turns: 25
     Writes: source code changes, deferred items
     Returns: "STATUS: completed|blocked. FILES_MODIFIED: [...]. BUILD: pass|fail."

3. **Implementation coherence check** (fast, before build gate):
   Verify the agent implemented what the spec asked for — catches design drift before it compounds.
   > Spawn closure-agent (haiku) from ~/.claude/agents/core/closure-agent.md
     Context: spec.md (relevant section), manifest.modifiedFiles, brief.md acceptance criteria
     Scope: "Quick coherence check — does the implementation match the spec? Not a full closure review."
     Returns: "COHERENT" or "DRIFT: {what's missing or wrong}"

   - If **COHERENT**: proceed to build gate
   - If **DRIFT**: feed drift description back to the implementation agent for ONE revision round, then re-check. Max 1 revision — if still drifting, log to `manifest.degradations` and proceed (full closure at validation will catch it)
   - **Budget**: one haiku call (~2-3 seconds). Not a bottleneck.

4. **Build gate**: Run typecheck + lint + tests. Build-fix rounds per policy `limits.buildFixRounds`.

5. **Convention scan** (lean only): Read conventions.json or CONVENTIONS.md. Grep modified files for violations. Fix inline. Budget: 3-6 Grep calls.

6. **Update documentation** (if docs task exists in task list):
   Based on `manifest.modifiedFiles`, update relevant documentation:
   - API routes modified → update API docs (README, OpenAPI spec, or dedicated docs)
   - New feature added → add to README features section
   - Config/env changed → update setup/installation instructions
   - DB schema changed → update migration notes
   - CHANGELOG exists → add entry for this change
   - Mark the docs task as `completed` when done. If no docs needed, mark as `completed` with note "no docs changes required".

**Git notes**: No incremental commits. One commit at completion.

6. **Proceed to QA**: Read `~/.claude/taskplex/phases/qa.md`

---

## Team Route

### Phase A: Planning Agent with Section Assignment

Spawn the planning agent in Team mode. The planning agent identifies independent sections and writes file ownership.

> Spawn planning-agent (sonnet) from ~/.claude/agents/core/planning-agent.md
  Context: brief.md path, taskId, designDepth, qualityProfile, mode: "team"
  Writes: .claude-task/{taskId}/spec.md, .claude-task/{taskId}/sections.json, .claude-task/{taskId}/file-ownership.json, .claude-task/{taskId}/conventions-snapshot.json
  Returns: "PLANNING COMPLETE. Spec: {path}. Sections: {N}. Files affected: {N}."

Set `manifest.planFile = "spec.md"`.

### Phase A.1: Research (conditional)

Same as Standard route — see above.

### Phase A.2: Spec Critic (mandatory)

> Spawn closure-agent (haiku) from ~/.claude/agents/core/closure-agent.md
  Context: spec.md, sections.json, brief.md, CONVENTIONS.md — review spec + section assignments against brief requirements
  Returns: "Verdict: APPROVED|NEEDS_REVISION"

Track in `manifest.iterationCounts.reviewRounds.specCritic`.

Same revision loop as Standard route (max 2 rounds).

### Phase A.3: Pre-Implementation Acknowledgment (MANDATORY)

Same as Standard route.

### Phase B: Multi-Agent Implementation

**⚠️ HARD RULES (Team implementation)**:
1. The orchestrator MUST NOT edit source code directly — delegate to agents
2. Independent agents MUST be dispatched in a **single message** (parallel Agent tool calls)
3. Do NOT pause between agent dispatches — see Execution Continuity Rule above
4. Run straight through: dispatch all agents → merge → build check → QA → validation
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

4. **Build gate** (after all agents return + coherence verified): Run typecheck + lint + tests. If failures, spawn build-fixer (max rounds per policy). This catches integration issues between workers before moving to QA.

5. **Update documentation** (same as Standard route step 6 — update docs based on modified files).

6. **Proceed to QA → Full Validation**: Read `~/.claude/taskplex/phases/qa.md`, then `~/.claude/taskplex/phases/validation.md`. Full validation runs once after all workers complete and build gate passes.

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

3. **Present architecture to user — do NOT just ask "ready to proceed?"**

   After the architect returns:
   a. Read `.claude-task/{taskId}/architecture.md`
   b. Read `.claude-task/{taskId}/spec.md`
   c. **Present the architecture in the conversation**, section by section:
      - Architecture overview and component breakdown
      - Key design decisions and trade-offs
      - File ownership map (which worker handles what)
      - Implementation approach per section
   d. After presenting, ask:
      > "This is the architecture and implementation plan. Review the approach and component breakdown."
      >
      > **Proceed** / **Revisit architecture** / **Modify specific sections** / **Cancel**"

   **Do NOT** say "review architecture.md" and point to a file. Present the content inline.

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

**Activated when**: `manifest.initiativeMode === true` (set by `--prd` flag or multi-feature detection).

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

**Interactive mode** (Recommended):
- Single confirmation per feature before implementation starts
- User sees progress updates between features

**Autonomous mode**:
- Critic APPROVED = auto-proceed (no user confirmation)
- Drift from spec is logged but not prompted

**→ After user confirms, set `manifest.designInteraction.prdExecutionModeChosen = true` and `manifest.designPhase = "planning-active"`**

### Initiative Phase 2d: Branch Strategy

If `manifest.git.available === true` and `manifest.git.config.createBranch === true`:
- Each wave gets its own branch: `feat/prd-{id}-wave{N}`
- Wave 0 branch created from `manifest.git.baseBranch`
- Subsequent wave branches created from the merged result of the previous wave

### Initiative Phase 3: Wave Execution

**⚠️ Execution Continuity applies here.** Once wave execution begins, run all waves to completion. Do NOT ask the user between features or between waves unless a blocking failure occurs. The user already approved the plan (Phase 2b) and chose execution mode (Phase 2c).

**Interactive mode** means: one confirmation per WAVE (not per feature). After the user confirms a wave, all features in that wave execute non-stop.

**Autonomous mode** means: no confirmations at all — waves execute back-to-back.

For each wave (0, 1, 2, ...):

1. Update `prd-state.json`: set wave status to `in-progress`
2. **Create wave branch** (if git available + createBranch)
3. **If Interactive mode AND wave > 0**: Single confirmation: "Wave {N} ready ({features}). Proceed?" — then run non-stop
4. **If Autonomous mode**: proceed directly, no confirmation

5. **Wave 0** (sequential — foundational features):
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

6. **Wave 1+** (parallel — independent features):
   - Dispatch ALL features in the wave in a **single message** using parallel Agent tool calls
   - Every agent MUST use `isolation: "worktree"`
   - After all return: merge, update state, proceed to next wave immediately

### Initiative Phase 4: Wave Merge + Wave Validation

After each wave completes:
1. List completed feature branches
2. Apply merge strategy (from `manifest.git.config.mergeStrategy`, default `squash`)
3. If conflicts:
   > Spawn merge-resolver (sonnet) from ~/.claude/agents/core/merge-resolver.md
     Context: source branch, target branch, conflict files, feature descriptions
     Returns: "Merge resolved: {N} files. Typecheck: PASS/FAIL"
   - If unresolvable: mark as `blocked:merge-conflict`, notify user

4. **Wave validation (build gate)** — catches broken foundations before next wave builds on them:
   - Run typecheck (from `manifest.buildCommands.typecheck`)
   - Run lint (from `manifest.buildCommands.lint`)
   - Run tests (from `manifest.buildCommands.test`) — **critical**: if Wave 0 breaks tests, Wave 1 must not proceed
   - If any fail: spawn build-fixer (max rounds per policy). If still failing after limit, mark wave as `blocked:build-failure`, stop wave execution, present to user.

5. Update prd-state.json: wave status = `completed`

**Wave validation is a build gate, not full validation.** It checks: does the merged code compile, pass lint, and pass tests? It does NOT run security review, closure, code review, hardening, or compliance — those run once at the end.

### Initiative Phase 5: Finalization

1. **Cross-Feature Intent Check**:
   > Spawn closure-agent (haiku) from ~/.claude/agents/core/closure-agent.md
     Context: brief.md, aggregated modifiedFiles from all features, prd.md
     Writes: .claude-task/PRD-{id}/reviews/cross-feature-closure.md
     Returns: "PASS" | "FAIL: {what's missing}"

   If FAIL: report to user before full validation.

2. **Full Validation Pipeline** — runs ONCE after all waves complete. This is where security review, closure, code review, hardening, and compliance run. See `~/.claude/taskplex/phases/validation.md`.
3. Generate completion report
4. If git available: create comprehensive PR covering the entire initiative
5. Present to user
6. Clean up: archive prd-state.json

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
