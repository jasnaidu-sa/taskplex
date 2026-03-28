# TaskPlex + Memplex Integration Specification

## Overview

Memplex is an optional enhancement for TaskPlex. When available, it provides cross-session knowledge persistence — error resolutions, file coupling, past decisions, conventions, and correction history. When unavailable, TaskPlex operates fully using codebase scanning, with no degradation in workflow correctness.

**Design principles:**
- Memplex is never required — every workflow path has a non-memplex fallback
- No fabricated counterfactuals — never claim "memplex would have caught this" when you can't know that
- Capability awareness, not pressure — inform users what memplex does, not what they're missing
- Check once, use everywhere — detect availability at init, store in manifest, reference throughout

---

## 1. Availability Detection

### When: Phase 0 (init.md), Step 3 — Load Context

After creating the manifest and session file, before loading project context:

```
Check if memplex MCP tools are available:
  → Try: ToolSearch for "mcp__mp__search_knowledge"
  → If found: set manifest.memplexAvailable = true
  → If not found: set manifest.memplexAvailable = false
```

Store in manifest for all subsequent phases to reference. No retry, no error — a single boolean check.

### First-session note (when memplex is not available)

On the first task in a project (no prior `.claude-task/` exists), if memplex is not available, include in the initialization output:

> "TaskPlex will use codebase scanning for context gathering. For cross-session knowledge persistence (error resolutions, file coupling, past decisions), memplex is available as an optional enhancement."

This note appears **once per project**, not per task. Track via `manifest.memplexNoticeShown` or a flag in the task directory.

### When memplex IS available

No note needed. Use it silently. The value speaks through the workflow quality — faster context gathering, fewer build-fix rounds, better agent prompts.

---

## 2. Integration Points by Phase

### 2.1 Phase -1: Bootstrap (INTENT.md)

**With memplex:**
| Call | Purpose | Timing |
|------|---------|--------|
| `search_knowledge` scoped to project name | Find prior context about this project from other sessions | During context gathering (Step 0), before first user question |
| `query_knowledge_graph` | Understand project architecture, entity relationships | During context gathering |

Results feed into the context synthesis — the orchestrator has richer input for its first question to the user.

**Without memplex:**
Standard fallback: README.md, CLAUDE.md, package.json, structural scan. Fully functional. The first question may be broader because there's less prior context to synthesize.

### 2.2 Phase 0, Sub-phase A: Convention Check

**With memplex:**
| Call | Purpose | Timing |
|------|---------|--------|
| `search_knowledge("conventions {target area}")` | Find conventions discovered in past tasks | Before automated convention scan |
| `file_intelligence` on target area files | Known issues, coupled files, change patterns | After convention scan, before questions |

Conventions from memplex supplement the automated scan. If memplex has a convention that the scan misses (e.g., "this module always uses Result types for errors"), it's included in the context.

**Without memplex:**
Convention scan runs from scratch. Infers patterns from code. Works correctly but may miss conventions that were explicitly confirmed by the user in a prior session and aren't obvious from code alone.

### 2.3 Phase 0, Sub-phase B: Intent Exploration

**With memplex:**
| Call | Purpose | Timing |
|------|---------|--------|
| `search_knowledge` scoped to task description | Past decisions about this area, rejected approaches, known constraints | During context gathering (before first user question) |
| `search_conversations` | Prior discussions about similar features or the same area | If context density is low and ambiguity remains |

Past decisions are especially valuable here. If a previous task decided "we use JWT not session cookies for auth," that decision should inform the current task's design — the user shouldn't have to re-state it.

**Without memplex:**
Context comes from invocation + docs + codebase scan. The user may need to re-provide context that was discussed in a prior session. This is observable by the user but not by the system — the system can't know what it doesn't know.

### 2.4 Phase 1: Planning

