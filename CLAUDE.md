# TaskPlex — Structured Workflow Orchestration for AI Agents

## What TaskPlex Is

TaskPlex is a **workflow orchestration framework** that enforces structured, multi-phase development workflows for AI coding agents (Claude Code, pi). It wraps every development task in a design-first process with quality gates, user interaction checkpoints, and validation pipelines.

**Core thesis**: The governance infrastructure IS the product. Not the agents. Agents are commoditized — what differentiates is context architecture, validation, autonomy management, and audit trails.

## Architecture Overview

### Entry Points
- `/taskplex` or `/tp` — Claude Code slash commands (`~/.claude/commands/taskplex.md`, `tp.md`)
- `/plan` — Strategic thinking & architecture command (`~/.claude/commands/plan.md`)
- `/evaluate` — Product evaluation: audit and review modes (`~/.claude/skills/evaluate/`)
- `/solidify` — Merge skill evolutions into source (`~/.claude/commands/solidify.md`)
- `frontend` — Standalone frontend development skill (`~/.claude/skills/frontend/`)

### Workflow Phases (7 phases, sequential)
1. **Initialization** — Parse task, create manifest, create task list, detect memplex, load context, detect project type
2. **Design (Sub-phase A)** — Convention scan + memplex convention lookup + optional user questions
3. **Design (Sub-phase B)** — Adaptive intent exploration: gather 4 context sources, synthesize, confirm, resolve gaps
4. **Planning** — Spec writing by planning agent, spec critic review, user reviews full spec inline, refine task list
5. **Implementation** — Coherence check, build gate, documentation update. Team/Blueprint: delegate to agents (enforced by hook)
5.5. **QA** — Product-type-aware testing with memplex error resolution
6. **Validation** — 12-step pipeline: artifact, traceability, build, security, closure, code review, conditional, hardening, compliance
7. **Completion** — Memplex knowledge persistence, skill evolution, git commit, PR, task summary

### Execution Routes (3)
| Route | Flag | Agents | Use Case |
|-------|------|--------|----------|
| **Standard** | `--standard` | 1 agent | Default, single-threaded |
| **Team** | `--team` | 1-3 parallel agents | Independent sections |
| **Blueprint** | `--blueprint` | Opus architect + critics + multi-agent + worktrees | Complex features |

Initiative mode (`--prd`) extends Blueprint with feature decomposition and wave-based execution.

### Quality Profiles (3)
| Profile | Gates | Hardening |
|---------|-------|-----------|
| **Lean** | Minimal | Skipped |
| **Standard** | Full (security, closure, code review, compliance) | Advisory |
| **Enterprise** | Full + readiness + dependency/license/migration/operability | Blocking |

### Adaptive Interaction Model (v2)
Design phase uses **adaptive interaction** — question count driven by context density, not hard minimums. System gathers context from 4 sources (invocation, docs, codebase scan, memplex), synthesizes, then asks only about gaps.

**Gate criteria** (flags, not counters):
- **Full mode**: `contextConfirmed && ambiguitiesResolved && approachSelected && sectionsApproved >= 1`
- **Light mode**: `contextConfirmed`

## Key Enforcement Mechanisms

### Hooks (9 hooks, all wired in settings.json)
| Hook | Purpose | Enforcement |
|------|---------|-------------|
| `tp-design-gate` | Design gate: blocks writes before design complete. Implementation gate: blocks orchestrator source edits in team/blueprint | Hard |
| `tp-heartbeat` | Tracks every file edit, updates manifest, renders progress | Hard |
| `tp-pre-commit` | Blocks git commits without `validation-gate.json` | Hard |
| `tp-session-start` | Detects active tasks, renders checklist, instructs TaskCreate recreation | Advisory |
| `tp-pre-compact` | Checkpoints state (including phaseChecklist) before compaction | Advisory |
| `tp-prompt-check` | Detects /tp invocations, warns about active tasks, injects workflow reminder | Advisory |
| `tp-stop` | Warns/blocks on incomplete validation at session end | Conditional |
| `start-task-sentinel` | Tracks non-edit tool calls for compaction guard | Advisory |

### Key Rules
- **Execution Continuity**: After user approves plan, implementation runs non-stop — no "should I continue?"
- **User Visibility**: Present artifacts inline in conversation, never point to files. Pre-spawn status for every agent.
- **Mid-Task Changes**: User can redirect at any time. Task list and manifest must stay current.
- **Implementation Delegation**: Team/Blueprint must use agents (hook-enforced). Blueprint must use worktrees.

## Memplex Integration (Optional)

When available, enriches every phase with cross-session knowledge. When unavailable, skips silently.
- **Pre-spawn context assembly**: Orchestrator calls file_intelligence + get_error_resolution + search_knowledge before spawning each agent, includes as "Known Context" block
- **Knowledge persistence**: At completion, saves file couplings, error resolutions, patterns, decisions
- **Convention/intent enhancement**: Past decisions and corrections inform design phase
- **Honest upsell**: Capability statements only, never counterfactuals

