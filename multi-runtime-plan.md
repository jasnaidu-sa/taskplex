# TaskPlex Multi-Runtime Distribution Plan

> **Revised**: April 2026. Original plan (thin SKILL.md + adapters/) replaced with native packaging per runtime. Every major runtime now has a plugin/extension system that bundles skills + hooks + agents + MCP.

## Goal

Distribute TaskPlex as **native packages for each runtime** — not a generic skill with adapter shims. Each runtime gets a first-class plugin that feels native, uses the runtime's primitives, and distributes through the runtime's marketplace.

## Target Runtimes (April 2026)

| Runtime | Package Type | Skills | Hooks | Subagents | MCP | Marketplace | Priority |
|---------|-------------|:---:|:---:|:---:|:---:|:---:|---|
| **Claude Code** | Manual (hooks + commands) | Yes | 9 events | Agent tool | Yes | No | Already built |
| **Cursor 3** | Marketplace plugin | Yes | Via plugins | Async, nested | Yes | Yes (30+ plugins) | **High** |
| **Codex CLI** | Plugin + marketplace | Yes | 5 events | Multi-agent v2 | Yes | Yes | **High** |
| **Gemini CLI** | Extension package | Yes | BeforeTool+ask | Multi-registry | Yes | Extensions | **High** |
| **OpenCode** | TypeScript plugin (npm) | Yes | 25+ events | Custom agents | Yes | npm | Medium |
| **Windsurf** | Manual (hooks.json + rules) | Yes | 12 events | No | Yes | No | Medium |
| **Antigravity** | MCP + skills | Yes | Unknown | Yes (Jetski) | Yes | MCP hub | Low |
| **Aider** | CONVENTIONS.md only | Partial | No | No | No | No | Low |

## Architecture

### Core Package (shared, runtime-agnostic)

```
taskplex-core/
├── phases/
│   ├── init.md              # Phase 0: initialization + design
│   ├── planning.md          # Phase 1: planning + implementation
│   ├── qa.md                # Phase 4.5: product-type QA
│   ├── validation.md        # Phase 5: validation + completion
│   ├── bootstrap.md         # Phase -1/-0.5: INTENT.md + conventions
│   └─��� prd.md               # Initiative mode
├── contracts/
│   ├── policy.json          # Quality profiles, limits, agent lists
│   ├── gates.md             # Gate catalog
│   ├── artifact-contract.md # Required artifacts by profile
│   ├── manifest-schema.json # Manifest JSON schema
│   ├── handoff-contract.md  # Agent-to-agent transitions
│   ├── hardening-checks.md  # Hardening check registry
│   └── skill-evolution.md   # Skill evolution spec
├── agents/
│   ├── architect.md         # Opus-level design
│   ├── planning-agent.md    # Spec writing
│   ├── implementation-agent.md
│   ├── security-reviewer.md
│   ├── closure-agent.md
│   ├── code-reviewer.md
│   ├── hardening-reviewer.md
│   ├── database-reviewer.md
│   ├── e2e-reviewer.md
│   ├── user-workflow-reviewer.md
│   ├── compliance-agent.md
│   ├── researcher.md
│   ├── merge-resolver.md
│   ├── bootstrap.md
│   ├── prd-bootstrap.md
│   ├── strategic-critic.md
│   ├── tactical-critic.md
│   ├── build-fixer.md
│   └── drift-scanner.md
└── skills/
    ├── frontend/            # Standalone frontend development
    ├── evaluate/            # Product evaluation
    └── plan/                # Strategic planning
```

All internal references use `$TASKPLEX_HOME/` paths. Each runtime package resolves this to its native path.

### Per-Runtime Native Packages

