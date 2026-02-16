# Phase 8: Combine Review and Verify-Work Skills into Parallel Execution - Research

**Researched:** 2026-02-16
**Domain:** Workflow orchestration (parallel subagent execution within execute-phase pipeline)
**Confidence:** HIGH

## Summary

Phase 8 optimizes the post-execution pipeline by running the verify-work (runtime browser testing) and review (code quality assessment) skills in parallel instead of sequentially. Currently, verify-work runs as Step 4.5 inside execute-phase, then the user manually invokes review afterward. The goal is to have both run concurrently after wave execution completes, reducing the overall pipeline wall-clock time.

The core challenge is a soft dependency: the review skill currently reads `*-RUNTIME-VERIFICATION.md` "if it exists" to enrich its review context with runtime findings. When running in parallel, this file will not exist yet when review starts. The solution is straightforward -- remove the dependency from the review agent's input (review never uses runtime findings for PASS/FAIL decisions anyway, only for enrichment), run both as parallel Task() calls, then handle the combined results in the orchestrator. The commit gate (commit both REVIEW.md and RUNTIME-VERIFICATION.md on PASS) naturally moves to a post-parallel-completion step.

**Primary recommendation:** Add a new Step 4.5 to execute-phase that spawns verify-work and review as parallel Task() calls, collects both results, then handles the combined outcome (commit on review PASS, surface issues from either). Remove the review skill's soft dependency on RUNTIME-VERIFICATION.md input. Keep both skills independently invocable for standalone use.

## Standard Stack

### Core

This phase modifies existing markdown skill files. No new libraries or tools are introduced.

| Component | Location | Purpose | Change Type |
|-----------|----------|---------|-------------|
| nick-execute-phase/SKILL.md | `~/.claude/skills/nick-execute-phase/SKILL.md` | Orchestrator -- gains parallel verify+review step | Major: rewrite Step 4.5 to parallel |
| nick-review/SKILL.md | `~/.claude/skills/nick-review/SKILL.md` | Review skill -- remove RUNTIME-VERIFICATION dependency | Minor: remove optional input |
| nick-code-reviewer.md | `~/.claude/skills/nick-review/references/agents/nick-code-reviewer.md` | Review agent -- remove RUNTIME-VERIFICATION from files_to_read | Minor: remove optional input |

### Unchanged

| Component | Location | Why No Change |
|-----------|----------|---------------|
| nick-verify-work/SKILL.md | `~/.claude/skills/nick-verify-work/SKILL.md` | Already produces RUNTIME-VERIFICATION.md independently |
| nick-browser-tester.md | `~/.claude/skills/nick-verify-work/references/agents/nick-browser-tester.md` | Already handles snapshot-interact-verify cycle |
| nick-verifier.md | `~/.claude/skills/nick-execute-phase/references/agents/nick-verifier.md` | Phase goal verification (Step 5) is separate; stays sequential after parallel step |
| nick-executor.md | `~/.claude/skills/nick-execute-phase/references/agents/nick-executor.md` | Wave execution unchanged |
| nick-plan-checker.md | `~/.claude/skills/nick-plan-phase/references/agents/nick-plan-checker.md` | Planning unchanged |

## Architecture Patterns

### Current Pipeline Flow (Before Phase 8)

```
execute-phase orchestrator:
  Step 1: Initialize
  Step 2: Handle Branching
  Step 3: Discover and Group Plans
  Step 4: Execute Waves (parallel executors per wave)
  Step 4.5: Auto-Verify Frontend (conditional, sequential)
      |-- if has_frontend: spawn verify-work -> RUNTIME-VERIFICATION.md
      |-- handle passed/issues_found/inconclusive
  Step 5: Verify Phase Goal (sequential)
      |-- spawn nick-verifier -> VERIFICATION.md
  Step 6: Update Roadmap
  Step 7: Offer Next

User manually runs:
  /gsd:review {phase}  --> spawns review skill --> REVIEW.md (commits on PASS)
```

**Problems:**
1. verify-work and review run sequentially (user must invoke review separately)
2. Review waits for verify-work to complete before starting
3. Two separate invocations increase user friction
4. Total wall-clock time = verify-work time + review time

### Target Pipeline Flow (After Phase 8)