## Skill Evolution

Skills improve from task feedback via staged pipeline:
- Signal detection (keyword-based, no LLM) at completion
- Evolution entries written to `evolutions.json` (per skill, staged)
- Loaded before next skill invocation
- `/solidify` merges approved evolutions into SKILL.md permanently

## File Locations

### Core Contracts
```
~/.claude/taskplex/
├── phases/ (init, planning, qa, validation, bootstrap, prd)
├── policy.json, gates.md, artifact-contract.md, manifest-schema.json
├── handoff-contract.md, hardening-checks.md, portability.md
└── skill-evolution.md
```

### Commands & Skills
```
~/.claude/commands/ — taskplex.md, tp.md, plan.md, solidify.md, drift.md
~/.claude/skills/evaluate/ — audit + review modes
~/.claude/skills/frontend/ — standalone frontend dev (design system, a11y, responsive, component spec)
~/.claude/skills/plan/ — planning trigger wrapper
```

### Agents (19 core + 3 utility)
```
Core: architect, planning-agent, implementation-agent, verification-agent,
      review-standards (shared reference), security-reviewer, closure-agent,
      code-reviewer, hardening-reviewer, database-reviewer, e2e-reviewer,
      user-workflow-reviewer, compliance-agent, researcher, merge-resolver,
      bootstrap, prd-bootstrap, strategic-critic, tactical-critic
Utility: build-fixer, drift-scanner, explore
```

### Hooks (9 files in ~/.claude/hooks/)
```
hook-utils.mjs, tp-design-gate.mjs, tp-session-start.mjs, tp-heartbeat.mjs,
tp-pre-commit.mjs, tp-pre-compact.mjs, tp-prompt-check.mjs, tp-stop.mjs,
start-task-sentinel.mjs
```

## Design Documents (this project directory)

| File | Purpose |
|------|---------|
| `taskplex-documentation.md` | Complete technical documentation |
| `memplex-integration.md` | Memplex integration spec + combined roadmap |
| `interrupt-handling.md` | Interrupt handling design (simplified to natural conversation) |
| `skill-evolution.md` | Skill evolution system design (copy of contract file) |
| `harness-engineering-gaps.md` | Harness gap analysis: narrative, verification, Playwright, drift |
| `multi-runtime-plan.md` | Cross-runtime distribution plan (Cursor, Codex, Gemini, Pi, OpenCode, Windsurf) |
| `runtime-research-april-2026.md` | Latest runtime extensibility research |
| `test-plan.md` | Comprehensive test plan for all features (15 tests) |
| `board-architecture.md` | CEO & Board multi-agent decision system for pi |
| `business-agent-framework.md` | Enterprise agent deployment research synthesis |
| `taskplex-pi-gap-analysis.md` | Claude Code → pi hook mapping with gap analysis |
| `taskplex-pi-plugin.md` | Complete pi plugin build specification |

## What's Built vs What's Designed

### Built and Active
- All commands (5), hooks (9), phase files (6), contract files (8), agent definitions (19+3)
- Three routes: Light / Standard (default, multi-agent) / Blueprint (architect + waves)
- Adaptive interaction model (v2) with flag-based gates
- Three-contract chain: intent traceability → test plan → verification
- Verification agent (adversarial, two modes: test-plan + verify)
- Anti-rationalization review standards across all reviewers
- Implementation gate enforcement (standard/blueprint delegation)
- Memplex integration (optional, graceful degradation, pre-spawn context assembly)
- Skill evolution pipeline (signal detection + evolutions.json + /solidify)
- Goal traceability (AC → INTENT.md) + intent traceability (AC → architecture)
- Narrative progress artifact (task narrative in progress.md, injected on resume)
- Playwright MCP for visual/browser verification
- Drift detection (/drift command + drift-scanner agent)
- Verification commands in agent handoffs (mandatory self-verification)
- Implementation coherence check (haiku, fast, pre-build-gate)
- Task list lifecycle (create → refine → update → recreate on resume)
- Presentation detail levels (structured summaries, not full artifacts or terse one-liners)
- Frontend development skill (standalone, works in any agent)
- 7 split review agents + verification agent + drift scanner (was 1 unified reviewer)
- OfficeCLI MCP for document generation

### Designed but Not Built
- Multi-runtime distribution — Cursor, Codex, Gemini, Pi, OpenCode, Windsurf (plan in `multi-runtime-plan.md`)
- Pi plugin — fully specified in `taskplex-pi-plugin.md`
- Board architecture — designed in `board-architecture.md`
- Memplex HTTP API — needed for Pi integration (no MCP in Pi)
- Skill performance tracking — deferred until multi-agent is default
- Issue relations/dependencies — deferred, partially covered by prd-state.json