**With memplex:**
| Call | Purpose | Timing |
|------|---------|--------|
| `file_intelligence` on primary files from spec | Coupled files the plan should account for, known issues | Before spawning planning agent |
| `get_file_coupling` | Files commonly edited together | Before spawning planning agent |
| `search_knowledge` scoped to implementation patterns | Patterns used for this type of work in this project | Before spawning planning agent |

Results are included in the planning agent's prompt context. The planner starts with knowledge of file coupling and past patterns rather than discovering them during planning.

**Without memplex:**
Planning agent reads the spec, brief, and conventions. Discovers coupling by reading imports and references. May miss non-obvious coupling (e.g., "when you change the API types, you also need to regenerate the client SDK").

### 2.5 Phase 2: Implementation (Agent Spawning) — Critical Integration Point

This is where memplex has the most impact. Subagents **cannot call MCP tools** — they start from zero. The orchestrator must inject memplex context into each agent's prompt.

**With memplex — orchestrator pre-spawn assembly:**

For each implementation agent, before spawning:

```
1. Identify the agent's primary files (from spec/worker-brief)
2. For each primary file:
   → file_intelligence(file) — coupled files, known issues, change patterns
   → get_error_resolution(file) — previously solved errors in this file
3. search_knowledge scoped to the agent's section — patterns, decisions
4. Assemble results into a "Known Context" block
5. Include in the agent's prompt:
   "## Known Context (from project memory)
    File coupling: {X is commonly edited with Y}
    Known issues: {Z has a race condition on concurrent writes}
    Error resolutions: {TypeError in Z:45 — resolved by adding null check}
    Patterns: {This module uses the repository pattern for data access}"
```

This is the **context assembly pattern** — the orchestrator hydrates agents with knowledge they can't access themselves.

**Without memplex:**
Agent receives: spec, brief, conventions, and whatever the orchestrator gathered from the codebase scan. Agent discovers coupling and patterns by reading code. May re-encounter and re-debug errors that were already solved in past sessions.

**What's observable without memplex:**
- Build-fix rounds used (a metric, not a counterfactual)
- Files modified that weren't in the original plan (possible coupling miss)
- Time spent debugging (if tracked)

### 2.6 Phase 4.5: QA

**With memplex:**
| Call | Purpose | Timing |
|------|---------|--------|
| `get_error_resolution` | Check if a bug found during QA was seen before | When bugs are found in Steps 4.5.2-4.5.4 |
| `search_knowledge("regression {area}")` | Known regression patterns in this area | During edge case probing |

If a QA bug matches a known error resolution, the fix is applied immediately — potentially saving a full fix round.

**Without memplex:**
QA finds bugs. Build-fixer debugs them from scratch. Standard workflow, fully functional.

### 2.7 Phase 5: Validation (Build Fixer)

**With memplex:**
| Call | Purpose | Timing |
|------|---------|--------|
| `get_error_resolution` per unique error | Check if this exact error was solved before | Before each fix attempt |

The build-fixer agent prompt already references this: "check for error resolutions before attempting a fix." When memplex is available, the orchestrator includes known resolutions in the build-fixer's prompt context (same pre-spawn assembly pattern as 2.5).

**Without memplex:**
Build-fixer debugs from scratch. Uses standard diagnostic approach. Works correctly but may use more fix rounds for errors that were previously solved.

### 2.8 Phase 5: Completion (Knowledge Persistence)

**With memplex:**
| Call | Purpose | Timing |
|------|---------|--------|
| `write_knowledge` | Save implementation patterns, decisions, file coupling | After validation passes, before git |
| `write_knowledge` | Save error resolutions discovered during build-fix | After validation passes |

**What to save:**
- File coupling discovered during implementation (files modified together)
- Error resolutions from build-fixer rounds (error pattern + fix)
- Architectural decisions from the design phase (approach chosen + rationale)
- Convention confirmations from Sub-phase A (user-confirmed patterns)

**Without memplex:**
Knowledge dies with the session. Patterns, couplings, and error resolutions are not persisted. The next session on the same codebase starts from scratch.

