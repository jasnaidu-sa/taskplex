# TaskPlex Portability Guide

This document separates the `/taskplex` workflow into **core workflow logic** (agent-agnostic) and **adapter-specific runtime behavior** (Claude Code-specific). Future adapter implementations for other coding agents (Pi, Codex, Gemini CLI, Cursor, etc.) should implement the adapter layer while preserving core workflow semantics.

**Scope**: This is a descriptive foundation — it catalogs what is core vs adapter-specific and identifies portability boundaries. It is not a formal adapter interface specification or an API contract. Actual adapter implementations will need to define their own concrete interfaces based on this guide.

---

## Architecture Overview

```
┌──────────────────────────────────────────┐
│           TaskPlex Core Workflow         │
│  (phase dispatch, gates, contracts,      │
│   artifact ownership, quality profiles)  │
├──────────────────────────────────────────┤
│            Adapter Layer                 │
│  (agent spawning, hooks, user prompts,   │
│   model selection, path resolution,      │
│   context management, session bridge)    │
├──────────────────────────────────────────┤
│          Runtime Environment             │
│  (Claude Code, Pi, Codex, Cursor, etc.)  │
└──────────────────────────────────────────┘
```

---

## Core Workflow (Agent-Agnostic)

These components define *what* TaskPlex does. Any adapter must preserve these semantics.

### Phase System
- **Phases**: init → brief → analysis → planning → execution → validation → hardening → completion
- **Phase transitions**: Explicit manifest.phase updates, emitted as trace events
- **Route selection**: Light (minimal + single agent), Standard (multi-agent + tactical critic), Blueprint (opus architect + critics + multi-agent + waves) — user-selected via flags or menu

### Quality Profiles
- **lean**: Minimal validation, no extra artifacts
- **standard**: + traceability, degradations, convention compliance, hardening warnings
- **enterprise**: + dependency audit, migration safety, operability, readiness verdict, hardening blocks

### Gate System
- Gate catalog: `~/.claude/taskplex/gates.md`
- Execution order: Steps 0-9 with conditional and enterprise-only gates
- Verdict enums: Per-gate (PASS/FAIL/WARN/APPROVED/etc.)
- Gate decision logging to `gate-decisions.json`

### Artifact Contract
- Directory structure: `.claude-task/{taskId}/`
- Required artifacts by profile: `~/.claude/taskplex/artifact-contract.md`
- Manifest schema: `~/.claude/taskplex/manifest-schema.json` (schemaVersion: 1)

### Validation Pipeline
- Build validation (typecheck, lint, test)
- Security review, closure check, code review
- Conditional gates (database, e2e, user-workflow)
- Enterprise gates (dependency compliance, migration safety, operability)
- Custom gates from conventions.json
- Compliance as final gate

### Hardening
- Two-level approach: Level 1 (automated) + Level 2 (human checklist)
- Check catalog: `~/.claude/taskplex/hardening-checks.md`
- Readiness scorecard with weighted categories
- Red-line rules with profile-specific blocking

### Quality Governance
- Degradation logging (never-silent deviations)
- Traceability mapping (requirement → code → test)
- Iteration limits (build-fix, review resubmission, pipeline runs)
- Plan lifecycle validation (mandatory user acknowledgment)

---

## Adapter Layer (Claude Code-Specific)

These are the runtime-specific implementations that an adapter must provide.

### A1: Path Resolution

| Concept | Claude Code Path | Adapter Must Provide |
|---------|-----------------|---------------------|
| Workflow commands | `~/.claude/commands/taskplex.md` + `~/.claude/taskplex/` | Equivalent command/skill location |
| Agent definitions | `~/.claude/agents/core/{agent}.md` | Agent prompt templates |
| Hook scripts | `~/.claude/hooks/taskplex-*.mjs` | Event handlers |
| Session files | `~/.claude/sessions/sess-{pid}.json` | Session bridge (or omit) |
| Schemas | `~/.claude/schemas/*.json` | Validation schemas |
| Plans | `~/.claude/plans/` | Plan storage |
| Extension defs | `.claude-project/` | Project-local extensions |

**Recommendation**: Introduce a path resolver abstraction. All workflow docs reference logical paths (`$TASKFORGE_HOME/agents/security-reviewer.md`) rather than hardcoded `~/.claude/` paths.

### A2: Agent Spawning

Claude Code uses the `Agent` tool with parameters:
- `subagent_type: 'general-purpose'`
- `max_turns`: 15/25/30 based on route
- `isolation: 'worktree'` for git worktree isolation
- `prompt`: Full agent definition + assembled context

**Adapter equivalent**: Any mechanism that can:
1. Launch a sub-task with a system prompt and context
2. Limit execution scope (turns/tokens/time)
3. Optionally isolate file system changes
4. Return a structured result

### A3: Model Selection

| Role | Claude Code Model | Capability Tier |
|------|------------------|-----------------|
| Orchestrator | opus | Tier 1 (highest reasoning) |
| Architect, strategic-critic | opus | Tier 1 |
| Code-reviewer, security-reviewer, build-fixer, implementation-agent | sonnet | Tier 2 (balanced) |
| Closure-agent, compliance-agent, user-workflow | haiku | Tier 3 (fast/cheap) |

**Adapter equivalent**: Map to the target platform's model tiers. The key constraint is that architect and strategic-critic need the strongest available model; commodity agents (closure, compliance) can use the cheapest.

