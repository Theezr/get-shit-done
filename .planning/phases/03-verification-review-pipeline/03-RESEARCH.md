# Phase 3: Verification & Review Pipeline - Research

**Researched:** 2026-02-16
**Domain:** Chrome DevTools MCP runtime testing, IDE diagnostics MCP, code review skill, commit-on-PASS pipeline, SKILL.md architecture
**Confidence:** HIGH

## Summary

Phase 3 creates TWO NEW skills -- `gsd-verify-work` and `gsd-review` -- that complete the four-stage pipeline: plan -> execute -> verify -> review. The verify skill uses Chrome DevTools MCP to perform snapshot-interact-verify runtime testing on the running application and produces a VERIFICATION.md report WITHOUT modifying source files. The review skill performs code quality checks using best-practice skills and IDE diagnostics MCP, produces a REVIEW.md with severity-graded findings, and commits the executor's work ONLY if the review result is PASS.

These skills follow the exact same SKILL.md architecture established in Phases 1 and 2: lean orchestrator body (<1,500 tokens), agent definitions in `references/agents/`, path-based context passing, absolute paths in Task prompts, and explicit `allowed-tools` in frontmatter for role separation. The key architectural insight is that Phase 3 also completes the deferred PIPE-03 requirement: the executor currently commits per-task, but once the review skill exists, the executor should stop committing and the reviewer takes over commit responsibility.