```
taskplex-cursor/             # Cursor 3 marketplace plugin
├── cursor-plugin.json       # Plugin manifest
├── skills/                  # SKILL.md files (phase entry points)
├── hooks/                   # Cursor hook definitions
├── subagents/               # Subagent configs per execution route
├── mcp/                     # MCP server configs (playwright, memplex)
├── rules/                   # .mdc rules for workflow guidance
└── core/ → symlink/copy     # taskplex-core content

taskplex-codex/              # Codex CLI plugin
├── agents/openai.yaml       # Plugin metadata + icon + policy
├── skills/                  # SKILL.md files
├── hooks.json               # Hook definitions (5 events)
├── agents/                  # Multi-agent v2 configs
├── mcp/                     # MCP server configs
└── core/ → taskplex-core

taskplex-gemini/             # Gemini CLI extension
├── gemini-extension.json    # Extension manifest
├── commands/                # TOML command files (/taskplex, /plan, /drift)
├── hooks/hooks.json         # Hook definitions (BeforeTool with ask)
��── skills/                  # SKILL.md files
├── agents/                  # Subagent .md files with tool isolation
├── policies/                # Quality profile policy files (tier 2)
└── core/ → taskplex-core

taskplex-opencode/           # OpenCode npm plugin
├── package.json             # npm package with @opencode-ai/plugin dep
├── index.ts                 # Plugin entry point (full state machine)
├── hooks/                   # 25+ event handlers
├── tools/                   # Custom tool definitions
├── agents/                  # Agent YAML configs
├── skills/                  # SKILL.md files
└── core/ → taskplex-core

taskplex-claude/             # Claude Code (current implementation)
├── hooks/                   # 9 .mjs hook files
├── commands/                # taskplex.md, tp.md, plan.md, solidify.md, drift.md
├── settings-fragment.json   # Hook wiring for settings.json
└── core/ → taskplex-core

taskplex-windsurf/           # Windsurf (manual install)
├─�� hooks.json               # 12 event types
├── rules/                   # .md rule files
├── skills/                  # SKILL.md files
├── install.sh               # Copies to ~/.windsurf/
└── core/ → taskplex-core
```

## Runtime-Specific Implementation Notes

### Cursor 3 (highest priority)

**Why first**: Largest user base, most capable platform (async subagents, /worktree, marketplace).

**Mapping**:
| TaskPlex Concept | Cursor Primitive |
|-----------------|-----------------|
| `/tp` command | Plugin-defined skill trigger |
| Design gate hook | Plugin hook (PreToolUse equivalent) |
| Pre-commit hook | Plugin hook |
| Agent spawning | Cursor subagents (async, nested, own context + model) |
| Worktree isolation | `/worktree` command (native!) |
| Session recovery | Plugin state + memory tool |
| Memplex integration | MCP server (universal) |
| Playwright | MCP server (`@playwright/mcp@latest`) |
| Execution continuity | Subagent model (runs to completion by default) |
| Automations | Event-driven cloud agents for CI/CD integration (future) |

**Key advantage**: Cursor's `/worktree` replaces our manual git worktree management. Subagents are async by default — they don't block the orchestrator. This naturally enforces the execution continuity rule.

**Distribution**: Cursor marketplace plugin → one-click install.

### Codex CLI

**Mapping**:
| TaskPlex Concept | Codex Primitive |
|-----------------|----------------|
| `/tp` command | Slash command via plugin |
| Design gate | PreToolUse hook (Bash-only currently — limitation) |
| Agent spawning | Multi-Agent v2 with path-based addresses |
| Handoff contract | Structured inter-agent messaging |
| UserPromptSubmit | Prompt augmentation for phase context |
| MCP | Plugin-bundled servers |

**Key limitation**: PreToolUse only intercepts Bash tool currently — can't block Edit/Write. Design gate enforcement would need a workaround (UserPromptSubmit augmentation).

**Distribution**: Codex plugin marketplace.

### Gemini CLI

**Why architecturally best**: Extension format bundles everything TaskPlex needs — commands, hooks, skills, agents, policies, themes. The `BeforeTool` hook with `ask` decision maps perfectly to the design gate's approval workflow.

**Mapping**:
| TaskPlex Concept | Gemini Primitive |
|-----------------|-----------------|
| `/tp` command | TOML command file |
| Design gate | BeforeTool hook with `ask` decision |
| Agent spawning | Multi-registry subagents with tool isolation |
| Quality profiles | Policy files (tier 2 governance) |
| Enterprise compliance | Write-protected governance files |
| Worktree isolation | Native sandbox + worktree support |