### A4: Hook System

| Hook | Event | Core Behavior |
|------|-------|---------------|
| heartbeat | PostToolUse (Edit/Write) | Update manifest timestamps, track modified files, render progress.md, auto-promote phases |
| pre-compact | PreCompact | Snapshot task state to checkpoint for recovery |
| session-start | SessionStart | Detect in-progress tasks, inject recovery context |
| pre-commit | PreToolUse (git commit) | Block commit if validation not passed |
| stop | Stop | Block stop if validation incomplete |

**Adapter equivalent**: Event-driven hooks, file watchers, middleware, or polling-based alternatives. The heartbeat behavior (manifest updates, progress rendering) is the most critical to replicate.

### A5: User Interaction

Claude Code uses `AskUserQuestion` for:
- Resume detection (Step 4)
- Convention refresh prompts (Step 5)
- PRD mode detection (Step 7a)
- Plan lifecycle validation (Step 7b — MANDATORY)
- Product brief questions (Step 9)
- Hardening result review (Step 8)
- Push/PR confirmation (completion)

**Adapter equivalent**: Any mechanism for structured prompts with named options. The plan lifecycle validation gate (Step 7b) is non-negotiable — the user must always be asked.

### A6: Context Management

Claude Code manages context via:
- Compaction guard (F6): monitors context growth, delegates to subagents
- Pre-compact snapshots: checkpoint state before compaction
- Phase-local loading: only load the active phase doc

**Adapter equivalent**: Most agents have some form of context limit management. The key invariant is that task state survives context resets (manifest.json is the source of truth, not in-memory state).

### A7: Session Bridge (Visualizer)

Claude Code writes `~/.claude/sessions/sess-{pid}.json` for the Agent World Viz app. This includes:
- Phase, route, status
- Agent spawn events
- Trace spans
- Fan-out state for parallel workers

**Adapter equivalent**: Optional. Only needed if a visualization/monitoring tool is connected. The manifest.json already contains most of this data.

### A8: Slash Commands

| Command | Core Behavior |
|---------|---------------|
| `/taskplex` | Entry point with route auto-selection (3 routes: light, standard, blueprint) |
| `/taskplex --light` | Minimal design, single agent, self-review |
| `/taskplex --blueprint` | Architect + critics + multi-agent + waves |
| `/taskplex --plan PLAN-{id}` | Execute a pre-validated plan |
| `/plan` | Standalone planning (produces lifecycle-validated plans) |
| `/harden` | Standalone hardening check |

**Adapter equivalent**: CLI commands, chat commands, or API endpoints that map to the same dispatch logic.

---

## Portability Scorecard

| Aspect | Portability | Notes |
|--------|------------|-------|
| Phase system | High | Pure state machine, no runtime deps |
| Quality profiles | High | Configuration-driven |
| Gate catalog | High | Declarative definitions |
| Artifact contract | High | File-based, any agent can write |
| Manifest schema | High | JSON, schema-validated |
| Hardening checks | High | Tool commands are language-standard |
| Agent spawning | Medium | Requires sub-task mechanism |
| Hook system | Medium | Requires event system or equivalent |
| Model selection | Medium | Requires model tier mapping |
| Context management | Low | Deeply tied to Claude's compaction |
| Session bridge | Low | Specific to Agent World Viz |
| Slash commands | Low | Specific to Claude Code CLI |

---

## Adapter Implementation Checklist

To port TaskPlex to a new coding agent:

1. [ ] **Path resolver**: Map `~/.claude/` to agent-specific config location
2. [ ] **Agent spawning**: Implement sub-task launch with prompt injection and result capture
3. [ ] **Model mapping**: Map sonnet/haiku/opus tiers to available models
4. [ ] **Hook equivalents**: Implement heartbeat (manifest updates) and phase auto-promotion
5. [ ] **User prompts**: Map AskUserQuestion to agent's interaction pattern
6. [ ] **Command registration**: Register entry points for taskplex, plan, harden
7. [ ] **Context recovery**: Ensure manifest.json survives context resets
8. [ ] **Git integration**: Verify git commands work in agent's sandbox
9. [ ] **Session bridge**: Optional — implement if visualization needed
10. [ ] **Convention loading**: Verify CONVENTIONS.md and conventions.json reading works

---

## Contract Conformance Test Plan

Minimal checks to validate that contracts remain consistent after changes:

1. **Manifest fixture**: Validate a representative manifest.json against `taskplex-manifest-schema.json` (JSON Schema validation). Check `schemaVersion: 1` is present and required fields exist.
2. **Gate-decisions fixture**: Validate a sample gate-decisions.json entry against the schema in `taskplex-gates.md` → "Gate Decision Logging". Verify gate names match the gate registry.
3. **Hardening gate-decision fixture**: Validate `hardening/gate-decision.json` structure against `taskplex-hardening-checks.md` → "Readiness Scorecard" categories and red-line rules.
4. **Hook compatibility**: Verify heartbeat hook reads/writes manifest fields that exist in the canonical schema. Verify pre-commit hook checks `validation-gate.json` per the artifact contract.
5. **Viz/adapter compatibility**: Verify `adapter-claude` ManifestJson type covers fields present in the canonical schema. Flag any parallel type definitions that diverge from the schema.
