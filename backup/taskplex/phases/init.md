# taskplex: Initialization (Phase 0)
<!-- Loaded by orchestrator at task start. Self-contained. -->
<!-- v2: Enhanced design phase with Convention Check (Sub-phase A) + Intent/User Journey (Sub-phase B) -->

---

## Step 0: Task Description & Workflow Selection

### 0a: Task Description

If the user provided a task description (e.g., `/tp add user auth`), use it.

**If NO task description was provided** (user typed just `/tp` or `/tp` with only flags):
> `AskUserQuestion`: "What would you like to work on?"

Use the answer as the task description. Do NOT proceed without a description.

### 0b: Route Selection

Parse any explicit flags from the invocation:

| Flag | Route | Design Depth | Notes |
|------|-------|-------------|-------|
| `--standard` or `--full` | Standard | full | Full design, single agent *(default)* |
| `--team` or `--parallel` | Team | full | Full design, multi-agent execution |
| `--blueprint` or `--architect` or `--deep` | Blueprint | full | Opus architect + critics + multi-agent + worktrees |
| `--prd` | Blueprint | full | Blueprint + initiative mode (decomposition + waves) |
| `--light` | Standard | light | Reduced design depth (fewer questions) |
| `--skip-design` | Standard | skip | No design phase, logs degradation |
| `--plan PLAN-{id}` | (any) | (any) | Use existing plan file |

**If explicit flags were provided**: use them directly. Skip the menu.

**If NO route flags were provided** (user typed `/tp <description>` with no flags):
> `AskUserQuestion`:
> "How would you like to approach this?
>
> 1. **Standard** — full design, single agent *(default)*
> 2. **Team** — full design, multi-agent execution
> 3. **Blueprint** — opus architect + critics + multi-agent + worktrees
>
> Pick a number, or press Enter for Standard."

Map the user's choice:
- 1 or Enter/default → Standard
- 2 → Team
- 3 → Blueprint

Set `manifest.designDepth` to `"light"` or `"full"`.
Set `manifest.executionMode` to `"standard"`, `"team"`, or `"blueprint"`.
If `--prd` was used: also set `manifest.initiativeMode = true`.

**Backward compatibility**: If resuming a task with legacy `executionMode` values, map them: `single→standard`, `parallel→team`, `architect→blueprint`, `prd→blueprint` (with `initiativeMode: true`).

**Mid-flow escape:** If the user says "just do it", "skip design", or similar during Sub-phase B, switch from full to light immediately and write a minimal brief.

---

## Step 1: Create Task Directory & Manifest

```
.claude-task/{taskId}/
  manifest.json    # Task metadata, phase tracking, modified files
  spec.md          # Implementation plan
  qa-report.md     # QA findings (written in Phase 4.5)
  reviews/         # Review results
  checkpoints/     # PreCompact snapshots
  workers/         # Worker status files (for parallel execution)
  deferred/        # Deferred items found during implementation
```

`taskId` = sanitized task description or timestamp.

**manifest.json** — Create immediately using the canonical schema at `~/.claude/taskplex/manifest-schema.json`. Initialize with:
```json
{
  "schemaVersion": 2,
  "taskId": "{taskId}",
  "description": "{task description}",
  "phase": "init",
  "status": "in-progress",
  "designDepth": "light|full",
  "executionMode": "standard|team|blueprint",
  "initiativeMode": false,
  "designPhase": "convention-scan",
  "createdAt": "ISO timestamp",
  "lastUpdated": "ISO timestamp",
  "workflow": "taskplex"
}
```

All other fields use schema defaults (null, empty arrays/objects, false). See the canonical schema for the complete field list, types, and defaults.

**Key field behaviors** (not captured by schema alone):
- `progressNotes`: The heartbeat hook renders `progress.md` from this array. **The orchestrator MUST NOT write progress.md directly** — always append to this array.
- `planSource.userAcknowledged`: MUST be `true` before implementation begins (set by Step 5 validation gate).
- `workflowState.standardPlanning.executionAuthorized`: MUST be `true` before implementation begins.
- **⚠️ `planningChecklist` is VESTIGIAL (v1).** Do NOT create it, do NOT check it, do NOT block on `implementationPlanReady`. The authoritative execution gates are `planSource.userAcknowledged` + `workflowState.*.executionAuthorized` only.
- `degradations`: Array in manifest.json. Never silent — every quality deviation is logged here.
- `buildCommands`: Resolved in Step 4b from conventions.json overrides + detected defaults.
- `customGates`: Validated in Step 4b.
- `extensions`: Validated in Step 4c.

