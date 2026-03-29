# TaskPlex Interrupt Handling Specification

## Problem

During long-running tasks, the user may need to:
1. Expand scope ("also add rate limiting")
2. Pivot the design ("actually use session tokens not JWT")
3. Ask a side question ("how does the existing auth middleware work?")
4. Switch tasks ("pause this, fix CI first")
5. Add a deferred note ("remind me to update docs after")

Currently, the orchestrator treats every user message as a response to the current workflow step. There's no way to inject a side-channel request without derailing the workflow, confusing the checklist state, or losing context.

## Design

### Input Classification

Every user message during an active task is classified before routing. Classification happens in the orchestrator's reasoning, informed by the `tp-prompt-check` hook's context injection.

| Class | Signal Patterns | Routing |
|---|---|---|
| **workflow-response** | Direct answer to a question the orchestrator asked, numbered choice, "yes"/"no", design feedback | Feed into current step. Normal workflow progression. |
| **scope-addition** | "Also add...", "Include...", "What about...", "Can we also...", "One more thing..." | Queue in `manifest.interruptQueue[]`. Address at next checkpoint. |
| **design-pivot** | "Actually...", "Change to...", "Instead of...", "I changed my mind...", "Let's go with X not Y" | Queue as high-priority interrupt. Pause at next checkpoint. Present rollback options. |
| **side-question** | "How does...", "What is...", "Show me...", "Can you explain...", "Where is..." | Answer inline. Do NOT advance checklist. Do NOT update manifest phase. Resume workflow after answering. |
| **task-switch** | "Pause this", "Do X first", "Stop this task", "Switch to...", "Handle X before continuing" | Checkpoint current state. Start new context. Flag for resume. |
| **deferred-note** | "Remind me...", "After this...", "Don't forget...", "Note for later...", "TODO:..." | Append to `.claude-task/{taskId}/deferred/user-notes.md`. Acknowledge briefly. Continue workflow. |
| **meta-feedback** | "You're going too slow", "Skip this step", "Just do it", "Speed up" | Apply workflow modifier (e.g., switch to light mode, mid-flow escape). Continue. |

### Classification Method

The orchestrator classifies using context, not a separate LLM call:

1. **Check if there's a pending question** — if the orchestrator asked something and the user responds, default to `workflow-response`
2. **Pattern match** against the signal patterns above
3. **If ambiguous** — default to `workflow-response` (safest). The user can always clarify.

The classification should be a brief internal note, not displayed to the user:
```
[Interrupt classification: scope-addition — "also add rate limiting to auth endpoints"]
```

---

## Interrupt Types in Detail

### 1. Scope Addition

**Detection**: User mentions new functionality, additional requirements, or expanded scope during an active task.

**Handling**:
1. Acknowledge briefly: "Noted — I'll address rate limiting at the next checkpoint."
2. Append to `manifest.interruptQueue[]`:
   ```json
   {
     "id": "int-001",
     "type": "scope-addition",
     "content": "Add rate limiting to auth endpoints",
     "receivedAt": "ISO timestamp",
     "receivedDuring": { "phase": "implementation", "step": "5.1" },
     "status": "queued",
     "addressedAt": null
   }
   ```
3. Continue current workflow step without interruption.
4. At the **next natural checkpoint** (see Checkpoint Boundaries below), present the queued scope additions:
   > "Before proceeding, you mentioned adding rate limiting to auth endpoints.
   >
   > 1. **Add to current task** — expand brief + spec, implement as part of this task
   > 2. **Defer** — log as a follow-up task, complete current scope first
   > 3. **Discard** — not needed"

5. If **Add to current task**:
   - If still in design/planning: update brief.md and spec.md inline
   - If in implementation: add to spec, dispatch additional agent or extend current agent's scope
   - Update `manifest.scopeAdditions[]` with the addition details
   - Log a manifest note: `progressNotes.push({ text: "Scope expanded: rate limiting added", status: "active" })`