**Completion note (when memplex is not available):**

In the task summary, after the metrics:

> "Patterns discovered during this task: {N} file couplings, {M} error resolutions, {P} architectural decisions. Cross-session knowledge persistence requires memplex."

This is factual — these things were discovered and they won't persist. No claim about what past sessions knew. No urgency. Just a statement of what happened.

**Completion note (when memplex IS available):**

> "Knowledge saved: {N} file couplings, {M} error resolutions, {P} patterns. Available for future sessions."

Reinforces value for existing users.

---

## 3. Graceful Degradation Pattern

### Code Pattern (for phase files)

Every memplex call in the workflow follows this pattern:

```
If manifest.memplexAvailable:
  → Call memplex tool
  → Include results in context / agent prompt / decision
  → Record what was found in manifest.memplexContext.{phase}
Else:
  → Skip silently
  → Use fallback (codebase scan, grep, read imports)
  → No error, no warning, no degradation logged
```

Memplex enhancement is never a degradation when absent — it's a bonus when present. The `degradations[]` array is for workflow deviations (skipped design, failed gates), not for optional features being unavailable.

### What NOT to do

- Never log "memplex unavailable" as a degradation
- Never block a workflow step waiting for memplex
- Never retry a failed memplex call — if it errors, skip and continue
- Never fabricate what memplex "would have known" — you can't know the counterfactual
- Never prompt the user to install memplex during a task
- Never mention memplex more than once per task (init note OR completion note, not both)

---

## 4. Upsell Strategy (Honest, Subtle)

### What's honest

Statements about **capabilities** (what memplex does in general):
- "Cross-session knowledge persistence requires memplex."
- "Project memory tracks error resolutions and file coupling across sessions."
- "Patterns discovered during this task will not persist to future sessions."

Statements about **observable facts** (what happened in this task):
- "Build-fixer used 3/3 rounds." (fact — no attribution to memplex absence)
- "5 file couplings discovered during implementation." (fact — they exist)
- "Knowledge saved: 3 error resolutions." (fact — for memplex users)

### What's dishonest