**Key advantage**: Write-protected governance files mean enterprise quality profiles can't be overridden by the agent. This is stronger enforcement than Claude Code (where the LLM could theoretically modify policy.json).

**Distribution**: `gemini extensions install taskplex`.

### OpenCode

**Why most programmable**: 25+ hook events, custom tool SDK, in-process state. The plugin IS the state machine — no need for file-based manifest reads on every hook fire.

**Mapping**:
| TaskPlex Concept | OpenCode Primitive |
|-----------------|-------------------|
| Manifest | In-process state object (plugin retains state across events) |
| Hooks | `tool.execute.before/after`, `file.edited`, `session.*`, etc. |
| Agent spawning | Custom agent definitions + Bun shell for subprocesses |
| Custom tools | Plugin SDK `defineTool` — manifest management as native tools |

**Key advantage**: The plugin can maintain the manifest in memory AND on disk — hybrid approach with instant state access and disk persistence. No 5-10ms file read on every hook fire.

**Distribution**: npm package (`npm install taskplex-opencode`).

### Windsurf

**Most granular hooks** but no packaging. Best for users who want maximum enforcement.

**Mapping**:
| TaskPlex Concept | Windsurf Primitive |
|-----------------|-------------------|
| Design gate | `pre_write_code` hook (has edit diffs in context!) |
| Heartbeat | `post_write_code` hook |
| Pre-commit | `pre_run_command` hook (inspect command string) |
| Audit trail | `post_cascade_response_with_transcript` (full JSONL) |
| Worktree | `post_setup_worktree` hook |

**Key advantage**: `pre_write_code` receives the actual edit diff — the hook can inspect what's being written, not just whether a write is happening. Most precise enforcement possible.

**Distribution**: `./install.sh` copies files + patches hooks.json.

### Antigravity

**Lower priority** — Google's platform with its own agent orchestration (Manager View). TaskPlex would work alongside rather than replace it.

**Integration path**: MCP servers (memplex, playwright, officecli) work via Antigravity's MCP support. Skills work via standard SKILL.md format. Full workflow orchestration would conflict with Manager View.

## MCP Servers (Universal, All Runtimes)

These work identically everywhere MCP is supported:

| MCP Server | Package | Purpose |
|------------|---------|---------|
| Playwright | `@playwright/mcp@latest` | Browser automation, visual QA |
| Memplex | (custom) | Cross-session knowledge |
| OfficeCLI | `officecli mcp` | Document generation |

Each runtime package includes MCP config in its manifest. The servers themselves are runtime-agnostic.

## Phase File Path Abstraction

All phase files and agent definitions use `$TASKPLEX_HOME/` instead of hardcoded paths:

```markdown
Read: $TASKPLEX_HOME/phases/init.md
Spawn agent from: $TASKPLEX_HOME/agents/architect.md
Policy: $TASKPLEX_HOME/contracts/policy.json
```

Each runtime resolves `$TASKPLEX_HOME`:
| Runtime | Resolved Path |
|---------|---------------|
| Claude Code | `~/.claude/taskplex/` |
| Cursor | `.cursor/plugins/taskplex/core/` |
| Codex | `~/.codex/plugins/taskplex/core/` |
| Gemini CLI | `~/.gemini/extensions/taskplex/core/` |
| OpenCode | `node_modules/taskplex-opencode/core/` |
| Windsurf | `~/.windsurf/taskplex/` |

## Hook Enforcement Mapping

The 3 critical hooks (design gate, pre-commit, implementation gate) map to each runtime:

| Hook | Claude Code | Cursor | Codex | Gemini | OpenCode | Windsurf |
|------|-----------|--------|-------|--------|----------|----------|
| Design gate | PreToolUse Edit\|Write | Plugin hook | UserPromptSubmit* | BeforeTool+ask | tool.execute.before | pre_write_code |
| Pre-commit | PreToolUse Bash | Plugin hook | PreToolUse Bash | BeforeTool | tool.execute.before | pre_run_command |
| Impl gate | PreToolUse Edit\|Write | Plugin hook | UserPromptSubmit* | BeforeTool | tool.execute.before | pre_write_code |