## Step 2: Create Session File (MANDATORY — do not skip)

**This step is required for viz app detection, heartbeat hook updates, and session recovery. The heartbeat hook will error on every Edit/Write if this file doesn't exist. Create it immediately after manifest.json.**

Create `~/.claude/sessions/sess-{pid}.json` for the 3D visualizer:
```json
{
  "sessionId": "sess-{pid}",
  "taskId": "{taskId}",
  "cwd": "{project root}",
  "pid": "{process pid}",
  "phase": "init",
  "designDepth": "light|full",
  "executionMode": "standard|team|blueprint",
  "initiativeMode": false,
  "status": "in-progress",
  "workflow": "taskplex",
  "createdAt": "ISO timestamp",
  "lastUpdated": "ISO timestamp",
  "agents": [],
  "events": [
    {
      "traceId": "task-{taskId}",
      "spanId": "task-start-{timestamp}",
      "operation": "task-start",
      "phase": "init",
      "startTime": "ISO timestamp",
      "status": "ok",
      "attributes": { "description": "{task description}", "designDepth": "light|full", "executionMode": "..." }
    }
  ],
  "fanOut": null
}
```

After creating the session file, update `manifest.sessionFile` to the actual path. Do NOT leave it as a placeholder.

## Step 2b: Create Task List (MANDATORY — enables progress tracking in conversation)

**Immediately after creating the manifest and session file**, create the task list using `TaskCreate` for the high-level workflow phases. These are the initial shape — they get refined after the plan is written (see Phase A.3 in planning.md).

**For full design depth**, create these tasks:
1. "Initialize task — parse flags, create manifest, load context" (mark `in_progress`)
2. "Convention check — scan codebase, confirm patterns with user"
3. "Intent exploration — synthesize context, confirm with user, resolve gaps"
4. "Write brief.md — approaches, section approval, write approved design"
5. "Planning — write spec, critic review, user acknowledges plan"
6. "Implementation" (placeholder — refined after plan is approved)
7. "QA" (placeholder — refined after implementation scope is known)
8. "Validation" (placeholder — refined after implementation)
9. "Completion — git commit, PR, task summary"

**For light design depth**, create these tasks:
1. "Initialize task" (mark `in_progress`)
2. "Design (light) — synthesize, confirm, write brief"
3. "Planning — write spec, critic review, user acknowledges"
4. "Implementation" (placeholder — refined after plan)
5. "QA" (placeholder)
6. "Validation" (placeholder)
7. "Completion — git commit, PR, task summary"

Mark each task `in_progress` when you start it, `completed` when done. This is what the user sees in the conversation — it must stay current.

**Task list refinement happens at Phase A.3** (Pre-Implementation Acknowledgment in planning.md). After the user approves the plan, replace placeholder tasks with specific ones based on the actual spec, route, and scope.

**On session resume**: The `tp-session-start` hook will instruct you to recreate tasks from `manifest.phaseChecklist`. Follow that instruction before doing any other work.

## Step 3: Load Context & Check for Resume

1. Read `INTENT.md`
2. Read `CONVENTIONS.md`
3. Read `conventions.json` (if exists — machine-readable conventions)
4. Read `CLAUDE.md`
5. Read `.schema/_index.md` (if exists) for schema overview
6. Look for `.claude-task/{any}/manifest.json` with `status: 'in-progress'`
   - Also check for `checkpoints/compact-*.json` (from PreCompact hook)
   - If found: `AskUserQuestion` "Found in-progress task: {title}. Resume or start fresh?"
   - If resuming: the SessionStart hook has already injected recovery context

## Step 3a: Detect Memplex Availability

Check if memplex MCP tools are available. This determines whether cross-session knowledge (error resolutions, file coupling, past decisions) can be used throughout the workflow.

```
Check: Is mcp__mp__search_knowledge available?
  → If yes: set manifest.memplexAvailable = true
  → If no:  set manifest.memplexAvailable = false
```

**If available**, run an initial knowledge load:
1. `search_knowledge` scoped to the task description — surfaces past decisions, patterns, and gotchas about this area
2. Store results in `manifest.memplexContext.init` for use in subsequent phases

