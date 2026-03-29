# Skill Evolution System

## Overview

Skills improve automatically based on execution feedback. When a task fails or the user corrects a skill's output, the correction is captured, attributed to the skill, and used to generate an improvement — without modifying the skill's source file until explicitly solidified.

## Architecture

```
Task Execution
     │
     ▼
Signal Detection (keyword-based, no LLM, runs at completion)
  ├── Execution failures: build-fix rounds > 0, agent escalations, QA failures
  └── User corrections: "that's wrong", "not what I wanted", design revisions
     │
     ▼
Attribution (which skill was active?)
  → manifest.workflow + manifest.activeCommand
  → Links signal to skill name
     │
     ▼
Evolution Generator (LLM call, runs if signals detected)
  → Input: current SKILL.md + attributed signals
  → Output: evolution entry (troubleshooting or example)
     │
     ▼
evolutions.json (per skill, staged)
  → Loaded before next skill invocation
  → Skill improves immediately without modifying source
     │
     ▼
/solidify (user-initiated)
  → Merges evolutions into SKILL.md
  → Permanent improvement, clears evolutions.json
```

## 1. Signal Detection

Runs during the completion phase (validation.md Step 11), after all validation passes. Purely keyword-based — no LLM cost.

### Failure Signals (from manifest)

| Signal | Source | Detection |
|--------|--------|-----------|
| Build-fix rounds used | `manifest.iterationCounts.buildFixRounds > 0` | Direct check |
| Agent escalations | `manifest.escalations.length > 0` | Direct check |
| QA failures | `manifest.qa.bugsFound > manifest.qa.bugsFixed` | Direct check |
| Review rejections | `manifest.iterationCounts.reviewRounds.{any} > 1` | Any review needed resubmission |
| Degradations | `manifest.degradations.length > 0` | Quality deviations logged |

### Correction Signals (from conversation — detected by hook or at completion)

Pattern matching against user messages during the task:

```javascript
const CORRECTION_PATTERNS = [
  /that'?s (wrong|incorrect|not right|not what I)/i,
  /no,?\s*(I|you|it|that|this)\s*(want|need|should|meant|mean)/i,
  /you misunderstood/i,
  /not what I (asked|wanted|meant)/i,
  /go back|undo|revert|redo/i,
  /I said|I meant|I wanted/i,
  /wrong approach|different approach/i,
];
```

These are checked against the conversation during design phase interactions. If the user corrected the orchestrator's understanding during Sub-phase B (intent exploration), that's a signal about the active skill's context gathering quality.

### Signal Threshold

Don't generate an evolution for every minor issue. Threshold:
- **Any** user correction → generate evolution (corrections are high-signal)
- **buildFixRounds >= 2** → generate evolution (repeated build failures)
- **escalations.length > 0** → generate evolution (agent gave up)
- **QA bugsFound > 2** → generate evolution (implementation quality issue)
- **Single build-fix round, no corrections** → skip (normal noise)

## 2. Attribution

Link the signal to the active skill:

```json
{
  "skill": "taskplex",
  "command": "/tp",
  "phase": "implementation",
  "signal": {
    "type": "failure",
    "category": "build-fix",
    "detail": "TypeScript type error in auth module — buildFixRounds: 3",
    "files": ["src/auth/types.ts", "src/auth/middleware.ts"]
  }
}
```

**How to determine the active skill:**
- `manifest.workflow` — "taskplex" if running via /tp
- For `/plan` tasks: skill = "plan"
- For `/evaluate` tasks: skill = "evaluate"
- If no workflow field: skill = "unknown" — don't generate evolution

## 3. Evolution Generator

