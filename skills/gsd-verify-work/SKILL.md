---
name: gsd-verify-work
description: >
  Verify a phase's work by performing runtime testing with Chrome DevTools MCP.
  Navigates the running application, takes DOM snapshots, interacts with UI
  elements, and checks console/network output using the snapshot-interact-verify
  pattern. Use when user says "verify work", "test phase", "/gsd:verify-work",
  or after execute-phase completes.
argument-hint: "<phase-number>"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - mcp__chrome-devtools__take_snapshot
  - mcp__chrome-devtools__take_screenshot
  - mcp__chrome-devtools__navigate_page
  - mcp__chrome-devtools__list_pages
  - mcp__chrome-devtools__select_page
  - mcp__chrome-devtools__list_console_messages
  - mcp__chrome-devtools__get_console_message
  - mcp__chrome-devtools__list_network_requests
  - mcp__chrome-devtools__get_network_request
  - mcp__chrome-devtools__evaluate_script
  - mcp__chrome-devtools__performance_start_trace
  - mcp__chrome-devtools__performance_stop_trace
  - mcp__chrome-devtools__performance_analyze_insight
  - mcp__chrome-devtools__click
  - mcp__chrome-devtools__fill
  - mcp__chrome-devtools__hover
  - mcp__chrome-devtools__press_key
  - mcp__chrome-devtools__emulate
  - mcp__chrome-devtools__wait_for
---

# Verify Work

You orchestrate runtime verification using Chrome DevTools MCP. You spawn a browser-tester agent -- you do NOT test the app yourself.

Orchestrator stays lean. The browser-tester agent gets fresh 200k context for testing.

## Step 1: Initialize

```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.js init phase-op "$PHASE_ARG")
```

Parse: `phase_dir`, `phase_number`, `phase_name`, `padded_phase`.

- If phase not found: Error -- phase directory does not exist.

## Step 2: Load Phase Context

Read SUMMARY.md files from phase directory to understand what was built:

```bash
ls "$PHASE_DIR"/*-SUMMARY.md 2>/dev/null
```

- What features were implemented
- What files were created/modified
- What URLs/routes should be testable

Read PLAN.md files for `must_haves` (expected behaviors to verify):

```bash
ls "$PHASE_DIR"/*-PLAN.md 2>/dev/null
```

## Step 3: Pre-flight Check

Attempt `mcp__chrome-devtools__list_pages` to verify Chrome DevTools MCP is connected.

- **If unavailable:** Report "Chrome DevTools MCP not connected -- runtime verification skipped." Set status to "inconclusive". Offer user: skip or retry.
- **If available:** Proceed to spawn browser tester.

## Step 4: Spawn Browser Tester

```
Task(
  prompt="First, read ~/.claude/skills/gsd-verify-work/references/agents/gsd-browser-tester.md for your role.

  <objective>
  Verify phase {phase_number}-{phase_name} runtime behavior using Chrome DevTools.
  Test features built in this phase using the snapshot-interact-verify cycle.
  Phase directory: {phase_dir}
  </objective>

  <files_to_read>
  - {phase_dir}/*-SUMMARY.md (all summaries -- understand what was built)
  - {phase_dir}/*-PLAN.md (all plans -- understand expected behaviors)
  - .planning/ROADMAP.md (phase goal)
  </files_to_read>

  <output>Create {phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md</output>",
  subagent_type="general-purpose"
)
```

## Step 5: Handle Result

Read status from RUNTIME-VERIFICATION.md frontmatter:

- **passed:** Display summary. Runtime verification complete.
- **issues_found:** Display issues. Offer re-execution or proceed to review.
- **inconclusive:** Chrome DevTools unavailable or app not running. Offer skip.

## Step 6: Present Results and Offer Next

Display verification summary table:

```
## Runtime Verification Summary

| Metric              | Value              |
| ------------------- | ------------------ |
| Status              | {status}           |
| Tests passed        | {passed}/{total}   |
| Console errors      | {count}            |
| Network issues      | {count}            |
| Performance issues  | {count}            |
```

Offer `/gsd:review {phase}` as the next step in the pipeline.
