# TaskPlex Feature Test Plan

Run these tests in a **fresh session** on a real project (not the taskplex repo itself). Each test verifies a specific feature built in the 2026-03-31 session.

## Setup

Choose a project with:
- An existing codebase (not empty — needs files for convention scan)
- A `package.json` or similar (for project type detection)
- Ideally a `README.md` and `CLAUDE.md` (for context density testing)
- Git initialized

## Test 1: Fresh /tp Invocation + Task List

**Command**: `/tp --blueprint add user authentication with JWT`

**Verify**:
- [ ] `tp-prompt-check` hook fires — workflow reminder injected
- [ ] Step 0: Route parsed as Blueprint from `--blueprint` flag
- [ ] Step 1: `.claude-task/{taskId}/manifest.json` created
- [ ] Step 2: Session file created
- [ ] **Step 2b: Task list appears** — 9 high-level tasks created via TaskCreate
  - First task "Initialize task" marked `in_progress`
  - Tasks 6-8 are placeholders ("Implementation", "QA", "Validation")
- [ ] Step 3: Loads INTENT.md, CONVENTIONS.md, CLAUDE.md (if they exist)
- [ ] **Step 3a: Memplex check** — manifest.memplexAvailable set to true or false
  - If memplex available: initial search_knowledge call runs
  - If not: no error, continues normally

**Pass criteria**: Task list visible in conversation UI with 9 tasks.

## Test 2: Adaptive Interaction (Sub-phase B)

**Verify during the design phase**:
- [ ] **Context gathering happens BEFORE first question** — reads docs + scans codebase + checks memplex
- [ ] **Context density assessed** — check manifest.designInteraction.contextDensity
- [ ] **First question adapts to density**:
  - High density: confirmation question ("From {sources}, here's my understanding... Is that right?")
  - Low density: open question ("What's the goal here?")
- [ ] **Questions don't repeat what docs already say** — if README explains the purpose, the orchestrator doesn't ask "what is this project?"
- [ ] **contextSources recorded** in manifest.designInteraction
- [ ] If memplex available: past decisions surfaced in context gathering

**Pass criteria**: First question arrives within 30 seconds, informed by gathered context.

## Test 3: Design Gate Enforcement

**Test 3a — Flag-based gate (not counter-based)**:
- [ ] Try to answer only 1 question in full mode
- [ ] Try to jump to brief.md writing
- [ ] **Expected**: Hook blocks with message about missing flags (contextConfirmed, ambiguitiesResolved, approachSelected, sectionsApproved)
- [ ] Message should NOT mention question counts

**Test 3b — Confirm flags unlock the gate**:
- [ ] Complete the adaptive interaction naturally
- [ ] Verify manifest.designInteraction has: contextConfirmed=true, ambiguitiesResolved=true
- [ ] After approach selection: approachSelected=true
- [ ] After section approval: sectionsApproved >= 1
- [ ] brief.md write should succeed

**Pass criteria**: Gate blocks based on flags, not counters.

## Test 4: Task List Refinement (Phase A.4)

**After the user approves the plan (Phase A.3)**:
- [ ] Placeholder tasks get replaced with specific tasks
- [ ] For Blueprint: should see per-worker tasks ("Worker 1: {section}", "Worker 2: {section}")
- [ ] "Merge + build gate" task appears
- [ ] "Update documentation" task appears (if applicable)
- [ ] QA task refined: "QA — {method} ({product type})"
- [ ] Validation task refined: "Validation — {profile} ({N} gates)"

**Pass criteria**: Task list evolves from generic to specific after plan approval.

## Test 5: Implementation Gate (Blueprint)

**After plan approval, during implementation phase**:
- [ ] Orchestrator attempts to edit a source file directly (e.g., src/auth.ts)
- [ ] **Expected**: Hook blocks with "Implementation gate: Source file edits blocked. Execution mode: blueprint — delegate to agents"
- [ ] Message instructs to spawn agents with `isolation: "worktree"`
- [ ] After agents dispatched and `manifest.implementationDelegated = true` set: source edits allowed

**Pass criteria**: Orchestrator cannot code inline in blueprint mode.

## Test 6: Execution Continuity

**During implementation (after plan approved)**:
- [ ] Agents are dispatched without asking "should I continue?"
- [ ] Independent agents dispatched in a **single message** (parallel Agent tool calls)
- [ ] No pause between agent returns
- [ ] Flow: dispatch → agents return → coherence check → build gate → docs → QA — uninterrupted

**Pass criteria**: No "should I continue?" prompts during implementation.

## Test 7: Implementation Coherence Check

**After implementation agent(s) return, before build gate**:
- [ ] Closure-agent (haiku) spawned for quick spec-vs-code check
- [ ] Returns "COHERENT" or "DRIFT: {description}"
- [ ] If DRIFT: one revision round for the agent
- [ ] If COHERENT: proceeds to build gate

**Pass criteria**: Haiku coherence check runs between agent return and build gate.

## Test 8: Documentation Update Step

**After build gate passes, before QA**:
- [ ] Orchestrator checks manifest.modifiedFiles against docs that need updating
- [ ] If API routes modified: API docs updated
- [ ] If new feature: README updated
- [ ] If no docs needed: docs task marked "completed — no docs changes required"
- [ ] Docs task in task list updated accordingly

**Pass criteria**: Documentation step runs and task list reflects outcome.

## Test 9: Memplex Integration

**Only testable if memplex is available** (check manifest.memplexAvailable):