**If not available**: continue. No error, no warning, no degradation. Memplex is an optional enhancement.

**First-task notice** (once per project): If `memplexAvailable === false` and no prior `.claude-task/` directories exist (first task in project):
> "TaskPlex will use codebase scanning for context gathering. For cross-session knowledge persistence (error resolutions, file coupling, past decisions), memplex is available as an optional enhancement."

Set `manifest.memplexNoticeShown = true` to avoid repeating.

## Step 3b: PRD/Plan Detection

**Only triggered by explicit user intent.** Do NOT scan for ambient PRD files in the project.

**Detection triggers** — ONLY these:
1. `--prd` flag → set `manifest.executionMode = "blueprint"` + `manifest.initiativeMode = true`, then read `~/.claude/taskplex/phases/planning.md` (Blueprint: Initiative Mode section)
2. `--plan PLAN-{id}` argument
3. User explicitly references a PRD/plan file in their task description (e.g., `/tp implement PRD-auth.md`)

**NOT triggers** (do NOT auto-detect these):
- PRD-*.md files sitting in the project root
- Plan files from `~/.claude/plans/` not referenced by the user
- Ambient files with structured implementation details
- Inline content that happens to look structured

**If NO plan/PRD detected**: continue to Step 4.

**If a plan/PRD is detected**:

A PRD is **input context**, not a bypass. The workflow still runs — the PRD gives the agent a head start, but the user still reviews and approves before implementation.

1. **Read the referenced document**. Record source in manifest:
   ```json
   "planSource": {
     "origin": "/plan" | "plan-mode" | "external-file" | "user-prompt",
     "filePath": "{path or null}",
     "lifecycleValidated": true | false,
     "userAcknowledged": false
   }
   ```

2. **Summarize the PRD** to the user (key goals, scope, feature count). Then ask:
   > `AskUserQuestion`: "I've read the PRD. Before implementing, I'd like to review it with you.
   >
   > 1. **Full review** — walk through each section, discuss trade-offs, refine scope
   > 2. **Light review** — quick summary of my understanding, confirm scope and priorities, then proceed
   >
   > Which approach?"

3. **Run the chosen workflow**. The PRD content feeds into Sub-phase A (convention check) and Sub-phase B (intent exploration) as pre-loaded context:
   - **Light review**: Summarize understanding from PRD, ask 2-3 targeted questions about priorities/scope/concerns, write brief from PRD + answers
   - **Full review**: Walk through PRD section by section, discuss trade-offs, surface gaps, confirm scope boundaries, then write brief

4. The normal design flow continues from there — the PRD accelerates it but doesn't skip it.

---

## Step 4: Detect Project & Load Conventions

### Step 4a: Detect Project Type (defaults)
- `package.json` → `npm run lint`, `npm run typecheck`, `npm run test`
- `pyproject.toml` → `ruff check .`, `mypy .`, `pytest`
- `Cargo.toml` → `cargo check`, `cargo test`
- `go.mod` → `go build ./...`, `go test ./...`

### Step 4b: Load Convention Overrides

If `conventions.json` exists in the project root:

1. **Read and validate** against `~/.claude/schemas/convention-schema-v1.json`
2. **Override build commands**: For each `build.*` field present, override the detected default
3. **Register custom gates**: max 5 gates, timeout ≤ 60s, name is kebab-case. Run AFTER standard gates, BEFORE compliance.
4. **Register required checks**: Added as additional build check commands
5. **Write resolved commands to manifest.json**

If no `conventions.json`: use detected defaults, no custom gates.

### Step 4c: Load Extensions

If `conventions.json` has an `extensions` section:
1. Validate each agent/hook/skill definition
2. Enforce limits: max 5 agents, max 5 hooks, max 3 skills
3. Write validated extensions to `manifest.extensions`

Extension hooks are invoked by the orchestrator, NOT registered in `settings.json`.

### Step 4d: Conflict Detection

Run conflict resolver logic:
1. **Invariant floor check**: Verify conventions don't disable security review, closure, compliance, manifest tracking, gate decisions, degradation logging, or build checks. BLOCK if violated.
2. **Duplicate gate check**: Remove custom gates matching standard gate names. Log to degradations.
3. **Missing capability check**: Verify build commands exist. FLAG if missing.
4. **Extension conflict check**: Compare custom agents against built-in agents. FLAG overlaps.
5. **Naming conflict check**: Spot-check 5 files if naming conventions declared. FLAG if >30% mismatch.
6. Present flagged conflicts via `AskUserQuestion` (batch all).