6. If **Defer**: Write to `.claude-task/{taskId}/deferred/scope-addition-{id}.md`

### 2. Design Pivot

**Detection**: User changes direction on a previously agreed design decision.

**Handling**:
1. Acknowledge: "Understood — switching from JWT to session tokens is a design change. I'll address this at the next checkpoint."
2. Append to `manifest.interruptQueue[]` with `type: "design-pivot"` and `priority: "high"`
3. Continue current step (don't abort mid-agent).
4. At the **next natural checkpoint**, present rollback options:
   > "You want to switch from JWT to session tokens. This affects:
   > - brief.md (approach section)
   > - spec.md (auth implementation)
   > - {list of already-implemented files, if any}
   >
   > Options:
   > 1. **Rollback to planning** — update brief + spec, re-implement affected sections
   > 2. **Patch in place** — modify the existing implementation to use session tokens
   > 3. **Cancel pivot** — keep JWT as originally planned"

5. If **Rollback**: Set `manifest.phase` back to the appropriate point, update brief/spec, log degradation with reason
6. If **Patch**: Continue implementation with modified spec section, note the in-flight change
7. Either way: update `manifest.designInteraction` to reflect the pivot decision

**Impact assessment**: Before presenting options, quickly assess what's already built:
- If pivot arrives during design phase: low cost, just update the brief
- If pivot arrives during implementation: medium cost, may need to re-do agent work
- If pivot arrives during validation: high cost, may need to go back to implementation

### 3. Side Question

**Detection**: User asks about something without requesting a change to the task.

**Handling**:
1. Answer the question directly.
2. **Do NOT**:
   - Advance the checklist
   - Update `manifest.phase` or `manifest.designPhase`
   - Increment any interaction counters
   - Write to any task artifacts
   - Treat the answer as a step completion
3. After answering, resume the workflow from exactly where it was:
   > "{answer to the question}
   >
   > Continuing with {current step description}..."
4. If the question is long/complex and would fill context, offer to answer briefly:
   > "That's a broader topic. Quick answer: {2-3 sentences}. Want me to go deeper after this task, or is that enough for now?"

**Manifest impact**: None. Side questions are invisible to the workflow state.

### 4. Task Switch

**Detection**: User wants to pause the current task and do something else.

**Handling**:
1. **Checkpoint current state**:
   - Update `manifest.phaseChecklist` to reflect exact position
   - Write progress notes with current status
   - Set `manifest.status = "paused"` (new status value)
   - Update `manifest.lastUpdated`
2. Confirm: "Task paused at {phase}/{step}. State saved to manifest."
3. **Start the new work**:
   - If the new work is a `/tp` task: normal new task flow (prompt-check hook will detect the paused task)
   - If it's a quick fix: handle inline without creating a new task
   - If it's a question: answer and offer to resume
4. **Resume**: When the user is done, they can say "resume the auth task" or run `/resume`. The session-start hook already handles this — it detects in-progress (now also paused) tasks and injects recovery context.

**New manifest status**: Add `"paused"` to the phase enum. Paused tasks are shown by session-start but not auto-resumed — the user must explicitly resume.

### 5. Deferred Note

**Detection**: User wants to remember something for later.

**Handling**:
1. Acknowledge briefly: "Noted for after this task."
2. Write to `.claude-task/{taskId}/deferred/user-notes.md`:
   ```markdown
   ## User Notes

   - [2026-03-29 14:30] Update docs after auth implementation
   - [2026-03-29 15:15] Check if rate limiting needs Redis
   ```
3. Continue workflow without interruption.
4. At **task completion** (Step 11), present deferred notes:
   > "Deferred items from this task:
   > - Update docs after auth implementation
   > - Check if rate limiting needs Redis
   >
   > Address now or save for later?"

### 6. Meta-Feedback

**Detection**: User commenting on the workflow itself, not the task content.

**Handling**: Apply the appropriate modifier:
- "Skip this step" → mid-flow escape, log degradation
- "Just do it" → switch to light mode
- "Speed up" → reduce question depth, batch remaining steps
- "Too many questions" → set `manifest.designInteraction.userEscaped = true`
- "You're doing great" → continue as-is (no action needed)

---

## Checkpoint Boundaries

Queued interrupts (scope additions, design pivots) are addressed at natural boundaries — never mid-step, never mid-agent.

| Phase | Checkpoint Location |
|-------|-------------------|
| Design (Sub-phase B) | Between each interaction step (3.1 → 3.2 → 3.3...) |
| Planning | Between spec writing and spec critic |
| Planning | At pre-implementation acknowledgment (natural pause point) |
| Implementation | After each agent returns (Standard: after single agent, Team/Blueprint: after each worker) |
| QA | Between QA rounds |
| Validation | Between validation gates |

**At each checkpoint**, the orchestrator:
1. Check `manifest.interruptQueue[]` for `status: "queued"` entries
2. If any: present them to the user before continuing
3. After resolution: set `status: "addressed"` with the decision
4. Continue workflow

**Implementation agents running in worktrees cannot be interrupted.** Scope additions during implementation are always deferred to after the agent returns.

---

## Manifest Schema Additions

```json
{
  "interruptQueue": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "type": { "type": "string", "enum": ["scope-addition", "design-pivot", "task-switch", "deferred-note", "meta-feedback"] },
        "priority": { "type": "string", "enum": ["normal", "high"], "default": "normal" },
        "content": { "type": "string" },
        "receivedAt": { "type": "string", "format": "date-time" },
        "receivedDuring": {
          "type": "object",
          "properties": {
            "phase": { "type": "string" },
            "step": { "type": "string" }
          }
        },
        "status": { "type": "string", "enum": ["queued", "addressed", "deferred", "discarded"] },
        "addressedAt": { "type": ["string", "null"], "format": "date-time" },
        "decision": { "type": ["string", "null"] }
      }
    }
  },
  "scopeAdditions": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "description": { "type": "string" },
        "addedAt": { "type": "string", "format": "date-time" },
        "phase": { "type": "string" },
        "affectedArtifacts": { "type": "array", "items": { "type": "string" } }
      }
    }
  }
}
```

Also add `"paused"` to the `status` enum: `["in-progress", "completed", "blocked", "cancelled", "paused"]`

---

## Implementation Priority

| Component | Where | Effort |
|-----------|-------|--------|
| Input classification logic | taskplex.md command + init.md | Low — prompt instructions |
| `interruptQueue` in manifest | manifest-schema.json | Low — schema addition |
| Checkpoint queue processing | All phase files — add "check interruptQueue" at each boundary | Medium — touches every phase |
| Side question handling | taskplex.md command | Low — instruction to not advance state |
| Task pause/resume | manifest status + session-start hook | Low — add "paused" status |
| Deferred notes | Completion phase + deferred/ directory | Low — already partially exists |
| Design pivot rollback | planning.md + init.md | Medium — phase rollback logic |

The input classification is the foundation — everything else depends on the orchestrator correctly identifying what the user wants. The classification itself is just prompt instructions (not a hook or LLM call), so it's low-effort but high-impact.

---

## Integration with Existing Systems

### Hooks
- `tp-prompt-check`: Could add interrupt detection at the hook level, injecting classification hints into the orchestrator context. But classification is better done by the orchestrator itself (it has conversation context the hook doesn't).
- `tp-session-start`: Already handles resume. Needs to also handle `status: "paused"` tasks.
- `tp-pre-compact`: Include `interruptQueue` in checkpoint snapshot.

### Memplex
- If memplex available: write scope additions and design pivots as knowledge entries (these are decisions that should persist)
- User notes could be saved to memplex for cross-session persistence

### Execution Continuity Rule
The interrupt model **does not contradict** the execution continuity rule. The rule says "don't ask should I continue between agents." The interrupt model says "if the USER initiates a change, queue it and address at the next checkpoint." These are different — one is about the orchestrator's initiative, the other is about the user's initiative.