- [ ] **Pre-spawn context assembly**: Before agent spawn, orchestrator calls file_intelligence + get_error_resolution + search_knowledge
- [ ] **Known Context block**: Agent prompt includes "## Known Context (from project memory)" section
- [ ] **Convention check**: search_knowledge("conventions") called during Sub-phase A
- [ ] **Intent exploration**: search_knowledge used as 4th context source
- [ ] **Completion**: write_knowledge called to save file couplings, error resolutions, patterns
- [ ] **Task summary**: Shows "Knowledge saved: {N} couplings, {M} resolutions..."

**If memplex NOT available**:
- [ ] No errors, no degradation logged
- [ ] Known Context block omitted from agent prompts
- [ ] Task summary: "Patterns discovered: {N}. Cross-session persistence requires memplex."

**Pass criteria**: Memplex used when available, gracefully skipped when not.

## Test 10: User Visibility

**Pre-spawn messages**:
- [ ] Before planning agent: "Spawning the planning agent... This typically takes 3-8 minutes."
- [ ] Before architect (blueprint): "Spawning the architect agent (opus). This typically takes 5-15 minutes."
- [ ] Before researcher: "Spawning researcher to investigate {topics}..."

**Inline presentation**:
- [ ] After planning agent returns: spec.md content presented **in the conversation**, not "review spec.md"
- [ ] After architect returns (blueprint): architecture.md + spec.md presented inline, section by section
- [ ] After researcher returns: key findings presented inline
- [ ] Pre-Implementation Acknowledgment: full spec visible, not "3-5 bullet summary"

**Pass criteria**: User never needs to open a file to review agent output.

## Test 11: Mid-Task Change

**During any phase, say**: "also add rate limiting to the auth endpoints"

- [ ] Orchestrator handles naturally — doesn't derail workflow
- [ ] Task list updated (TaskCreate for new work or TaskUpdate for scope change)
- [ ] Brief/spec updated if applicable
- [ ] Manifest reflects the change

**Also test a side question**: "how does the existing middleware work?"
- [ ] Answered inline
- [ ] Checklist NOT advanced
- [ ] Manifest NOT updated
- [ ] Workflow resumes from where it was

**Pass criteria**: Scope changes update state. Side questions don't.

## Test 12: Session Resume

**Setup**: Run `/tp` and get to implementation phase (past design, into execution). Then **kill the session** (Ctrl+C or close terminal).

**In a new session on the same project**:
- [ ] `tp-session-start` hook fires — detects active task
- [ ] Phase checklist rendered from `manifest.phaseChecklist`
- [ ] "MANDATORY: Recreate the task list using TaskCreate for remaining steps" instruction present
- [ ] Task list recreated — pending/active steps become tasks
- [ ] Active step marked `in_progress`
- [ ] Workflow resumes from the correct step, not from scratch

**Pass criteria**: Full state recovery after session kill.

## Test 13: Skill Evolution (requires full completion)

**Run a task through to completion** with at least one of:
- Build-fix rounds >= 2
- QA bugs found > 2
- Review resubmission

**After validation passes, at completion**:
- [ ] Signal detection runs (Step 11.5 in validation.md)
- [ ] If signals exceed threshold: evolution entry generated (LLM call)
- [ ] `evolutions.json` created/updated at `~/.claude/skills/taskplex/evolutions.json`
- [ ] Task summary mentions: "Skill evolution: {type} — {title} (run /solidify to review)"

**If no signals exceed threshold**:
- [ ] Step 11.5 skipped — this is expected for clean tasks

**Pass criteria**: Evolution generated when signals detected, skipped when clean.

## Test 14: Goal Traceability

**Requires INTENT.md in the project root.**

**During brief writing (Step 7b)**:
- [ ] Each AC in brief.md mapped to INTENT.md success criteria
- [ ] `manifest.goalTraceability` populated with mappings
- [ ] Coverage stats: mapped vs unmapped vs total

**During validation (closure agent)**:
- [ ] Goal traceability section in closure report
- [ ] Unmapped ACs noted but not flagged as failures
- [ ] If zero ACs map: warning about disconnected task

**If no INTENT.md**: goalTraceability is null, steps skipped — not a failure.

**Pass criteria**: ACs trace to intent when INTENT.md exists.

## Test 15: Frontend Skill (if UI task)

**Run on a task that modifies UI files** (`.tsx`, `.jsx`, `.css`):

- [ ] Frontend skill triggers automatically
- [ ] Design system detection runs (references/design-system.md)
- [ ] CSS approach detected (references/css-patterns.md)
- [ ] Component spec template used for non-trivial components
- [ ] Accessibility patterns applied (roles, keyboard nav, aria)
- [ ] If agent-browser available: visual review with screenshots

**Pass criteria**: UI-specific guidance applied automatically.

---

## Scoring

| Result | Meaning |
|--------|---------|
| All checks pass | System working as designed |
| 1-3 failures | Minor gaps — likely prompt compliance issues, fix with instruction tweaks |
| 4-7 failures | Significant gaps — some features not being followed, need investigation |
| 8+ failures | Systemic issue — workflow not being read or hooks not firing |

## Quick Smoke Test (if short on time)

Run just these 5 tests for a fast validation:
1. **Test 1**: Task list appears on fresh /tp
2. **Test 3a**: Design gate blocks premature brief.md
3. **Test 5**: Implementation gate blocks inline coding in blueprint
4. **Test 10**: Spec presented inline, not "review file X"
5. **Test 12**: Session resume recovers state