### Step 4e: Load Git Configuration

1. **Detect git availability**: Check `.git` and `git --version`. If unavailable: set `manifest.git.config = { available: false }`, skip git integration.
2. **If git available**: Read `conventions.json` → `git` section. Apply backward-compat defaults. Auto-detect `defaultBranch`. Snapshot to `manifest.git.config`.
3. **No remote**: Push/PR skipped at completion. Log to progressNotes.
4. **No `gh` CLI**: PR creation skipped at completion with message.

---

## Sub-phase A: Convention Check

**Purpose:** Understand the codebase conventions that apply to this task. This is analytical — about the code, not the feature.

### ⚠️ Design Phase Gating (HARD RULE)

The design flow is divided into **sub-phases** tracked by `manifest.designPhase`. You MUST advance through them in order. The `tp-design-gate` hook will **block file writes** if you try to write artifacts before reaching the required sub-phase.

**You MUST update `manifest.designPhase` explicitly** after completing each step. Do NOT skip sub-phases. Do NOT batch multiple sub-phases into one step.

Sub-phase progression for **full** design depth:
```
convention-scan → convention-check → intent-exploration → approach-review → design-approval → brief-writing
```

Sub-phase progression for **light** design depth:
```
convention-scan → intent-exploration → brief-writing
```

PRD mode appends:
```
→ prd-bootstrap → prd-critic → prd-approval → planning-active
```

After each sub-phase completes, update the manifest:
```json
{ "designPhase": "<next-sub-phase>" }
```

### Light Mode (`--light`)

**designPhase starts at: `convention-scan`**

Automated scan only, no user questions:

1. **Read existing conventions**: `CONVENTIONS.md`, `conventions.json`
2. **Memplex convention lookup** (if `manifest.memplexAvailable`):
   - `search_knowledge("conventions {target area}")` — conventions confirmed in past tasks
   - `file_intelligence` on target area files — known issues, coupled files, change patterns
   - Store results in `manifest.memplexContext.conventionCheck`
   - If memplex not available: skip, rely on codebase scan only
3. **Scan target area**: Grep/Glob the area being modified for patterns (naming, structure, imports, error handling). 5-8 calls max.
4. **Infer conventions**: Note patterns found (e.g., "components use PascalCase", "errors use Result type", "tests co-located")
5. **Record**: Write inferences to `manifest.conventionContext` (task-scoped). If new conventions discovered that aren't in CONVENTIONS.md, note them for later (don't prompt user).

**→ Set `manifest.designPhase = "intent-exploration"` and continue to Sub-phase B.**

### Full Mode (`--full`, default)

**designPhase starts at: `convention-scan`**

Same automated scan as light (including memplex convention lookup if available), PLUS targeted questions:

1. **Run automated scan** (same as light steps 1-4, including memplex lookup). Record inferences to `manifest.conventionContext`.

**→ Set `manifest.designPhase = "convention-check"`. Initialize `manifest.designInteraction = {}` if not present.**

2. **HARD RULE — STOP AND ASK THE USER.** Generate 2-4 targeted questions about the specific area being modified. Questions should:
   - Confirm or correct automated inferences
   - Surface unwritten conventions specific to this area
   - Use multiple-choice format when possible
   - Focus on the area being changed, not general project setup

   Example questions:
   - "I see components in `src/ui/` use React.FC with Props types. Should new components follow the same pattern, or is there a preferred alternative?"
   - "Error handling in this module uses try/catch with custom error classes. Is that the pattern for new code here, or should I use Result types?"

3. **Ask questions one at a time** via `AskUserQuestion`. Wait for the user's response before asking the next question. Do NOT batch all questions into a single message.
   - **After each question asked**: Increment `manifest.designInteraction.conventionQuestionsAsked`
   - **After each user answer**: Increment `manifest.designInteraction.conventionQuestionsAnswered`
   - The `tp-design-gate` hook will block CONVENTIONS.md writes until at least 2 questions have been asked.
4. **Record confirmed conventions**: Update `manifest.conventionContext.confirmed` with user-confirmed patterns. If user reveals conventions not in CONVENTIONS.md, append them to CONVENTIONS.md (accumulates across tasks).

**→ Set `manifest.designPhase = "intent-exploration"` and continue to Sub-phase B.**