Specific counterfactual claims:
- "Memplex would have caught this error." (you don't know that)
- "This build-fix round would have been avoided with project memory." (you can't prove it)
- "The user wouldn't have needed to re-explain this." (you don't know what memplex stored)

### Upsell touchpoints (max 1 per task)

| Touchpoint | When | What to say | Frequency |
|---|---|---|---|
| Init (first task only) | memplex not detected, first task in project | General capability statement | Once per project |
| Completion | Task summary, memplex not available | "Patterns discovered: {N}. Cross-session persistence requires memplex." | Once per task |
| Completion | Task summary, memplex available | "Knowledge saved: {N} patterns, {M} resolutions." | Once per task |

**Choose init OR completion, not both.** If the init note was shown (first task), skip the completion note. On subsequent tasks, use the completion note only.

---

## 5. Context Assembly for Agents

Since agents cannot call MCP tools, the orchestrator assembles memplex context and includes it in the agent's prompt. This is the core pattern that makes memplex valuable for multi-agent workflows.

### Assembly Template

```markdown
## Known Context (from project memory)

### File Intelligence
{For each primary file the agent will modify:}
- **{file}**: Coupled with {list}. Known issues: {list or "none"}.
  Change patterns: {typical co-changes}.

### Error Resolutions
{For each known error in target files:}
- **{error pattern}** in {file}:{line} — Resolution: {fix description}

### Relevant Knowledge
{From search_knowledge scoped to agent's section:}
- {Pattern, decision, or convention relevant to this work}

### Corrections
{From search_knowledge("corrections {area}"):}
- {Past corrections the user made about this area}
```

### When to assemble

| Route | When | For whom |
|---|---|---|
| Standard | Before spawning implementation agent | implementation-agent |
| Team | Before spawning each worker | Each implementation-agent |
| Blueprint | Before spawning each worktree agent | Each implementation-agent |
| All routes | Before spawning build-fixer | build-fixer |
| All routes | Before spawning planning agent | planning-agent |

### Assembly budget

Limit memplex calls per agent to avoid latency:
- `file_intelligence`: max 3 files per agent
- `get_error_resolution`: max 5 errors per agent
- `search_knowledge`: 1 call per agent, scoped to section
- Total: ~5-9 MCP calls per agent spawn, taking < 5 seconds

### Without memplex

The "Known Context" block is omitted from the prompt. The agent receives spec, brief, and conventions only. No placeholder, no "memplex unavailable" note in the agent prompt.

---

## 6. Manifest Schema Additions

```json
{
  "memplexAvailable": {
    "type": "boolean",
    "default": false,
    "description": "Whether memplex MCP tools were detected at task init"
  },
  "memplexNoticeShown": {
    "type": "boolean",
    "default": false,
    "description": "Whether the first-task capability notice was shown (once per project)"
  },
  "memplexContext": {
    "type": ["object", "null"],
    "default": null,
    "description": "Context retrieved from memplex during this task, organized by phase",
    "properties": {
      "conventionCheck": { "type": "object", "description": "Conventions from past tasks" },
      "intentExploration": { "type": "object", "description": "Past decisions, rejected approaches" },
      "planning": { "type": "object", "description": "File coupling, patterns for planning" },
      "implementation": { "type": "object", "description": "Per-agent assembled context" },
      "qa": { "type": "object", "description": "Error resolutions found during QA" },
      "validation": { "type": "object", "description": "Error resolutions for build-fixer" }
    }
  },
  "knowledgeSaved": {
    "type": ["object", "null"],
    "default": null,
    "description": "Knowledge written to memplex at task completion",
    "properties": {
      "fileCouplings": { "type": "integer", "default": 0 },
      "errorResolutions": { "type": "integer", "default": 0 },
      "patterns": { "type": "integer", "default": 0 },
      "decisions": { "type": "integer", "default": 0 }
    }
  }
}
```

---

## 7. Phase File Changes Required

| File | Change | Section |
|------|--------|---------|
| `init.md` Step 3 | Add memplex availability check | Load Context |
| `init.md` Sub-phase A | Add memplex convention lookup (if available) | Convention Check |
| `init.md` Sub-phase B | Add memplex knowledge search in context gathering | Intent Exploration |
| `planning.md` all routes | Add pre-spawn context assembly pattern | Before each agent spawn |
| `planning.md` build gate | Add error resolution lookup for build-fixer | Build gate sections |
| `qa.md` Step 4.5.5 | Add error resolution check before fix attempts | Bug fix loop |
| `validation.md` Step 6 | Add error resolution to build-fixer context | Build-fixer spawn |
| `validation.md` Step 11 | Add knowledge persistence at completion | Post-completion |
| `taskplex.md` command | Add memplex summary to task summary format | Completion output |

---

## 8. Implementation Priority

| Integration | Impact | Effort | Priority |
|---|---|---|---|
| Pre-spawn context assembly (2.5) | High — agents start informed | Medium — template pattern per spawn point | Do first |
| Knowledge persistence at completion (2.8) | High — builds the flywheel | Low — write_knowledge calls at completion | Do second |
| Convention check enhancement (2.2) | Medium — richer convention context | Low — one search_knowledge call | Do third |
| Intent exploration enhancement (2.3) | Medium — fewer repeated questions | Low — one search_knowledge call | Do fourth |
| Error resolution in build-fixer (2.7) | Medium — fewer fix rounds | Low — pre-spawn assembly | Do fifth |
| QA error matching (2.6) | Low — QA bugs rarely repeat | Low — one call per bug | Do sixth |
| Bootstrap enhancement (2.1) | Low — runs once per project | Low — one call | Do last |
