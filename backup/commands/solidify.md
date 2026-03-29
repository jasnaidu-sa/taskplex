# /solidify — Merge Skill Evolutions into Source

**Command**: `/solidify [skill-name]` or `/solidify --all` or `/solidify --list`

Merges approved skill evolutions from `evolutions.json` into the skill's source file (`skill.md` or `SKILL.md`). Evolutions are improvements learned from past task failures and corrections — they sit in a staged file until explicitly merged.

## Usage

```
/solidify taskplex       # Review and merge evolutions for the taskplex skill
/solidify evaluate       # Review and merge evolutions for the evaluate skill
/solidify --all          # Review evolutions across all skills
/solidify --list         # Show all pending evolutions without merging
```

## Process

### Step 1: Find Evolutions

If a skill name is provided:
- Read `~/.claude/skills/{skill}/evolutions.json`
- If not found, check `~/.claude/skills/{skill}/SKILL.md` parent directory
- If no evolutions.json exists: "No evolutions found for {skill}."

If `--all`:
- Scan all `~/.claude/skills/*/evolutions.json` files
- Collect active evolutions across all skills

If `--list`:
- Same scan as `--all` but only display, don't merge

### Step 2: Present Each Evolution

For each evolution with `status: "active"`:

> **Evolution**: {title}
> **Type**: {troubleshooting | example}
> **Learned from**: Task {taskId} on {date}
> **Trigger**: {when this applies}
>
> **Content**:
> {the improvement text}
>
> 1. **Keep** — merge into SKILL.md
> 2. **Edit** — modify before merging
> 3. **Discard** — remove this evolution
> 4. **Skip** — leave for later

### Step 3: Apply Decisions

**For "Keep":**
- Read the skill's source file (`skill.md` or `SKILL.md`)
- Determine the appropriate section:
  - Troubleshooting evolutions → append to `## Troubleshooting` or `## Known Issues` section (create if missing)
  - Example evolutions → append to `## Context Notes` or `## Learned Patterns` section (create if missing)
- Write the evolution content as a bullet point with the trigger as context
- Set evolution `status: "solidified"` in evolutions.json
- Record `solidifiedAt` timestamp

**For "Edit":**
- Ask the user for the modified text
- Apply same merge logic as "Keep" with the edited content

**For "Discard":**
- Set evolution `status: "discarded"` in evolutions.json
- Record `discardedAt` timestamp and reason (if user provides one)

**For "Skip":**
- Leave `status: "active"` — will be presented again next time

### Step 4: Summary

After all evolutions processed:

> **Solidify complete for {skill}:**
> - {N} merged into SKILL.md
> - {M} discarded
> - {P} skipped (still active)

## Format for Merged Evolutions

When appending to SKILL.md:

```markdown
## Troubleshooting

- **TypeScript path alias resolution in monorepo** (learned 2026-03-29)
  When: target project has tsconfig.json with paths field
  Always check tsconfig.json paths before assuming import structure.
  Build-fixer spent 3 rounds on path resolution errors that could have been avoided.

## Learned Patterns

- **User means 'accessible' not 'pretty'** (learned 2026-03-29)
  When: task description mentions UI improvement or UX
  Probe whether they mean accessibility (keyboard nav, screen readers, contrast)
  or aesthetics (colors, spacing, animation). In this project, user consistently meant accessibility.
```

## Rules

- **Never auto-solidify** — always require explicit user review via this command
- **Never delete evolutions** — mark as "solidified" or "discarded" for audit trail
- **Never modify evolutions.json entries** — append new status, don't overwrite
- **One skill at a time** (unless `--all`) — prevents cross-contamination
- **Preserve evolution source** — the `source` field stays in evolutions.json even after solidification, linking back to the task that generated it