### Convention Refresh (both modes)

Check `CONVENTIONS.md` metadata comments:
- `<!-- project-phase: greenfield | established | mature -->`
- `<!-- last-refreshed: {date} | tasks-since-refresh: {N} -->`

**Greenfield**: Skip refresh entirely.
**Established**: Refresh threshold = 10 tasks.
**Mature**: Refresh threshold = 5 tasks.

- **Light mode**: If threshold met, increment counter silently, note in manifest. Don't prompt.
- **Full mode**: If threshold met, ask user: "It's been {N} tasks since conventions were reviewed. Refresh now?" Options: **Refresh** / **Skip** / **Never ask**. If Refresh: scan codebase for convention drift, update CONVENTIONS.md.

---

## Sub-phase B: Intent / User Journey Exploration
<!-- v2: Adaptive interaction model — question count driven by context density, not hard minimums -->

**Purpose:** Understand what to build and why. This is creative/collaborative — about the feature, not the code.

### ⚠️ CORE RULE — CONTEXT-DRIVEN INTERACTION

The amount of interaction adapts to the input. Rich context (detailed task description + existing docs + clear codebase) needs less ceremony. Vague requests on unfamiliar codebases need more. The goal is **resolved ambiguity**, not a fixed question count.

**The `tp-design-gate` hook WILL BLOCK `brief.md` writes** if `manifest.designPhase` has not reached `brief-writing`. You cannot skip ahead. The gate checks for `contextConfirmed` and `ambiguitiesResolved` flags — not question counters.

If the user explicitly says "just do it" / "skip design" / "go ahead" during this phase, treat it as a mid-flow escape to light mode: set `manifest.designInteraction.userEscaped = true`, log a degradation, write a minimal brief, and set `manifest.designPhase = "brief-writing"`. The `userEscaped` flag bypasses all interaction evidence checks in the design gate.

### Context Gathering (both modes, before first user interaction)

Before asking anything, gather what's already known. Do NOT ask the user something the documentation or invocation already answers.

**Three context sources** (+ optional memplex):

1. **Invocation context** — task description, flags, referenced files (`--plan`, PRD)
2. **Project documentation** — INTENT.md, CLAUDE.md, README.md, product/brief.md, CONVENTIONS.md
3. **Quick codebase scan** — 3-5 Grep/Glob calls on target area
4. **Memplex knowledge** (if `manifest.memplexAvailable`):
   - `search_knowledge` scoped to task description — past decisions, rejected approaches, known constraints about this area
   - `search_conversations` — if context density is low, check for prior discussions about similar features
   - Store results in `manifest.memplexContext.intentExploration`
   - These results may raise context density (e.g., low → medium if memplex has past decisions)
   - If memplex not available: skip, rely on sources 1-3 only

**Record context density** in `manifest.designInteraction`:
```json
{
  "contextSources": ["task-description", "INTENT.md", "CLAUDE.md", "codebase-scan", "memplex"],
  "contextDensity": "high|medium|low"
}
```

**Context density heuristic:**
- **High**: Detailed task description + existing docs cover purpose/scope/constraints, OR `--plan` with lifecycle-validated plan, OR product/brief.md exists with relevant journeys
- **Medium**: Task description is clear but docs are sparse, OR docs exist but target area is unfamiliar
- **Low**: Vague task description, no relevant docs, unfamiliar codebase area

**This scan must complete in < 30 seconds.** Do NOT front-load exhaustive research.

### Light Mode (`--light`)

**designPhase enters as: `intent-exploration`**

Adaptive minimal interaction:

1. **Synthesize and confirm**: Based on all context sources, present your understanding:
   - **High context**: "From {sources}, I understand: {summary}. Is that right?" — expects yes/correction.
   - **Medium/Low context**: "Here's what I see: {summary}. {Specific gap question}."
   - **After user confirms/corrects**: Set `manifest.designInteraction.contextConfirmed = true`
   - Increment `manifest.designInteraction.intentQuestionsAsked` and `intentQuestionsAnswered` (counters kept for observability)

2. **Fill remaining gaps** (0-2 questions, only if ambiguity remains):
   - Ask only what isn't covered by context sources
   - If the confirmation in step 1 was comprehensive, skip directly to step 3
   - After all gaps resolved: Set `manifest.designInteraction.ambiguitiesResolved = true`