An LLM call (using the orchestrator's model) that takes the signal + current skill and produces an improvement entry.

**Prompt template:**

```
You are a skill improvement system. A skill was used and produced a suboptimal outcome.
Your job is to generate a specific, actionable improvement that would prevent this issue
in future invocations.

SKILL FILE:
{contents of SKILL.md}

EXISTING EVOLUTIONS (if any):
{contents of evolutions.json}

SIGNAL:
{type}: {category}
{detail}
Files involved: {files}
Phase: {phase}

Generate ONE evolution entry. It must be:
- Specific to this failure/correction, not generic advice
- Actionable — the skill can apply it next time without human judgment
- Scoped — only about what went wrong, not a rewrite of the skill

Output JSON:
{
  "type": "troubleshooting" | "example",
  "title": "Short description of the improvement",
  "content": "The specific instruction, pattern, or example to add",
  "trigger": "When this evolution applies (file pattern, error pattern, or context)",
  "source": { "taskId": "...", "signal": "...", "date": "..." }
}
```

**Evolution types:**
- **Troubleshooting**: "When you see error X, the fix is Y" — from build-fix and escalation signals
- **Example**: "When the user says X, they mean Y" — from correction signals

## 4. evolutions.json Schema

Lives at `~/.claude/skills/{skill-name}/evolutions.json`. Created on first evolution, loaded before every skill invocation.

```json
{
  "schemaVersion": 1,
  "skill": "taskplex",
  "evolutions": [
    {
      "id": "evo-20260329-001",
      "type": "troubleshooting",
      "title": "TypeScript path alias resolution in monorepo",
      "content": "When implementing in a monorepo with path aliases (@/components, @/lib), always check tsconfig.json paths before assuming import structure. The build-fixer spent 3 rounds on path resolution errors that could have been avoided by reading tsconfig first.",
      "trigger": "When target project has tsconfig.json with paths field",
      "source": {
        "taskId": "TASK-add-auth",
        "signal": "build-fix:3-rounds",
        "date": "2026-03-29"
      },
      "status": "active",
      "createdAt": "2026-03-29T14:00:00Z"
    },
    {
      "id": "evo-20260329-002",
      "type": "example",
      "title": "User means 'accessible' not 'pretty' when asking for UI improvements",
      "content": "When the user asks to 'improve the UI', probe whether they mean accessibility (keyboard nav, screen readers, contrast) or aesthetics (colors, spacing, animation). In this project, the user consistently meant accessibility.",
      "trigger": "When task description mentions UI improvement or UX",
      "source": {
        "taskId": "TASK-improve-dashboard",
        "signal": "correction:user-meant-accessibility",
        "date": "2026-03-29"
      },
      "status": "active",
      "createdAt": "2026-03-29T15:00:00Z"
    }
  ]
}
```

### Loading Evolutions

When a skill is invoked (e.g., `/tp` triggers taskplex), the orchestrator checks for `~/.claude/skills/{skill}/evolutions.json`. If it exists and has active entries:

1. Read evolutions.json
2. Filter to `status: "active"` entries
3. For each, check if `trigger` matches the current context
4. Include matching evolutions in the skill's execution context:

```markdown
## Skill Evolutions (learned from past tasks)

### Troubleshooting
- **TypeScript path alias resolution in monorepo**: When target project has tsconfig.json
  with paths field — always check tsconfig.json paths before assuming import structure.

### Examples
- **User means 'accessible' not 'pretty'**: When task description mentions UI improvement —
  probe whether they mean accessibility or aesthetics.
```

This block is injected into the orchestrator's context when the skill is activated, before any phase file is read.

## 5. /solidify Command

User-initiated command that merges active evolutions into the skill's source file.

**File**: `~/.claude/commands/solidify.md`

**Usage**: `/solidify [skill-name]` or `/solidify --all`

**Process:**
1. Read `~/.claude/skills/{skill}/evolutions.json`
2. Present each active evolution to the user:
   > "Evolution: {title}
   > Type: {type}
   > Content: {content}
   > Source: {taskId} on {date}
   >
   > **Keep** / **Discard** / **Edit**"
3. For each "Keep": append to the appropriate section of SKILL.md
   - Troubleshooting entries → add to a "## Known Issues" or "## Troubleshooting" section
   - Example entries → add to a "## Examples" or "## Context Notes" section
4. For each "Discard": set `status: "discarded"` in evolutions.json
5. For each "Edit": user provides modified text, then append
6. After all processed: write updated SKILL.md and evolutions.json

**Anti-patterns:**
- Never auto-solidify — always require user review
- Never modify SKILL.md without /solidify — evolutions stay in evolutions.json until explicitly merged
- Never delete evolutions.json entries — mark as "solidified" or "discarded" for audit

## 6. Integration with TaskPlex Completion Phase

Add to `validation.md` Step 11, after knowledge persistence:

```
Step 11.5: Skill Evolution (if signals detected)

1. Run signal detection against manifest (see skill-evolution.md)
2. If signals exceed threshold:
   a. Determine active skill from manifest.workflow
   b. Read ~/.claude/skills/{skill}/skill.md (or SKILL.md)
   c. Read ~/.claude/skills/{skill}/evolutions.json (if exists)
   d. Call evolution generator (LLM) with signal + skill + existing evolutions
   e. Append new evolution to evolutions.json
   f. Record in manifest: manifest.skillEvolution = { skill, evolutionId, type, signal }
3. If no signals: skip
```

## 7. Integration with Memplex

**If memplex available:**
- `write_knowledge` with the evolution entry — makes it searchable across sessions
- Correction signals are cross-referenced with memplex correction entities for richer attribution
- `search_knowledge("evolution {skill}")` during evolution generation — avoid duplicate evolutions

**If memplex not available:**
- evolutions.json is the only persistence — still works, just local to this machine
- No cross-referencing, no dedup across sessions (acceptable — evolutions accumulate and /solidify curates)