The user already has working reference implementations at `docs/nick/verify.md` and `docs/nick/review.md` that demonstrate the target patterns for both skills. These use the `.pipeline/` directory structure (different from GSD's `.planning/phases/` structure) but the tool usage patterns, Chrome DevTools workflows, IDE diagnostics integration, and commit-on-PASS logic translate directly.

**Primary recommendation:** Create `skills/gsd-verify-work/SKILL.md` and `skills/gsd-review/SKILL.md` as orchestrator skills, each with their own agent definitions in `references/agents/`. The verify skill orchestrator spawns a browser-tester agent; the review skill orchestrator spawns a code-reviewer agent. Update the execute-phase skill to integrate verify and review as post-execution steps (or create a pipeline orchestrator). Modify the executor agent to stop committing per-task (completing PIPE-03).

## Standard Stack

### Core

| Component | Version/Location | Purpose | Why Standard |
|-----------|-----------------|---------|--------------|
| Chrome DevTools MCP | `chrome-devtools-mcp@latest` (npm) | Runtime browser testing -- snapshot, navigate, click, fill, console, network, performance | Official Chrome DevTools team MCP; 26 tools across 6 categories; Puppeteer-based |
| IDE Diagnostics MCP | `mcp__ide__getDiagnostics` (built-in to IDE integrations) | TypeScript error checking via editor's language server | Built-in to Claude Code IDE integrations; accesses live TypeScript diagnostics |
| SKILL.md format | Anthropic spec | Skill definition with YAML frontmatter, markdown body, references/ | Phase 1/2 pattern; progressive disclosure, allowed-tools enforcement |
| Task tool | Claude Code built-in | Spawn sub-agents with fresh 200k context | Established GSD agent spawning pattern |
| gsd-tools.js | `~/.claude/get-shit-done/bin/gsd-tools.js` | Init, state mgmt, commits, artifact verification | All existing skills depend on it |
| Best-practice skills | `~/.claude/skills/*/SKILL.md` | Code quality standards (React, NestJS, OWASP, web-design) | 4 skills already installed; proven in Phase 2 executor |

### Supporting

| Component | Location | Purpose | When to Use |
|-----------|----------|---------|-------------|
| references/agents/ | Inside each skill folder | Agent definitions for browser-tester, code-reviewer | Always -- core to progressive disclosure pattern |
| Absolute paths in Task prompts | `~/.claude/skills/gsd-verify-work/references/agents/...` | Reference files within skill folder from Task prompts | Every agent spawn |
| gsd-verifier.md (existing) | `skills/gsd-execute-phase/references/agents/gsd-verifier.md` | Goal-backward phase verification (artifacts, links, wiring) | Remains in execute-phase for post-execution phase goal verification; NOT the same as runtime Chrome DevTools verification |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Single verify skill covering both goal verification AND runtime testing | Separate skills (existing gsd-verifier stays in execute-phase, new gsd-verify-work does runtime) | Separate skills maintain the established pattern; gsd-verifier does goal-backward codebase verification while gsd-verify-work does runtime browser testing. Different concerns, different tools. |
| Embedding review logic in execute-phase | Standalone gsd-review skill | Standalone is cleaner, maintains role separation, and allows review to be invoked independently |
| TypeScript compiler (`npm run typecheck`) for type checking | IDE diagnostics MCP | IDE diagnostics gives live editor diagnostics with line numbers; ALSO run typecheck as backup (belt and suspenders) |
| Removing executor per-task commits immediately | Gradual transition with review skill handling final commit | Gradual is safer; executor can still commit per-task during transition, review skill does final verified commit |

## Architecture Patterns

### Recommended Skill Folder Structure

```
skills/
  gsd-verify-work/
    SKILL.md                              # ~100-150 lines, orchestration only
    references/
      agents/
        gsd-browser-tester.md             # Chrome DevTools runtime testing agent
      templates/
        verification-report.md            # VERIFICATION.md template (optional)

  gsd-review/
    SKILL.md                              # ~100-150 lines, orchestration only
    references/
      agents/
        gsd-code-reviewer.md              # Code review + best-practice checking agent
      templates/
        review-report.md                  # REVIEW.md template (optional)
```

Installed copies go to:
- `~/.claude/skills/gsd-verify-work/`
- `~/.claude/skills/gsd-review/`

Repo source copies stay at:
- `skills/gsd-verify-work/`
- `skills/gsd-review/`

### Pattern 1: Verify Skill -- Chrome DevTools Runtime Testing

**What:** The verify skill navigates to the running application, takes DOM snapshots, interacts with UI elements (click, fill), and inspects console/network output to verify the feature works at runtime.

**When to use:** After execute-phase completes, before code review.

**Orchestrator flow:**
1. Initialize via `gsd-tools.js init phase-op "$PHASE"`
2. Read SUMMARY.md files from phase to understand what was built
3. Determine what to verify (feature URLs, expected behaviors, API endpoints)
4. Spawn browser-tester agent with Chrome DevTools MCP access
5. Agent performs snapshot -> interact -> verify testing cycle
6. Agent writes VERIFICATION.md report (runtime findings only)
7. Orchestrator checks status: passed / issues_found / inconclusive
8. Return structured result to caller

**Frontmatter:**
```yaml
---
name: gsd-verify-work
description: >
  Verify a phase's work by performing runtime testing with Chrome DevTools MCP.
  Navigates the running application, takes snapshots, interacts with UI, and
  checks console/network output. Use when user says "verify phase", "test phase",
  "/gsd:verify-work", or after execute-phase completes.
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
```

**PIPE-04 enforcement:** No Edit tool in allowed-tools. The verify skill can only Read source files, Write reports, and use Chrome DevTools. It NEVER modifies source files.

### Pattern 2: Review Skill -- Code Quality + Commit-on-PASS

**What:** The review skill loads best-practice skills, runs IDE diagnostics, performs structured code review with severity levels, and commits ONLY if the review passes.

**When to use:** After verify-work completes (or after execute-phase if verify is skipped).

**Orchestrator flow:**
1. Initialize via `gsd-tools.js init phase-op "$PHASE"`
2. Read SUMMARY.md files to identify all files created/modified
3. Spawn code-reviewer agent with IDE diagnostics MCP and best-practice skill paths
4. Agent reads all modified files, loads relevant best-practice skills, checks IDE diagnostics
5. Agent writes REVIEW.md with severity-graded findings (Critical / High / Medium / Low)
6. Agent determines result: PASS or FAIL
7. Orchestrator reads result
8. If PASS: commit the executor's work with descriptive message
9. If FAIL: display issues, suggest re-execution

**Frontmatter:**
```yaml
---
name: gsd-review
description: >
  Review code quality for a phase's executed work. Loads best-practice skills,
  runs IDE diagnostics, and produces a structured review. Commits only after
  PASS result. Use when user says "review phase", "/gsd:review", or after
  verify-work completes.
argument-hint: "<phase-number>"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - mcp__ide__getDiagnostics
---
```

**PIPE-05 enforcement:** The reviewer commits only after PASS. The allowed-tools includes Bash (for git commit). The review agent's instructions explicitly mandate: write REVIEW.md first, determine PASS/FAIL, then commit only if PASS.

### Pattern 3: Browser-Tester Agent (for verify skill)

**What:** A specialized agent that receives the list of features to verify and executes the Chrome DevTools testing cycle.

**Testing cycle (from nick:verify reference):**
```
1. list_pages -- find the running app
2. navigate_page -- go to the feature URL
3. take_snapshot -- capture DOM structure
4. Interact: click, fill, press_key as needed
5. take_snapshot -- capture post-interaction state
6. list_console_messages (types: ["error", "warn"]) -- check for errors
7. list_network_requests -- verify API calls
8. get_network_request -- inspect specific responses
9. performance_start_trace + performance_stop_trace + performance_analyze_insight -- check performance
```

**Agent receives:**
- Phase directory path (to read SUMMARY.md files)
- List of features/URLs to test
- Expected behaviors from PLAN.md must_haves

**Agent produces:**
- VERIFICATION.md with runtime findings
- Structured result: passed / issues_found / inconclusive

### Pattern 4: Code-Reviewer Agent (for review skill)

**What:** A specialized agent that performs structured code review using best-practice skills, IDE diagnostics, and manual code reading.

**Review steps (from nick:review reference):**
1. Read ALL files from SUMMARY.md (created + modified)
2. Load relevant best-practice skills based on technology:
   - React/Next.js -> `vercel-react-best-practices`
   - NestJS -> `nestjs-best-practices`
   - UI components -> `web-design-guidelines`
   - Security-sensitive -> `owasp-security`
3. Run `mcp__ide__getDiagnostics` for TypeScript errors
4. Run `npm run typecheck`, `npm run test`, `npm run build` as available
5. Check plan adherence (code matches plan intent)
6. Write REVIEW.md with severity-graded findings
7. Determine PASS/FAIL

**Severity levels:**
- **Critical:** Must fix before commit (security vulnerabilities, data loss, broken functionality)
- **High:** Should fix before commit (type errors, missing error handling, violations of critical best-practice rules)
- **Medium:** Fix in follow-up (code clarity, minor best-practice deviations)
- **Low:** Nice to have (style, optimization suggestions)

**PASS criteria:** Zero Critical findings. Zero High findings. Medium and Low are documented but do not block.

### Pattern 5: Full Pipeline Integration (PIPE-01)

**What:** The complete plan -> execute -> verify -> review pipeline for a phase.

**Two implementation approaches:**

**Option A: Sequential manual invocation**
```
User: /gsd:plan-phase 3
User: /gsd:execute-phase 3
User: /gsd:verify-work 3
User: /gsd:review 3
```
Each skill runs independently. The user triggers each step.

**Option B: Execute-phase orchestrates the full pipeline**
The execute-phase SKILL.md is updated to optionally chain verify and review after execution:
```
Step 4: Execute Waves (existing)
Step 5: Verify Phase Goal (existing gsd-verifier)
Step 6: Verify Runtime (NEW -- spawn gsd-verify-work or its agent)
Step 7: Review Code (NEW -- spawn gsd-review or its agent)
Step 8: Commit (moved from executor to here, conditional on review PASS)
Step 9: Update Roadmap
```

**Recommendation:** Implement Option A first (standalone skills). This allows each skill to be tested independently and used flexibly. Option B (pipeline integration) can be added as a thin update to execute-phase SKILL.md that chains the skills. The pipeline is satisfied as long as all four stages can run in sequence for a phase.

### Pattern 6: PIPE-03 Completion -- Executor Stops Committing

**What:** The executor agent currently commits per-task. With the review skill in place, the executor should stop committing and let the reviewer handle commits.

**Current state (executor commits per-task):**
```markdown
<!-- From gsd-executor.md task_commit_protocol -->
After each task completes, commit immediately.
git add src/api/auth.ts
git commit -m "feat(01-01): add auth endpoint"
```

**Target state (executor stages but does not commit):**
```markdown
After each task completes:
1. Stage task-related files: git add src/api/auth.ts
2. Record staged files in SUMMARY.md (for reviewer to commit)
3. Do NOT run git commit
```

**Important consideration:** This is a significant behavioral change. The executor currently produces atomic per-task commits, which is valuable for rollback and audit. Removing per-task commits in favor of one big review-PASS commit changes the git history granularity.

**Alternative approach (recommended):** Keep executor per-task commits but add review as a GATE before the phase is marked complete. The reviewer validates all commits, and if FAIL, the executor's commits can be reverted. This preserves atomic commit history while adding quality gates.

**Decision for planner:** Whether to remove executor per-task commits (strict PIPE-03) or keep them and add reviewer as a quality gate (practical PIPE-03). Research recommends the quality-gate approach.

### Anti-Patterns to Avoid

- **Verify skill modifying source files:** The verify skill is read-only + write report. It NEVER edits, creates, or deletes source files. Only writes VERIFICATION.md.
- **Review skill committing before writing REVIEW.md:** The review file MUST be fully written before any commit occurs. Order: read files -> check quality -> write REVIEW.md -> determine PASS/FAIL -> commit only if PASS.
- **Combining verify and review into one skill:** They serve different purposes (runtime testing vs code quality) and use different tools (Chrome DevTools vs IDE diagnostics + best-practice skills). Keeping them separate maintains role separation.
- **Running Chrome DevTools tests without app running:** If the app is not running, Chrome DevTools tests are "inconclusive", not "failed". The verify skill must handle this gracefully.
- **Blocking pipeline on Chrome DevTools unavailability:** Chrome DevTools MCP requires Chrome running with remote debugging. If unavailable, the verify step should be skippable (not block the entire pipeline).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Browser automation | Custom Puppeteer scripts | Chrome DevTools MCP (26 tools) | Official MCP handles element selection, waiting, error recovery |
| TypeScript error checking | Custom tsc parser | `mcp__ide__getDiagnostics` + `npm run typecheck` | IDE diagnostics provides live errors with line numbers; tsc as backup |
| Best-practice rule checking | Custom lint rules | Existing SKILL.md skills (React, NestJS, OWASP, web-design) | Already curated, categorized by severity, maintained independently |
| VERIFICATION.md structure | Custom format | Existing gsd-verifier pattern from execute-phase | Proven format with frontmatter, structured gaps, human verification sections |
| Git commit with file staging | Raw git commands | `gsd-tools.js commit` | Handles secret scanning, file staging, message formatting |
| Phase initialization | Custom init logic | `gsd-tools.js init phase-op` or `init verify-phase` | Already handles context loading, file discovery |

**Key insight:** The nick:verify and nick:review skills are proven reference implementations. Phase 3's job is to adapt their patterns into the GSD multi-phase architecture (`.planning/phases/` instead of `.pipeline/`), add agent definitions to `references/agents/`, and integrate with the plan-execute pipeline.

## Common Pitfalls

### Pitfall 1: Chrome DevTools MCP Not Connected

**What goes wrong:** The verify skill tries to use `mcp__chrome-devtools__*` tools but the MCP server is not running, producing tool call errors.
**Why it happens:** Chrome DevTools MCP requires Chrome to be running with remote debugging enabled (`--remote-debugging-port=9222`). This is not always the case.
**How to avoid:** Pre-flight check: attempt `list_pages` first. If it fails, report "Chrome DevTools MCP unavailable -- runtime verification skipped" and set status to "inconclusive". Do NOT block the pipeline.
**Warning signs:** Tool call errors on first Chrome DevTools invocation; agent hanging waiting for Chrome.

### Pitfall 2: IDE Diagnostics Returning Stale/Incorrect Errors

**What goes wrong:** `mcp__ide__getDiagnostics` returns TypeScript errors that do not actually exist when verified with the TypeScript compiler.
**Why it happens:** Known issue -- IDE diagnostics can show stale errors, especially for files not currently open in the editor. File timing, editor state, and language server warmup all affect results.
**How to avoid:** Use IDE diagnostics as a FIRST CHECK but always cross-verify with `npm run typecheck` for critical findings. If IDE diagnostics shows errors that typecheck does not, ignore the IDE diagnostic errors (false positives). If typecheck shows errors that IDE diagnostics missed, include them (false negatives).
**Warning signs:** Diagnostic errors that reference deleted or renamed files; errors that contradict successful builds.

### Pitfall 3: Verify Skill Scope Confusion with Existing gsd-verifier

**What goes wrong:** The new `gsd-verify-work` skill is confused with the existing `gsd-verifier.md` agent in execute-phase. They have overlapping names but different purposes.
**Why it happens:** Both deal with "verification" but at different levels.
**How to avoid:** Clear naming and documentation:
- `gsd-verifier.md` (existing, in execute-phase): Goal-backward codebase verification. Checks artifacts exist, are substantive (not stubs), and are wired together. Uses grep/file checks, NOT the running app.
- `gsd-verify-work` (new, standalone skill): Runtime browser verification. Uses Chrome DevTools MCP to test the running application. Completely different tool set and approach.
**Warning signs:** Planning documents conflating the two; agents reading the wrong verification file.

### Pitfall 4: Review Committing Executor's Incomplete Work

**What goes wrong:** The reviewer commits after PASS but the executor has unstaged or uncommitted files that were part of the work.
**Why it happens:** The executor may have created files that are not tracked by git or not listed in SUMMARY.md.
**How to avoid:** The reviewer should run `git status` before committing to identify ALL modified/untracked files. Cross-reference with SUMMARY.md's key-files section. If there are files modified but not in SUMMARY.md, investigate before committing.
**Warning signs:** `git status` showing untracked files after review commit; files listed in SUMMARY.md but not staged.

### Pitfall 5: PIPE-03 Transition Breaking Atomic Commits

**What goes wrong:** Removing executor per-task commits loses the atomic commit history. A review-PASS commit bundles all tasks into one giant commit, making rollback difficult.
**Why it happens:** Strict interpretation of PIPE-03 ("executor never commits") conflicts with the value of atomic per-task commits.
**How to avoid:** Two viable approaches:
1. **Keep per-task commits + review gate:** Executor commits per-task as today. Reviewer validates all commits. On PASS, no additional commit needed (work already committed). On FAIL, reviewer can `git revert` the failing commits.
2. **Executor stages only + reviewer commits all:** Executor does `git add` per task but no `git commit`. Reviewer does one commit if PASS. Simpler review logic but loses atomic history.
Recommend approach 1 for this phase (least disruptive, preserves audit trail).
**Warning signs:** Single 500-line commit replacing 5 atomic per-task commits; inability to revert individual tasks.

### Pitfall 6: Best-Practice Skill Loading Consuming Excessive Context

**What goes wrong:** The code-reviewer agent loads all 4 best-practice skills for every review, consuming 15-20% of context before starting the actual review.
**Why it happens:** "Load relevant skills" interpreted as "load all skills."
**How to avoid:** Use the same detection heuristics from Phase 2's executor: only load skills matching the technology in the SUMMARY.md files. Read SKILL.md first (rule summary), then specific references/ only for relevant categories.
**Warning signs:** Context usage exceeding 40% before reading any source files; reading entire skill references/ directories.

## Code Examples

### Example 1: Verify Skill SKILL.md Body (Orchestrator)

```markdown
# Verify Work

You orchestrate runtime verification using Chrome DevTools MCP. You spawn a browser-tester agent -- you do NOT test the app yourself.

## Step 1: Initialize

```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.js init phase-op "$PHASE")
```

Parse: phase_dir, phase_number, phase_name.

## Step 2: Load Phase Context

Read SUMMARY.md files from phase directory to understand what was built:
- What features were implemented
- What files were created/modified
- What URLs/routes should be testable

## Step 3: Pre-flight Check

Attempt `list_pages` to verify Chrome DevTools MCP is available.
- If unavailable: report "Chrome DevTools MCP not connected" and set status to "inconclusive". Offer user to skip or retry.
- If available: proceed.

## Step 4: Spawn Browser Tester

```
Task(
  prompt="First, read ~/.claude/skills/gsd-verify-work/references/agents/gsd-browser-tester.md for your role.

  <objective>
  Verify phase {phase_number} runtime behavior using Chrome DevTools.
  Test features built in this phase using snapshot-interact-verify cycle.
  </objective>

  <files_to_read>
  - {phase_dir}/*-SUMMARY.md (all summaries -- understand what was built)
  - {phase_dir}/*-PLAN.md (all plans -- understand expected behavior)
  - .planning/ROADMAP.md (phase goal)
  </files_to_read>

  <output>Create {phase_dir}/{padded_phase}-VERIFICATION.md</output>",
  subagent_type="general-purpose"
)
```

## Step 5: Handle Result

Read status from VERIFICATION.md frontmatter:
- **passed:** Runtime verification complete. Proceed to review.
- **issues_found:** Display issues. Offer re-execution or proceed to review anyway.
- **inconclusive:** Chrome DevTools unavailable or app not running. Offer skip.

## Step 6: Present Results and Offer Next

Display verification summary. Offer `/gsd:review {phase}`.
```

### Example 2: Review Skill SKILL.md Body (Orchestrator)

```markdown
# Review

You orchestrate code review for phase work. You spawn a code-reviewer agent -- you do NOT review code yourself. You commit ONLY after the review result is PASS.

## Step 1: Initialize

```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.js init phase-op "$PHASE")
```

Parse: phase_dir, phase_number, phase_name, commit_docs.

## Step 2: Spawn Code Reviewer

```
Task(
  prompt="First, read ~/.claude/skills/gsd-review/references/agents/gsd-code-reviewer.md for your role.

  <objective>
  Review code quality for phase {phase_number}-{phase_name}.
  Load relevant best-practice skills. Run IDE diagnostics. Produce REVIEW.md.
  Determine PASS or FAIL.
  </objective>

  <files_to_read>
  - {phase_dir}/*-SUMMARY.md (all summaries -- identify files to review)
  - {phase_dir}/*-PLAN.md (all plans -- check plan adherence)
  - {phase_dir}/*-VERIFICATION.md (if exists -- include runtime findings)
  </files_to_read>

  <output>Create {phase_dir}/{padded_phase}-REVIEW.md</output>",
  subagent_type="general-purpose"
)
```

## Step 3: Read Review Result

Parse REVIEW.md frontmatter for `result: PASS | FAIL`.

## Step 4: Handle Result

**If PASS:**
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.js commit \
  "docs(phase-{X}): review PASSED -- phase verified and reviewed" \
  --files {phase_dir}/*-REVIEW.md {phase_dir}/*-VERIFICATION.md
```
Display: "Review PASSED. Phase {X} work committed."

**If FAIL:**
Do NOT commit. Display critical/high issues from REVIEW.md.
Offer: `/gsd:execute-phase {X} --gaps` to fix flagged issues.

## Step 5: Present Results
```

### Example 3: Browser-Tester Agent Definition (Key Sections)

```markdown
---
name: gsd-browser-tester
description: Runtime verification agent using Chrome DevTools MCP. Performs
  snapshot-interact-verify testing on the running application.
tools: Read, Bash, Grep, Glob, mcp__chrome-devtools__*
color: cyan
---

<role>
You are a GSD runtime verifier. You test the running application using Chrome
DevTools MCP to verify that built features actually work.

You do NOT modify source files. You do NOT review code quality. You only test
runtime behavior and report findings.
</role>

<testing_protocol>

## Snapshot-Interact-Verify Cycle

For each feature to verify:

1. **Navigate:** `navigate_page` to the feature URL
2. **Snapshot:** `take_snapshot` to capture initial DOM state
3. **Inspect:** Check snapshot for expected elements, content, structure
4. **Interact:** `click`, `fill`, `press_key` as needed for the workflow
5. **Re-snapshot:** `take_snapshot` after interaction
6. **Verify:** Check post-interaction state matches expected behavior
7. **Console:** `list_console_messages` with types: ["error", "warn"]
8. **Network:** `list_network_requests` to verify API calls
9. **Performance:** `performance_start_trace` + `performance_stop_trace` + `performance_analyze_insight`

## Checks for Every Page

- Zero console errors (warnings may be acceptable)
- API responses return expected status codes (check with `get_network_request`)
- No long tasks >50ms in performance trace
- Page loads cleanly without layout shifts
- Interactive elements respond to input

</testing_protocol>
```

### Example 4: Code-Reviewer Agent Definition (Key Sections)

```markdown
---
name: gsd-code-reviewer
description: Code quality reviewer that loads best-practice skills, runs IDE
  diagnostics, and produces structured review with severity levels.
tools: Read, Bash, Grep, Glob, Write, mcp__ide__getDiagnostics
color: purple
---

<role>
You are a GSD code reviewer. You review executed code for quality, security,
and best-practice compliance. You produce a structured REVIEW.md with
severity-graded findings.

You do NOT modify source files. You do NOT commit code. You only report findings
and determine PASS/FAIL. The orchestrator handles commits.
</role>

<review_protocol>

## Mandatory Review Steps

### 1. Identify Files to Review
Read SUMMARY.md files. Extract all created/modified file paths from key-files sections.

### 2. Load Best-Practice Skills
Based on files being reviewed:
| File Patterns | Skill to Load |
|---------------|---------------|
| *.tsx, *.jsx, page.tsx, layout.tsx | vercel-react-best-practices |
| *.controller.ts, *.service.ts, *.module.ts | nestjs-best-practices |
| UI components, layouts, visual files | web-design-guidelines |
| Auth, validation, API endpoints, user data | owasp-security |

### 3. Read Every File
Read ALL files from SUMMARY.md. Check for:
- Logic errors, edge cases, missing error handling
- TypeScript strictness (no `any`, proper types)
- Code clarity and maintainability
- Plan adherence

### 4. Run IDE Diagnostics
```bash
mcp__ide__getDiagnostics
```
Cross-verify with:
```bash
npm run typecheck 2>&1
```

### 5. Run Test Suite (if available)
```bash
npm run test 2>&1
npm run build 2>&1
```

### 6. Determine Result
- **PASS:** Zero Critical, Zero High findings
- **FAIL:** Any Critical or High finding present

</review_protocol>

<review_output>

## REVIEW.md Structure

```markdown
---
phase: XX-name
reviewed: YYYY-MM-DDTHH:MM:SSZ
result: PASS | FAIL
skills_loaded:
  - vercel-react-best-practices
  - owasp-security
diagnostics:
  typecheck: PASS | FAIL
  tests: PASS | FAIL | N/A
  build: PASS | FAIL | N/A
  ide_errors: 0
findings:
  critical: 0
  high: 0
  medium: 2
  low: 1
---

# Phase {X}: {Name} Code Review

## Result: PASS / FAIL

## Critical Findings (must fix)
[None / list with file:line, issue, fix recommendation]

## High Findings (should fix)
[None / list with file:line, issue, recommendation]

## Medium Findings (fix later)
[list with file, issue, suggestion]

## Low Findings (nice to have)
[list with observation]

## Best Practices Review
### Violations Found
### Patterns Confirmed

## Test Results
- Typecheck: PASS / FAIL
- Unit tests: X passing, Y failing
- Build: PASS / FAIL

## Plan Adherence
Did the implementation match the plan? Note any deviations.

## Summary
Brief overall assessment.
```

</review_output>
```

### Example 5: VERIFICATION.md Structure (Runtime)

Note: This is DIFFERENT from the existing gsd-verifier's VERIFICATION.md (which does goal-backward codebase verification). This is runtime verification.

```markdown
---
phase: XX-name
verified: YYYY-MM-DDTHH:MM:SSZ
type: runtime
status: passed | issues_found | inconclusive
chrome_devtools: connected | unavailable
tests:
  total: 12
  passed: 11
  failed: 1
  skipped: 0
console_errors: 0
network_issues: 0
performance_long_tasks: 0
---

# Phase {X}: {Name} Runtime Verification

## Status: {status}

## Test Results

### Feature: {feature name}
| # | Test | Status | Evidence |
|---|------|--------|----------|
| 1 | Navigate to /feature | PASSED | Page loaded, snapshot shows expected elements |
| 2 | Fill form fields | PASSED | Fields accepted input |
| 3 | Submit form | FAILED | Console error: "TypeError: Cannot read property..." |

## Console Inspection
- Errors: {count}
- Warnings: {count}
- Details: [list any errors/warnings]

## Network Inspection
- Total requests: {count}
- Failed requests: {count}
- Details: [list any failures]

## Performance
- Long tasks (>50ms): {count}
- Insights: [from performance_analyze_insight]

## Issues Found
[If any -- structured for gap closure]

## Summary
[Brief assessment of runtime behavior]
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| No runtime verification in GSD pipeline | Chrome DevTools MCP for snapshot-interact-verify testing | Phase 3 (this work) | Features verified in running browser, not just code inspection |
| No code review in GSD pipeline | Best-practice skill loading + IDE diagnostics | Phase 3 (this work) | Code quality enforced before merge |
| Executor commits per-task, no gate | Review skill gates commits on PASS/FAIL | Phase 3 (this work) | Only reviewed code gets committed (PIPE-03, PIPE-05) |
| Two-stage pipeline (plan -> execute) | Four-stage pipeline (plan -> execute -> verify -> review) | Phase 3 (this work) | Complete quality pipeline (PIPE-01) |
| Goal-backward verification only (gsd-verifier) | Goal-backward + runtime verification (separate concerns) | Phase 3 (this work) | Both codebase structure AND runtime behavior verified |

**Deprecated/outdated:**
- `gsd-verifier.md` in execute-phase is NOT deprecated. It still handles goal-backward codebase verification (artifact existence, stub detection, wiring checks). The new `gsd-verify-work` skill handles RUNTIME verification via Chrome DevTools. They coexist.
- nick:verify and nick:review at `docs/nick/` are reference implementations using the `.pipeline/` structure. The GSD versions adapt these patterns to `.planning/phases/` structure.

## Requirement-to-Change Mapping

| Requirement | What Changes | Files to Create/Modify |
|-------------|-------------|------------------------|
| SKILL-03: Verify skill with Chrome DevTools | Create new skill `gsd-verify-work` with SKILL.md + browser-tester agent | `skills/gsd-verify-work/SKILL.md`, `skills/gsd-verify-work/references/agents/gsd-browser-tester.md` |
| SKILL-04: Review skill with best-practice loading | Create new skill `gsd-review` with SKILL.md + code-reviewer agent | `skills/gsd-review/SKILL.md`, `skills/gsd-review/references/agents/gsd-code-reviewer.md` |
| MCP-06: Chrome DevTools in verify skill | Browser-tester agent uses all 20+ Chrome DevTools MCP tools | `gsd-browser-tester.md` allowed-tools and testing protocol |
| MCP-07: IDE diagnostics in review skill | Code-reviewer agent uses `mcp__ide__getDiagnostics` | `gsd-code-reviewer.md` review protocol step 4 |
| PIPE-01: Full pipeline | All four skills can run in sequence for a phase | Verify by integration testing; optionally update execute-phase to chain verify+review |
| PIPE-04: Verifier never modifies source | Verify skill's allowed-tools excludes Edit; agent instructions enforce read-only | `gsd-verify-work/SKILL.md` frontmatter |
| PIPE-05: Reviewer commits only after PASS | Review skill orchestrator commits after reading PASS from REVIEW.md | `gsd-review/SKILL.md` Step 4 logic |

## Open Questions

1. **PIPE-03: Remove executor per-task commits or keep as quality gate?**
   - What we know: PIPE-03 says "executor never commits." The executor currently commits per-task (deferred from Phase 1). The review skill can now handle commits.
   - What's unclear: Whether removing atomic per-task commits is worth the loss of granular git history.
   - Recommendation: Keep executor per-task commits for Phase 3. Add the review skill as a quality GATE (validates commits, blocks phase completion on FAIL). Strict PIPE-03 (executor never commits) can be revisited in Phase 4 if needed. This is the least disruptive approach and preserves the audit trail.

2. **Pipeline orchestration: separate skills or chained from execute-phase?**
   - What we know: The verify and review skills can be standalone (/gsd:verify-work, /gsd:review) or chained from execute-phase as post-execution steps.
   - What's unclear: Whether users want to invoke them separately or as part of a single flow.
   - Recommendation: Create them as standalone skills first. Later, add optional chaining from execute-phase. This allows flexible usage and independent testing.

3. **What does the verify skill test when there is no UI?**
   - What we know: Chrome DevTools MCP is designed for web applications. GSD customization project (this project) has no UI -- it produces markdown files and SKILL.md definitions.
   - What's unclear: How the verify skill applies to non-UI phases.
   - Recommendation: The verify skill should be OPTIONAL. For non-UI phases, skip verify (set status to "N/A -- no runtime component") and proceed directly to review. The review skill (code quality) applies universally. The verify skill is valuable for web application projects that GSD manages.

4. **Runtime VERIFICATION.md vs existing goal-backward VERIFICATION.md naming collision**
   - What we know: The existing gsd-verifier in execute-phase creates `{phase}-VERIFICATION.md`. The new gsd-verify-work would also want to create a verification report.
   - What's unclear: Whether both should use the same filename or different filenames.
   - Recommendation: Use different filenames to avoid collision: existing gsd-verifier creates `XX-VERIFICATION.md` (goal-backward), new gsd-verify-work creates `XX-RUNTIME-VERIFICATION.md` (or merges findings into existing VERIFICATION.md with a new section). The planner should decide the exact naming.

5. **IDE diagnostics reliability**
   - What we know: GitHub issues document that `mcp__ide__getDiagnostics` can return stale/incorrect errors, especially for files not active in the editor.
   - What's unclear: How reliable it is in practice for the review skill's use case.
   - Recommendation: Use IDE diagnostics as a FIRST CHECK (fast, live errors) but always cross-verify critical findings with `npm run typecheck`. If they disagree, trust the compiler. Document this belt-and-suspenders approach in the code-reviewer agent.

## Sources

### Primary (HIGH confidence)
- `docs/nick/verify.md` -- User's reference verify skill with Chrome DevTools MCP integration (read directly, 157 lines)
- `docs/nick/review.md` -- User's reference review skill with IDE diagnostics and commit-on-PASS (read directly, 207 lines)
- `skills/gsd-execute-phase/SKILL.md` -- Current execute-phase orchestrator showing agent spawn pattern (read directly)
- `skills/gsd-execute-phase/references/agents/gsd-verifier.md` -- Existing goal-backward verifier agent (read directly, 523 lines)
- `skills/gsd-execute-phase/references/agents/gsd-executor.md` -- Current executor with per-task commit protocol (read directly, 476 lines)
- `skills/gsd-plan-phase/SKILL.md` -- Current plan-phase orchestrator showing SKILL.md body pattern (read directly)
- `.planning/ROADMAP.md` -- Phase 3 requirements and success criteria (read directly)
- `.planning/REQUIREMENTS.md` -- SKILL-03, SKILL-04, MCP-06, MCP-07, PIPE-01, PIPE-04, PIPE-05 definitions (read directly)
- `.planning/phases/01-core-skills-token-architecture/01-VERIFICATION.md` -- Verification report format reference (read directly)
- `.planning/phases/01-core-skills-token-architecture/01-01-PLAN.md` -- PLAN.md format reference (read directly)

### Secondary (MEDIUM confidence)
- [Chrome DevTools MCP GitHub](https://github.com/ChromeDevTools/chrome-devtools-mcp) -- Official repo, 26 tools across 6 categories, Puppeteer-based
- [Chrome DevTools MCP npm](https://www.npmjs.com/package/chrome-devtools-mcp) -- Package info, setup configuration
- [Chrome DevTools MCP Blog](https://developer.chrome.com/blog/chrome-devtools-mcp) -- Official Chrome team blog post
- [IDE diagnostics GitHub issue #6393](https://github.com/anthropics/claude-code/issues/6393) -- Known stale/incorrect error issue
- [IDE diagnostics GitHub issue #3085](https://github.com/anthropics/claude-code/issues/3085) -- getDiagnostics timeout for non-active files

### Tertiary (LOW confidence)
- PIPE-03 strict enforcement impact on git history -- no empirical data on whether removing per-task commits causes problems. Recommendation based on architectural reasoning.
- IDE diagnostics reliability in practice -- known issues documented but actual false positive rate unclear.

## Metadata

**Confidence breakdown:**
- Standard stack (Chrome DevTools MCP, IDE diagnostics): HIGH -- tools confirmed available, tool names verified against user's existing skills and official docs
- Architecture (SKILL.md + agents pattern): HIGH -- follows exact same pattern as Phase 1/2 skills, no novel architecture
- Verify skill design: HIGH -- direct adaptation of proven nick:verify reference implementation
- Review skill design: HIGH -- direct adaptation of proven nick:review reference implementation
- PIPE-03 transition strategy: MEDIUM -- multiple viable approaches, recommendation based on architectural reasoning without empirical validation
- IDE diagnostics reliability: MEDIUM -- known issues documented in GitHub, workaround (belt-and-suspenders with typecheck) is solid

**Research date:** 2026-02-16
**Valid until:** 2026-03-16 (Chrome DevTools MCP is stable; IDE diagnostics is a built-in that evolves with Claude Code releases)