```
execute-phase orchestrator:
  Step 1: Initialize
  Step 2: Handle Branching
  Step 3: Discover and Group Plans
  Step 4: Execute Waves (parallel executors per wave)
  Step 4.5: Parallel Verify + Review (NEW)
      |-- Spawn TWO parallel Task() calls:
      |   Task A: verify-work --> RUNTIME-VERIFICATION.md
      |   Task B: review --> REVIEW.md
      |-- Wait for BOTH to complete
      |-- Handle combined results:
      |   - Review PASS + verify passed --> commit both artifacts, continue
      |   - Review PASS + verify issues --> commit review, surface verify issues
      |   - Review PASS + verify inconclusive --> commit review, warn about manual verify
      |   - Review FAIL --> do NOT commit, surface review findings
      |   - Review FAIL + verify issues --> surface both, offer gap closure
  Step 5: Verify Phase Goal (sequential, unchanged)
  Step 6: Update Roadmap
  Step 7: Offer Next
```

**Benefits:**
1. Both run concurrently -- wall-clock time = max(verify-work, review) instead of sum
2. Single orchestrator step handles both
3. User no longer needs to manually invoke review
4. Commit gate still works (orchestrator commits after both complete)

### Pattern 1: Parallel Task() Spawning in Execute-Phase

**What:** The execute-phase orchestrator already uses parallel Task() calls for wave execution (Step 4b: "Parallel if `parallelization=true`"). The same pattern applies to verify+review.

**When to use:** After all waves complete (Step 4 done), before phase goal verification (Step 5).

**How it works:** Claude Code's Task tool allows multiple Task() calls in a single response. When multiple tasks are spawned simultaneously, they run as parallel subagents with independent 200k contexts. The orchestrator waits for all to complete before proceeding.