3. **Set `manifest.designPhase = "brief-writing"`**.

4. **Write brief.md**: Minimal brief with:
   - Intent (1-2 sentences)
   - Scope (in/out list)
   - Acceptance criteria (3-5 bullet points)
   - File areas to modify

**Gate criteria for light mode**: `contextConfirmed = true`.

### Full Mode (`--full`, default)

**designPhase enters as: `intent-exploration`**

Collaborative design flow. Each numbered step below is a **separate interaction turn** with the user — do NOT combine multiple steps into one message. The number of questions adapts to context density.

### ⚠️ Interaction-First Principle (HARD RULE)

The user MUST be engaged within 30 seconds of entering Sub-phase B. Research is pulled by user answers, not pushed before them. Maximum 2 minutes of silent work between any two user interactions.

**Anti-pattern — NEVER do this:**
- ❌ Run 10-20 file reads, cm queries, and convention scans for 5+ minutes, THEN ask the first question
- ❌ Present a fully-formed plan and ask "does this look right?" without prior interaction
- ❌ Front-load all research before any user interaction
- ❌ Ask questions the documentation already answers
- ❌ Force questions when context is already sufficient — don't add ceremony for ceremony's sake

**Step 1 — Synthesize and confirm (< 30 seconds, MANDATORY user interaction):**
- Use the context gathered above (invocation + docs + scan)
- **FIRST USER INTERACTION** — adapts to context density:
  - **High context**: "From {sources}, here's my understanding: {summary of intent, scope, constraints}. Does this capture it, or is there more to consider?"
  - **Medium context**: "Here's what I understand: {summary}. Before I dig deeper: {specific gap — scope boundary, constraint, or priority question}."
  - **Low context**: "I have a rough picture from the codebase. Can you help me understand: what's the goal here, and who's it for?"
- This response determines the direction of ALL subsequent research
- **After user responds**: Set `manifest.designInteraction.contextConfirmed = true`
- Increment `manifest.designInteraction.intentQuestionsAsked` and `intentQuestionsAnswered`

**Step 2 — Targeted questions (adaptive — 0 to 5, interleaved with research):**
Based on user's Step 1 answer → targeted file reads + cm queries (scoped by user's direction). Then ask about remaining gaps:

- Each user answer triggers focused research on the specific area they confirmed
- Each research round may produce the next question — or may resolve the gap without needing to ask
- Focus on: purpose, constraints, success criteria, conventions specific to this area
- Only one question per message — wait for the answer before asking the next
- **Stop when ambiguity is resolved** — not when a counter is hit
- **After all gaps resolved**: Set `manifest.designInteraction.ambiguitiesResolved = true`
- Continue incrementing `intentQuestionsAsked` / `intentQuestionsAnswered` for observability

**Sufficiency check** — you have enough context when you can answer all of these:
- What is being built and why?
- Who is affected and how?
- What's in scope and what's explicitly out?
- What are the key constraints or risks?
- What does "done" look like?

If the invocation + docs + Step 1 confirmation already cover these, Step 2 can have zero follow-up questions.

**Step 3 — Map user journey** (present to user, get confirmation):
- Before/after states for the user
- Emotional outcome the implementation should achieve
- Key interaction points
- Edge cases and failure scenarios

**Step 4 — Scope boundaries** (confirm with user):
- What's explicitly in scope
- What's explicitly out of scope
- What could be deferred to a future task

**→ After steps 2-4 complete, set `manifest.designPhase = "approach-review"`**

**Step 5 — Propose approaches** (MANDATORY user interaction):
- Present 2-3 approaches with trade-offs
- Lead with your recommended option and explain why
- Include effort estimate relative to each other (not time)
- **After presenting approaches**: Set `manifest.designInteraction.approachesProposed = true`
- `AskUserQuestion`: "Which approach do you prefer? Or suggest an alternative."
- **After user selects**: Set `manifest.designInteraction.approachSelected = true`

**→ After user selects an approach, set `manifest.designPhase = "design-approval"`**

**Step 6 — Section-by-section design approval** (MANDATORY user interaction):
- Present the design section by section, scaled to complexity
- Ask after EACH section whether it looks right: `AskUserQuestion`
- **After presenting each section**: Increment `manifest.designInteraction.sectionsPresented`
- **After user approves each section**: Increment `manifest.designInteraction.sectionsApproved`
- Sections to cover: architecture approach, components, data flow, error handling, testing strategy
- Be ready to revise if something doesn't make sense
- Do NOT present all sections at once — one section per message

