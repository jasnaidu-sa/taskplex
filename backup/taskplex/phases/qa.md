# taskplex: QA Phase (Phase 4.5)
<!-- Loaded by orchestrator after implementation, before validation. Self-contained. -->
<!-- v1: Product-type-aware QA with brief-driven journey testing -->

**The QA phase runs after implementation and before validation.** It answers: "does this actually work when you use it?" — not "does the code pass checks?" (that's validation's job).

**CRITICAL — Phase transition**: Immediately update manifest.json:
```json
manifest.phase = "qa"
```
Write manifest to disk NOW.

---

## Step 4.5.1: Detect Product Type & QA Strategy

Read `manifest.json` for project type, modified files, and task description.

**Determine QA approach:**

| Product Type | QA Method | How to detect |
|-------------|-----------|---------------|
| UI App (web) | Browser walkthrough | Routes, components, CSS in modified files; URL available |
| UI App (desktop) | Local launch + browser | Tauri/Electron in deps; desktop entry point |
| CLI | Run commands | CLI entry point in modified files; console_scripts / bin in package |
| API / Service | Call endpoints | Route handlers, controllers in modified files |
| Library / Module | **Skip QA** | Only exports/types changed; no runnable surface |
| Infrastructure | Health check | Pipeline/job configs, workers in modified files |

**Skip conditions** (log to `manifest.degradations` if skipping):
- `--skip-qa` flag → skip, log degradation
- Library/module with no runnable surface → skip, not a degradation
- No user-facing changes detected (pure refactor) → skip, not a degradation
- Product type undetectable → ask user: "Does this have a runnable surface I should test?"

**If skipping**: Set `manifest.qa.status = "skipped"`, update checklist, proceed to validation.

**Brief check**: Look for `product/brief.md` or `.claude-task/{taskId}/product/brief.md`. If found, extract core user journeys — these become QA test cases in Step 4.5.3. Record `manifest.qa.briefUsed = true`.

Initialize manifest QA fields:
```json
"qa": {
  "status": "in-progress",
  "method": "browser" | "cli" | "api" | "health-check" | "skipped",
  "briefUsed": false,
  "journeysTested": 0,
  "journeysPassed": 0,
  "edgeCasesTested": 0,
  "bugsFound": 0,
  "bugsFixed": 0,
  "fixRoundsUsed": 0,
  "unresolvedIssues": []
}
```

Update checklist: mark 4.5.1 complete.

---

## Step 4.5.2: Smoke Test

Before walking journeys, verify the basic thing works at all. This catches "it doesn't even start" before wasting time on detailed testing.

### By product type:

**UI App (web):**

Detect browser tool: prefer Playwright MCP (`mcp__playwright__*`), fallback to `agent-browser` CLI, skip if neither available.

```
# With Playwright MCP (preferred):
mcp__playwright__browser_navigate → {url}
mcp__playwright__browser_screenshot → .claude-task/{taskId}/screenshots/smoke-test.png
mcp__playwright__browser_console → check for errors

# With agent-browser (fallback):
agent-browser open {url}
agent-browser snapshot
agent-browser console
```
- Pass: Main view renders, no critical console errors
- Fail: Blank screen, crash, or error page

**CLI:**
```bash
{command} --help                # Does it respond?
echo $?                         # Exit code 0?
{command} {simplest-valid-args} # Does the primary command work?
echo $?
```
- Pass: Help outputs, primary command produces expected output
- Fail: Crash, traceback, or no output

**API / Service:**
```bash
# Start the server if not running
curl -s {base-url}/health       # Health endpoint?
curl -s -o /dev/null -w "%{http_code}" {base-url}/{primary-endpoint}
```
- Pass: Health responds, primary endpoint returns valid response
- Fail: Connection refused, 500, or no response

**Infrastructure:**
```bash
# Check if the service/pipeline is running
# Check logs for errors
# Verify basic health metrics
```

### If smoke test fails:
Report immediately. This counts as bug fix round 1 if a fix is attempted. Do NOT proceed to journey walkthrough until the smoke test passes or the user decides to skip.

Update manifest: `manifest.qa.smokeTest = "pass" | "fail"`
Update checklist: mark 4.5.2 complete.

---

## Step 4.5.3: Journey Walkthrough