\* Codex PreToolUse is Bash-only — workaround via prompt augmentation.

## Model Tier Mapping

| Tier | Claude Code | Cursor 3 | Codex | Gemini CLI | OpenCode |
|------|-----------|----------|-------|------------|----------|
| High (architect) | opus | o3/opus | o3 | gemini-3-pro | strongest |
| Standard (impl) | sonnet | sonnet/gpt-4.1 | gpt-4.1 | gemini-2.5-flash | default |
| Fast (closure) | haiku | haiku/gpt-4.1-mini | gpt-4.1-mini | gemini-2.5-flash | fastest |

## Implementation Phases

### Phase 1: Extract Core (1-2 days)
- Create `taskplex-core/` from existing `~/.claude/taskplex/` + agents + skills
- Replace hardcoded `~/.claude/` paths with `$TASKPLEX_HOME/`
- Verify Claude Code still works with resolved paths
- Publish core to GitHub or npm

### Phase 2: Cursor 3 Plugin (3-5 days)
- Create plugin manifest + SKILL.md entry points
- Map hooks to Cursor's plugin hook system
- Map agent spawning to Cursor subagents
- Map worktree isolation to `/worktree`
- Configure MCP servers (playwright, memplex)
- Test end-to-end: `/tp` → design → planning → implementation → validation
- Submit to Cursor marketplace

### Phase 3: Codex CLI Plugin (2-3 days)
- Create plugin with agents/openai.yaml metadata
- Map hooks (5 events — workaround for PreToolUse Bash-only)
- Map agents to Multi-Agent v2
- Configure MCP servers
- Publish to Codex plugin marketplace

### Phase 4: Gemini CLI Extension (2-3 days)
- Create extension with gemini-extension.json manifest
- TOML command files for /taskplex, /plan, /drift, /solidify
- BeforeTool hooks with `ask` for design gate
- Subagent configs with tool isolation
- Policy files for quality profiles
- Test and publish

### Phase 5: OpenCode Plugin (3-5 days)
- TypeScript plugin with @opencode-ai/plugin SDK
- Full state machine implementation (25+ hooks)
- Custom tools for manifest management
- In-process + disk hybrid state
- npm publish

### Phase 6: Windsurf + Antigravity (1-2 days each)
- Windsurf: hooks.json + rules + install script
- Antigravity: MCP + skills (minimal, leverages existing MCP support)

## Companion Skills Distribution

These skills are already SKILL.md format and work across all runtimes:

| Skill | Status | Notes |
|-------|--------|-------|
| `frontend/` | Ready | Design system, a11y, responsive, component spec |
| `evaluate/` | Ready | Audit + review modes |
| `plan/` | Needs update | Currently references Claude Code commands |
| `officecli/` | Ready | MCP-based, universal |

Each runtime package bundles these skills alongside the core workflow.

## Revenue Model Consideration

| Component | Free | Paid (via memplex) |
|-----------|------|-------------------|
| TaskPlex workflow | Yes — all phases, hooks, gates | — |
| Companion skills | Yes — frontend, evaluate, plan | — |
| Memplex integration | — | Cross-session knowledge, error resolutions, file coupling |
| OfficeCLI | Free | — |
| Playwright MCP | Free | — |

TaskPlex is free. Memplex is the upsell. The cross-runtime distribution expands the memplex addressable market — every TaskPlex user on any runtime sees the "cross-session persistence requires memplex" note.

## Open Questions

1. **Should taskplex-core be an npm package or just a GitHub repo?** npm gives dependency management. GitHub gives manual control. Could be both.

2. **Should each runtime package include the full core or reference it?** Bundling is simpler (no dependency). Referencing saves space and enables shared updates. Cursor marketplace may require bundling.

3. **How do we handle runtime-specific features?** Blueprint worktrees work natively in Cursor (best), via sandbox in Gemini, not at all in Codex/Windsurf. Graceful degradation: sequential execution as fallback.

4. **Should we build a universal installer?** `npx taskplex install` → detects runtimes → installs appropriate packages. Nice UX but adds a build step.