**Example (pseudocode for SKILL.md):**
```
## Step 4.5: Parallel Verify + Review

After all waves complete, spawn verify-work and review in parallel.

**Determine if frontend verification is needed:**
```bash
HAS_FRONTEND=$(grep -l "has_frontend: true" "$PHASE_DIR"/*-PLAN.md 2>/dev/null | wc -l | tr -d ' ')
```

**Spawn parallel agents:**

Always spawn the review agent. Conditionally spawn verify-work if frontend work exists.

If HAS_FRONTEND > 0, spawn BOTH in parallel:

Task A (verify-work):
  prompt="First, read ~/.claude/skills/nick-verify-work/SKILL.md..."
  <output>Create {phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md</output>

Task B (review):
  prompt="First, read ~/.claude/skills/nick-review/SKILL.md..."
  <output>Create {phase_dir}/{padded_phase}-REVIEW.md</output>

If HAS_FRONTEND = 0, spawn review only:

Task B (review):
  prompt="First, read ~/.claude/skills/nick-review/SKILL.md..."
  <output>Create {phase_dir}/{padded_phase}-REVIEW.md</output>
```

### Pattern 2: Decoupled Review (No RUNTIME-VERIFICATION Dependency)

**What:** The review skill currently reads RUNTIME-VERIFICATION.md "if it exists" to enrich context. For parallel execution, this dependency must be removed.

**Impact analysis of removing the dependency:**
- The review agent (nick-code-reviewer.md) checks code quality, types, tests, plan adherence
- Runtime verification checks browser behavior, console errors, network issues
- These are ORTHOGONAL concerns -- the review agent never uses runtime findings for PASS/FAIL decisions
- The "if it exists" clause confirms this was always optional enrichment, not a hard dependency
- Removing it from `files_to_read` in the review's Task spawn is safe

**Files to change:**
1. `nick-review/SKILL.md` Step 2 -- remove the line: `- Read *-RUNTIME-VERIFICATION.md if it exists (include runtime findings in review context)`
2. `nick-review/SKILL.md` Step 3 -- remove from files_to_read: `- {phase_dir}/*-RUNTIME-VERIFICATION.md (if exists -- include runtime findings)`
3. `nick-review/SKILL.md` Step 5 (PASS commit) -- remove RUNTIME-VERIFICATION.md from commit files list (orchestrator handles this now)

### Pattern 3: Orchestrator-Owned Commit Gate

**What:** Currently, the review skill commits REVIEW.md and RUNTIME-VERIFICATION.md on PASS. With parallel execution, the orchestrator owns the commit gate because it has both results.

**Current (in nick-review/SKILL.md Step 5):**
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit \
  "docs(phase-{X}): review PASSED -- code quality verified" \
  --files {phase_dir}/{padded_phase}-REVIEW.md {phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md
```

**New (in execute-phase/SKILL.md Step 4.5 result handling):**
The orchestrator reads both results, then commits:
```bash
# After both tasks complete, if review PASS:
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit \
  "docs(phase-{X}): review PASSED + runtime verified" \
  --files {phase_dir}/{padded_phase}-REVIEW.md {phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md
```

If RUNTIME-VERIFICATION.md does not exist (non-frontend phase), commit only REVIEW.md.

### Pattern 4: Review Skill Remains Independently Invocable

**What:** The review skill must still work when invoked standalone via `/gsd:review {phase}`. The parallel execution is an optimization in the execute-phase orchestrator, not a removal of the standalone skill.

**Standalone behavior (unchanged):**
- User runs `/gsd:review {phase}`
- Review skill initializes, loads context, spawns code-reviewer
- Code-reviewer produces REVIEW.md
- Review skill commits on PASS

**Only change:** Review no longer reads RUNTIME-VERIFICATION.md. This is fine for standalone use -- the review was always complete without it.

**Verify-work remains independently invocable too** via `/gsd:verify-work {phase}`.

### Anti-Patterns to Avoid

- **Race condition on output files:** Both agents write to the same phase directory but different files (RUNTIME-VERIFICATION.md vs REVIEW.md). No conflict. Do NOT have either agent read the other's output.
- **Double commit:** The orchestrator commits after both complete. The review skill must NOT commit on its own when spawned from the orchestrator. When spawned standalone, it still commits. Solution: the orchestrator's Task prompt tells the review agent to NOT commit (same pattern as nick-verifier: "DO NOT COMMIT. Leave committing to the orchestrator.").
- **Blocking on verify-work for non-frontend:** If no frontend work, only spawn review. Do NOT spawn verify-work with empty/irrelevant input.
- **Losing the standalone review commit:** When review runs standalone (not from orchestrator), it must still commit on PASS. The skill file's Step 5 commit logic stays. The orchestrator just overrides by telling the spawned review agent not to commit.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Parallel agent execution | Custom promise/thread wrapper | Claude Code Task() parallel spawning | Already proven in wave execution (Step 4b) |
| Result aggregation | Custom status file watcher | Sequential orchestrator reads after Task() returns | Task() blocks until complete; no polling needed |
| Commit orchestration | Per-agent commits with merge logic | Single orchestrator commit after both complete | Simpler, no merge conflicts, single atomic commit |

**Key insight:** This phase is purely a workflow orchestration change. No new tools, libraries, or agents are created. The existing Task() parallel pattern from wave execution is reused for the verify+review step.

## Common Pitfalls

### Pitfall 1: Review Agent Commits When Spawned from Orchestrator

**What goes wrong:** The review skill's Step 5 commits REVIEW.md on PASS. If the orchestrator also tries to commit, you get duplicate commits or the orchestrator's commit fails because the file is already committed.
**Why it happens:** The review skill was designed for standalone use where it owns the commit.
**How to avoid:** The orchestrator's Task prompt for review must include an explicit instruction: "DO NOT COMMIT. Leave committing to the orchestrator." This matches the pattern used for nick-verifier (which also says "DO NOT COMMIT"). The review skill's Step 5 logic remains for standalone invocations.
**Warning signs:** `git log` shows two commits for the same REVIEW.md in the same phase.

### Pitfall 2: Review Skill Standalone Invocation Breaks

**What goes wrong:** Changes to the review skill to support orchestrator-spawned mode accidentally break standalone `/gsd:review` behavior.
**Why it happens:** Over-coupling the review skill to the orchestrator's parallel pattern.
**How to avoid:** The review skill's SKILL.md retains ALL existing functionality. The ONLY change is removing RUNTIME-VERIFICATION.md from inputs. The "DO NOT COMMIT" instruction comes from the orchestrator's Task prompt, not from the skill itself. The skill's own Step 5 commit logic is unchanged.
**Warning signs:** Running `/gsd:review {phase}` standalone produces REVIEW.md but does not commit on PASS.

### Pitfall 3: Verify-Work Inconclusive Blocks Review Completion

**What goes wrong:** The orchestrator waits for both tasks but verify-work hangs or takes very long because Chrome DevTools is not connected.
**Why it happens:** verify-work pre-flight check (list_pages) may timeout waiting for Chrome DevTools MCP.
**How to avoid:** Verify-work already handles this gracefully -- it reports "inconclusive" when Chrome DevTools is unavailable (Step 3 of nick-verify-work/SKILL.md). The orchestrator should not wait indefinitely. Both tasks should complete in reasonable time regardless of Chrome DevTools availability.
**Warning signs:** Review completes quickly but the orchestrator appears to hang waiting for verify-work.

### Pitfall 4: Step 5 (Phase Goal Verification) Becomes Redundant

**What goes wrong:** With verify+review in Step 4.5, someone assumes Step 5 is no longer needed.
**Why it happens:** Confusion between runtime verification (verify-work), code review (review), and phase goal verification (nick-verifier). They are three different things.
**How to avoid:** Keep Step 5 (phase goal verification) exactly as-is. It serves a different purpose: goal-backward static analysis of whether the phase achieved its stated goal. This is orthogonal to runtime testing and code quality review.
**Warning signs:** VERIFICATION.md no longer produced after phase execution.

### Pitfall 5: Combined Result Handling Matrix Is Incomplete

**What goes wrong:** The orchestrator handles "review PASS + verify passed" but does not handle all combinations (review FAIL + verify passed, review PASS + verify inconclusive, etc.).
**Why it happens:** The matrix has 3x3 combinations (review: PASS/FAIL vs verify: passed/issues_found/inconclusive) plus the non-frontend case.
**How to avoid:** Explicitly enumerate ALL result combinations in the SKILL.md. Use the result matrix documented in this research (see Architecture Patterns section).
**Warning signs:** Orchestrator hits an unhandled case and stops without clear guidance.

## Code Examples

### Example 1: Orchestrator Step 4.5 -- Parallel Verify + Review (Frontend Phase)

```markdown
## Step 4.5: Parallel Verify + Review

After all waves complete, run verification and review in parallel.

**Check for frontend work:**
```bash
HAS_FRONTEND=$(grep -l "has_frontend: true" "$PHASE_DIR"/*-PLAN.md 2>/dev/null | wc -l | tr -d ' ')
```

**If HAS_FRONTEND > 0:** Spawn BOTH agents in parallel:

Inform user: "Running runtime verification and code review in parallel..."

Task A (verify-work):
```
Task(
  prompt="First, read ~/.claude/skills/nick-verify-work/SKILL.md for the verification workflow.

  <objective>
  Auto-verify frontend work for phase {phase_number}-{phase_name}.
  Phase directory: {phase_dir}
  </objective>

  <files_to_read>
  - {phase_dir}/*-SUMMARY.md
  - {phase_dir}/*-PLAN.md
  </files_to_read>

  <output>Create {phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md</output>",
  subagent_type="general-purpose"
)
```

Task B (review):
```
Task(
  prompt="First, read ~/.claude/skills/nick-review/references/agents/nick-code-reviewer.md for your role.

  <objective>
  Review code quality for phase {phase_number}-{phase_name}.
  Load relevant best-practice skills based on file patterns.
  Run IDE diagnostics and typecheck (belt-and-suspenders).
  Produce REVIEW.md with severity-graded findings.
  Determine PASS or FAIL.
  </objective>

  <files_to_read>
  - {phase_dir}/*-SUMMARY.md (all summaries -- identify files to review)
  - {phase_dir}/*-PLAN.md (all plans -- check plan adherence)
  </files_to_read>

  <output>Create {phase_dir}/{padded_phase}-REVIEW.md</output>

  <critical>DO NOT COMMIT. Leave committing to the orchestrator.</critical>",
  subagent_type="general-purpose"
)
```

**If HAS_FRONTEND = 0:** Spawn review only:

Inform user: "No frontend work detected. Running code review..."

[Same Task B as above, only review agent spawned]
```

### Example 2: Combined Result Handling Matrix

```markdown
**After both tasks complete, read results:**

Read REVIEW.md frontmatter for `result: PASS | FAIL`.
Read RUNTIME-VERIFICATION.md frontmatter for `status: passed | issues_found | inconclusive` (if it exists).

**Result matrix:**

| Review Result | Verify Status | Action |
|---------------|---------------|--------|
| PASS | passed | Commit REVIEW.md + RUNTIME-VERIFICATION.md. Log "Review PASSED, runtime verified." |
| PASS | issues_found | Commit REVIEW.md + RUNTIME-VERIFICATION.md. Display verify issues. Ask user: "Runtime issues found. Address before phase goal verification?" |
| PASS | inconclusive | Commit REVIEW.md only. Log "Review PASSED. Runtime verification skipped (Chrome DevTools unavailable)." |
| PASS | N/A (no frontend) | Commit REVIEW.md only. Log "Review PASSED. No frontend -- runtime verification skipped." |
| FAIL | any | Do NOT commit. Display review findings (Critical + High). Offer: "Fix issues and re-execute: `/gsd:execute-phase {X} --gaps`" |

**Commit (when review PASS):**
```bash
# Build file list based on what exists
FILES="{phase_dir}/{padded_phase}-REVIEW.md"
if [ -f "{phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md" ]; then
  FILES="$FILES {phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md"
fi

node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit \
  "docs(phase-{X}): review PASSED -- code quality verified" \
  --files $FILES
```
```

### Example 3: Review Skill Changes (Removing RUNTIME-VERIFICATION Dependency)

Current nick-review/SKILL.md Step 2:
```markdown
## Step 2: Load Phase Context

Read from the phase directory to build review context:

- Read all `*-SUMMARY.md` files to identify ALL files created/modified (key-files sections)
- Read all `*-PLAN.md` files for plan adherence checking
- Read `*-RUNTIME-VERIFICATION.md` if it exists (include runtime findings in review context)  <-- REMOVE THIS LINE
```

Current nick-review/SKILL.md Step 3 (Task spawn files_to_read):
```markdown
  <files_to_read>
  - {phase_dir}/*-SUMMARY.md (all summaries -- identify files to review)
  - {phase_dir}/*-PLAN.md (all plans -- check plan adherence)
  - {phase_dir}/*-RUNTIME-VERIFICATION.md (if exists -- include runtime findings)  <-- REMOVE THIS LINE
  </files_to_read>
```

Current nick-review/SKILL.md Step 5 (PASS commit):
```bash
# BEFORE (review owns commit):
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit \
  "docs(phase-{X}): review PASSED -- code quality verified" \
  --files {phase_dir}/{padded_phase}-REVIEW.md {phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md

# AFTER (review still commits when standalone, but without RUNTIME-VERIFICATION):
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit \
  "docs(phase-{X}): review PASSED -- code quality verified" \
  --files {phase_dir}/{padded_phase}-REVIEW.md
```

### Example 4: Review Agent Task Prompt Differences

**When spawned from orchestrator (parallel mode):**
- Files_to_read: SUMMARY.md + PLAN.md only (no RUNTIME-VERIFICATION)
- Critical instruction: "DO NOT COMMIT. Leave committing to the orchestrator."
- The orchestrator directly spawns the code-reviewer agent (skipping the review skill orchestrator layer)

**When invoked standalone (/gsd:review):**
- Review skill orchestrator (SKILL.md) runs normally
- Code-reviewer agent spawned by review orchestrator
- Review orchestrator commits on PASS (its own Step 5)
- No RUNTIME-VERIFICATION.md in files_to_read (removed by this phase)

## Modification Surface

### Files That Need Changes

| File | Repo Location | Installed Location | Change Type | What Changes |
|------|--------------|-------------------|-------------|-------------|
| nick-execute-phase/SKILL.md | `skills/nick-execute-phase/SKILL.md` | `~/.claude/skills/nick-execute-phase/SKILL.md` | Major rewrite of Step 4.5 | Step 4.5 becomes parallel verify+review with result matrix |
| nick-review/SKILL.md | `skills/nick-review/SKILL.md` | `~/.claude/skills/nick-review/SKILL.md` | Minor removal | Remove RUNTIME-VERIFICATION.md from Step 2, Step 3, and Step 5 |

### Files That Do NOT Change

| File | Why No Change |
|------|---------------|
| nick-verify-work/SKILL.md | Already produces RUNTIME-VERIFICATION.md independently; no changes needed |
| nick-browser-tester.md | Already handles runtime testing independently |
| nick-code-reviewer.md | Review agent reads files_to_read from Task prompt, not from SKILL.md directly. The orchestrator's Task prompt controls what files to read. |
| nick-verifier.md (phase goal) | Step 5 is separate and sequential; unchanged |
| nick-executor.md | Wave execution unchanged |

### Dual Location Pattern

All modified skill files exist in TWO locations (established in Phase 5):
1. **Repo source:** `skills/` directory in the project (version control)
2. **Installed location:** `~/.claude/skills/` (where Claude Code loads them)

Both must be updated identically. Sync with:
```bash
cp skills/nick-execute-phase/SKILL.md ~/.claude/skills/nick-execute-phase/SKILL.md
cp skills/nick-review/SKILL.md ~/.claude/skills/nick-review/SKILL.md
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Verify then review (sequential, separate invocations) | Verify + review in parallel (single orchestrator step) | Phase 8 (this phase) | ~50% wall-clock reduction for post-execution pipeline |
| Review reads RUNTIME-VERIFICATION.md | Review is independent of runtime verification | Phase 8 (this phase) | Enables parallel execution; review is self-contained |
| Review skill owns commit gate | Orchestrator owns commit gate (when parallel) | Phase 8 (this phase) | Single atomic commit after both agents complete |
| User manually invokes /gsd:review | Review auto-triggered as part of execute-phase | Phase 8 (this phase) | One less manual step; full pipeline in single invocation |

**Deprecated/outdated:**
- Review's RUNTIME-VERIFICATION.md dependency: removed. Was always optional enrichment, not a functional requirement.

## Open Questions

1. **Should the review agent be spawned directly (code-reviewer) or via the review skill orchestrator?**
   - What we know: The review skill (SKILL.md) is itself a thin orchestrator that spawns the code-reviewer agent. When running from execute-phase, we are already inside an orchestrator.
   - What's unclear: Whether to spawn the review skill (which spawns code-reviewer) or spawn code-reviewer directly. Spawning the skill means two layers of orchestration. Spawning the agent directly saves one layer but duplicates some setup logic.
   - Recommendation: Spawn the code-reviewer agent directly from execute-phase's Step 4.5, bypassing the review skill orchestrator. This avoids nested orchestration and keeps the execute-phase orchestrator lean. The "DO NOT COMMIT" instruction is naturally part of the direct spawn prompt. The review skill orchestrator remains for standalone use.

2. **Should Step 5 (phase goal verification) also run in parallel with verify+review?**
   - What we know: Step 5 (nick-verifier) does static codebase analysis. It does not depend on RUNTIME-VERIFICATION.md or REVIEW.md. In theory, all three could run in parallel.
   - What's unclear: Whether the phase goal verifier benefits from seeing review/verify results before running, and whether three parallel agents is too much for the orchestrator to manage.
   - Recommendation: Keep Step 5 sequential for now. The phase goal verifier was designed to run after execution, and its result determines whether to update the roadmap. Adding it to the parallel step complicates the result handling matrix (now 3x3x3 combinations). This can be a future optimization.

3. **What if the review agent takes much longer than verify-work (or vice versa)?**
   - What we know: Review loads best-practice skills, reads all modified files, runs typecheck/build, and IDE diagnostics. Verify-work navigates pages, takes snapshots, and interacts with UI. They could have very different durations.
   - What's unclear: Typical relative durations.
   - Recommendation: Not a concern. The orchestrator waits for both. The slower one determines wall-clock time. Even if one is much slower, the total is still less than the sum. No timeout is needed beyond Claude Code's natural Task timeout.

## Sources

### Primary (HIGH confidence)

- Direct file read: `~/.claude/skills/nick-execute-phase/SKILL.md` -- current orchestrator with Step 4.5 auto-verify
- Direct file read: `~/.claude/skills/nick-verify-work/SKILL.md` -- current verify-work skill
- Direct file read: `~/.claude/skills/nick-review/SKILL.md` -- current review skill with RUNTIME-VERIFICATION dependency
- Direct file read: `~/.claude/skills/nick-review/references/agents/nick-code-reviewer.md` -- review agent definition
- Direct file read: `~/.claude/skills/nick-execute-phase/references/agents/nick-verifier.md` -- phase goal verifier
- Direct file read: `~/.claude/skills/nick-verify-work/references/agents/nick-browser-tester.md` -- browser tester agent
- Direct file read: `.planning/ROADMAP.md` -- phase descriptions and dependencies
- Direct file read: `.planning/STATE.md` -- accumulated decisions from phases 1-7
- Direct file read: `.planning/config.json` -- parallelization=true confirmed
- Direct file read: Phase 7 RESEARCH.md and PLAN.md files -- understand how Step 4.5 was designed

### Secondary (MEDIUM confidence)

- Claude Code Task() parallel pattern -- observed in execute-phase wave execution (Step 4b), reused for this phase

## Metadata

**Confidence breakdown:**
- Modification surface: HIGH -- all files read directly, change points identified precisely, only 2 files need changes
- Architecture patterns: HIGH -- reusing existing parallel Task() pattern from wave execution
- Dependency analysis: HIGH -- review's RUNTIME-VERIFICATION dependency is clearly optional ("if it exists"), confirmed by reading the code-reviewer agent which never references runtime findings in its PASS/FAIL logic
- Pitfalls: HIGH -- based on direct analysis of current commit flow and agent interactions
- Open questions: MEDIUM -- direct spawning vs skill orchestrator is a design decision with clear tradeoffs

**Research date:** 2026-02-16
**Valid until:** 2026-03-16 (30 days -- stable domain, all changes are to local workflow files)