This is the core of QA — walk the user's actual experience end to end.

### If `product/brief.md` exists:

Extract the core user journeys from the brief. For each journey:

1. Read the journey's steps (DOES / SEES / FEELS or DOES / GETS / FRICTION)
2. Execute each step against the live product
3. At each step, verify:
   - **Functional**: Does the step work? Does the right thing happen?
   - **Feedback**: Is there appropriate feedback? (loading states, confirmations, error messages)
   - **Design implication**: Does it meet the constraint from the brief's design implication column?

### If no brief exists:

Infer core journeys from the task description and modified files:
- What was the user trying to build? That's the happy path journey.
- What's the most common variation? Test that too.
- Aim for 2-3 journeys max.

### Journey execution by product type:

**UI App:**

Use Playwright MCP (preferred) or agent-browser (fallback):
```
# With Playwright MCP:
mcp__playwright__browser_navigate → {url}
# For each step in the journey:
mcp__playwright__browser_screenshot → capture before state
mcp__playwright__browser_click → take the action
mcp__playwright__browser_screenshot → capture after state
mcp__playwright__browser_console → check for errors

# With agent-browser (fallback):
agent-browser open {url}
agent-browser snapshot / agent-browser click @{element} / agent-browser console
```

Store screenshots at `.claude-task/{taskId}/screenshots/journey-{name}-step{N}.png`

**CLI:**
```bash
# For each step in the journey:
{command} {args-for-this-step}            # Execute the step
# Check: stdout has expected content?
# Check: exit code correct?
# Check: side effects occurred? (files created, data changed)
```

**API:**
```bash
# For each step in the journey:
curl -X {METHOD} {endpoint} \
  -H "Content-Type: application/json" \
  -d '{request body}'
# Check: response status code correct?
# Check: response body has expected shape?
# Check: side effects occurred? (DB state, events fired)
```

### Record results per journey:

```markdown
### Journey: {name}
**Source**: brief | inferred
**Status**: Pass | Partial | Fail
**Steps completed**: {n}/{total}
**Broke at**: {step description, if failed}
**Findings**:
- {what happened vs what should have happened}
```

Update manifest: increment `journeysTested`, `journeysPassed`.
Update checklist: mark 4.5.3 complete.

---

## Step 4.5.4: Adversarial Verification (MANDATORY — spawns verification agent)

Spawn the verification agent to actively try to break the implementation. This is not inline spot-checking — it's a dedicated adversarial agent that runs commands, tests endpoints, and probes boundaries. It cannot edit files.

> Spawn verification-agent (sonnet) from $TASKPLEX_HOME/agents/core/verification-agent.md
  Context: spec.md, brief.md, manifest.modifiedFiles, manifest.buildCommands, QA method (UI/CLI/API)
  Writes: .claude-task/{taskId}/reviews/verification.md
  Returns: "PASS: N checks, P probes, 0 failures" or "FAIL: Q bugs found"

The verification agent:
- Runs happy path checks (must execute commands, not read code)
- Runs at least 3 adversarial probes (boundary, concurrency, idempotency, injection, etc.)
- Produces evidence in command+output format (code reading rejected)
- Has anti-rationalization prompts to prevent skip-and-pass behavior

**If PASS**: Proceed to Step 4.5.5 (bug triage — which may have 0 bugs).
**If FAIL**: Bugs feed into Step 4.5.5 bug triage loop. The verification agent's findings are treated as blocker/major severity.

Update manifest: `manifest.qa.adversarialProbes = N`, `manifest.qa.adversarialFailures = M`.
Update checklist: mark 4.5.4 complete.

---

## Step 4.5.5: Bug Triage & Fix Loop

If bugs were found in steps 4.5.2-4.5.4, triage and fix them.

### Triage by severity:

| Severity | Description | Action |
|----------|-------------|--------|
| **Blocker** | Core journey doesn't work at all | Fix immediately |
| **Major** | Journey works but with significant UX issue (missing feedback, wrong output, broken error handling) | Fix if within round budget |
| **Minor** | Cosmetic, edge case polish, non-critical formatting | Log for later, do NOT fix in QA |

### Fix loop rules:

1. **Max 3 fix rounds.** After 3, stop and report remaining issues.
2. **Memplex error check** (if `manifest.memplexAvailable`): Before each fix attempt, call `get_error_resolution` for the bug's error pattern. If a known resolution exists, apply it directly — this may resolve the bug without a full debugging cycle. Store results in `manifest.memplexContext.qa`.
3. Each round:
   a. Pick the highest-severity unfixed bug
   b. Check for known resolution (step 2) — if found, apply directly
   c. If no known resolution: fix it (edit the relevant files)
   d. Re-test the affected journey to verify the fix
   e. Quick-check that the fix didn't break other journeys (regression)
3. If a fix fails (introduces new bugs or doesn't resolve the issue):
   a. Revert the fix
   b. Log as unresolved: `manifest.qa.unresolvedIssues.push({description})`
   c. This still counts as a round
4. Minor bugs go straight to `unresolvedIssues` without attempting a fix

### Why cap at 3?

If it can't be fixed in 3 rounds, the issue is likely architectural — it needs to go back to planning, not more patching in QA. The cap prevents QA from becoming an open-ended debugging session.

Update manifest after each round: increment `fixRoundsUsed`, `bugsFixed`.
Update checklist: mark 4.5.5 complete with round count.

---

## Step 4.5.6: QA Report

Write `.claude-task/{taskId}/qa-report.md`:

```markdown
# QA Report: {task description}

**Date**: {date}
**Product type**: {UI App | CLI | API | Infrastructure}
**QA method**: {browser walkthrough | command execution | endpoint testing | health check}
**Brief used**: {yes — path | no — journeys inferred}

## Smoke Test
**Status**: Pass | Fail
**Notes**: {any issues encountered}

## Journey Results

| Journey | Source | Status | Steps | Notes |
|---------|--------|--------|-------|-------|
| {name} | brief/inferred | Pass/Partial/Fail | {n}/{total} | {summary} |

## Edge Cases

| Case | Result | Notes |
|------|--------|-------|
| {scenario} | Pass/Fail | {what happened} |

## Bugs Found & Fixed

| # | Bug | Severity | Status | Fix |
|---|-----|----------|--------|-----|
| 1 | {description} | Blocker/Major/Minor | Fixed/Unresolved/Deferred | {what was done} |

## Fix Rounds: {n}/3

## Unresolved Issues
{Numbered list of bugs not fixed — these carry into validation as known issues.
The code review agent in validation should be aware of these.}

## QA Verdict
**{Pass | Partial | Fail}**
{One sentence: "All core journeys work" | "Core works, edges have issues" | "Core journey broken — {which one}"}
```

### Present in terminal:

Display the QA verdict and journey table. Don't dump the full report — keep it concise:

```
QA Complete: {Pass|Partial|Fail}
  Journeys: {passed}/{tested} passed
  Edge cases: {passed}/{tested} passed
  Bugs: {found} found, {fixed} fixed, {unresolved} unresolved
  Fix rounds: {used}/3
  {If unresolved: "Unresolved: {brief descriptions}"}
```

### Gate behavior:

QA is **not a hard gate** — validation runs regardless. But:

- **QA Pass**: Proceed to validation normally.
- **QA Partial**: Proceed with warning. Unresolved issues noted in manifest for code reviewer.
- **QA Fail**: Ask user before proceeding:
  > "QA found that a core journey doesn't work: {description}.
  > The code review will flag this. Want to fix the blocker first, or continue to validation?"
  - **Fix first** → goes back to implementation for targeted fix, then re-runs QA (does NOT count as a new fix round — this is a full re-entry)
  - **Continue** → proceed to validation with QA fail logged

### Final manifest update:

```json
"qa": {
  "status": "pass" | "partial" | "fail",
  "method": "browser" | "cli" | "api" | "health-check",
  "briefUsed": true | false,
  "smokeTest": "pass" | "fail",
  "journeysTested": 3,
  "journeysPassed": 2,
  "edgeCasesTested": 5,
  "bugsFound": 4,
  "bugsFixed": 3,
  "fixRoundsUsed": 2,
  "unresolvedIssues": ["description..."],
  "reportPath": ".claude-task/{taskId}/qa-report.md"
}
```

Update checklist: mark 4.5.6 complete.
Proceed to Phase 5: Validation.