**→ After all sections approved, set `manifest.designPhase = "brief-writing"`**

**Gate criteria for full mode**: `contextConfirmed && ambiguitiesResolved && approachSelected && sectionsApproved >= 1`.

**Step 7 — Write brief.md** with approved design:
- User stories with stable identifiers (US-1, AC-1.1, SC-1)
- Acceptance criteria derived from the approved design (not invented)
- Chosen approach with rationale from step 5
- Scope boundaries from step 4
- Success criteria that map to test cases

**Step 7b — Goal traceability** (if `INTENT.md` exists in project root):

Map each acceptance criterion in brief.md back to a success criterion or quality priority in INTENT.md. This ensures the task serves the project's stated purpose — not just "does the code work" but "does this advance the project's goals."

For each AC in the brief, find the INTENT.md criterion it serves:
```json
manifest.goalTraceability = {
  "intentFile": "INTENT.md",
  "mappings": [
    { "ac": "AC-1.1: User can log in with email", "intentCriteria": "Users can authenticate in < 3 seconds", "intentSection": "Success Criteria" },
    { "ac": "AC-1.2: Passwords hashed with bcrypt", "intentCriteria": "Zero credential exposure", "intentSection": "Quality Priorities" },
    { "ac": "AC-2.1: Rate limiting on login", "intentCriteria": null, "intentSection": null, "note": "Security hardening — no direct intent mapping" }
  ],
  "coverage": { "mapped": 2, "unmapped": 1, "total": 3 }
}
```

**Rules:**
- Not every AC needs a mapping — implementation details (rate limiting, caching) may not trace to INTENT.md and that's fine
- Flag if **zero** ACs map to INTENT.md — the task may be disconnected from project goals
- Do NOT block on unmapped ACs — this is informational, not a gate
- If INTENT.md doesn't exist: skip entirely, set `manifest.goalTraceability = null`

### Hardening Expectations (standard + enterprise)

After the brief is confirmed:
> "This task will include production hardening checks (dependency scan, secrets scan, type safety, error handling, test coverage). {For enterprise: Hardening failures will block completion.} {For standard: Hardening issues will be flagged as warnings.}"

Lean profile: skip this note.

### If `--skip-design` was specified

Log degradation:
```json
{ "type": "skip", "phase": "init", "reason": "User chose to skip design phase (--skip-design)", "timestamp": "ISO" }
```
Set `manifest.designPhase = "brief-writing"`. Write a minimal brief.md from the task description (no user interaction). Proceed directly to planning.

---

## Step 5: Quality Profile Selection

Default profile by route:
- Standard → `standard`
- Team → `standard`
- Blueprint → `enterprise`
- If task description is trivial (single file, obvious fix) → `lean`

Override: User can request a different profile during Sub-phase B.

| Profile | What it adds |
|---------|-------------|
| **lean** | Minimal flow. No traceability, no NFRs, no readiness verdict. |
| **standard** | + traceability.json, degradations in manifest, NFRs in brief/spec, verification plan |
| **enterprise** | + dependency/license audit, migration safety, operability check, readiness verdict |

Set `manifest.qualityProfile`.

## Step 5b: Feature Branch Creation

1. **Skip if** `manifest.git.available === false` or `manifest.git.config.createBranch === false`
2. **Lean override**: if lean profile and `createBranch` not explicitly set → default to `false`
3. **Protected branch check**: If on a protected branch → create feature branch. If not → ask user.
4. **Detect type** from task description: `feat`/`fix`/`refactor`/`chore`/`docs`
5. **Generate branch name**: Apply template, sanitize, max 63 chars
6. **Create branch**: `git checkout -b {branchName}`
7. **Update manifest**: `manifest.git.branch`, `manifest.git.baseBranch`, `manifest.git.branchCreated`

## Step 5c: Initialize Quality Artifacts

For `standard` and `enterprise` profiles:
- **Standard + Enterprise**: Create `traceability.json` with empty initial structure
- **Enterprise only**: Also create `readiness.json`

Initialize each with `taskId` and empty arrays/objects.

**Lean profile**: Skip this step entirely.

---

## Next

Update manifest phase to `"brief"`. Read the planning phase file: `~/.claude/taskplex/phases/planning.md`
