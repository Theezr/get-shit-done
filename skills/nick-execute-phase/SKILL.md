---
name: nick-execute-phase
description: >
  Execute all plans in a phase with wave-based parallel agents. Spawns executor
  agents per plan and a verifier agent for phase goal checking. Use when user
  says "execute phase", "run phase", "build phase", "/gsd:execute-phase", or
  wants to execute a planned phase. Requires planned phase with PLAN.md files.
argument-hint: "<phase-number> [--gaps-only]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Task
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
---

# Execute Phase

You orchestrate phase execution. You spawn agents -- you do NOT execute plans yourself.

Orchestrator stays lean (~10-15% context). Each executor agent gets fresh 200k context.

## Step 1: Initialize

```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs init execute-phase "$PHASE_ARG")
```

Parse: `executor_model`, `verifier_model`, `commit_docs`, `parallelization`, `branching_strategy`, `branch_name`, `phase_found`, `phase_dir`, `phase_number`, `phase_name`, `phase_slug`, `plans`, `incomplete_plans`, `plan_count`, `incomplete_count`.

- If `phase_found` is false: Error -- phase directory not found.
- If `plan_count` is 0: Error -- no plans found.

## Step 2: Handle Branching

Check `branching_strategy` from init:

- **"none":** Skip, continue on current branch.
- **"phase" or "milestone":** Checkout `branch_name` from init.

## Step 3: Discover and Group Plans

```bash
PLAN_INDEX=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs phase-plan-index "$PHASE_NUMBER")
```

Parse: `plans[]` (each with `id`, `wave`, `autonomous`, `type`, `objective`, `has_summary`), `waves`, `incomplete`.

**Filtering:** Skip plans where `has_summary: true`. If `--gaps-only`: skip non-gap_closure plans. If all filtered: exit.

Display execution plan table: Wave | Plans | What it builds.

## Step 4: Execute Waves

For each wave in sequence:

**4a. Describe what's being built** -- Read plan objectives, summarize technical approach.

**4b. Spawn executor agents** -- Parallel if `parallelization=true`, sequential if `false`:

```
Task(
  prompt="First, read ~/.claude/skills/nick-execute-phase/references/agents/nick-executor.md for your role.

  <objective>
  Execute plan {plan_id} of phase {phase_number}-{phase_name}.

  BEFORE implementing each task:
  1. Verify the plan's API references via Context7 (check RESEARCH.md first)
  2. Load relevant best-practice skills for the technology being used

  Commit each task atomically. Create SUMMARY.md (include skills-loaded). Update STATE.md.
  </objective>

  <execution_context>
  Read these workflow files for execution instructions:
  @~/.claude/get-shit-done/workflows/execute-plan.md
  @~/.claude/get-shit-done/templates/summary.md
  </execution_context>

  **Conditional references (include in execution_context based on plan frontmatter):**
  - If the plan's `autonomous` is `false`: Add this line to execution_context:
    `@~/.claude/skills/nick-execute-phase/references/agents/nick-executor-checkpoints.md`
  - If the plan's `type` is `tdd`: Add this line to execution_context:
    `@~/.claude/skills/nick-execute-phase/references/agents/nick-executor-tdd.md`

  <files_to_read>
  - Plan: {phase_dir}/{plan_file}
  - Research: {phase_dir}/*-RESEARCH.md (if exists -- check for pre-verified APIs)
  - State: .planning/STATE.md
  - Config: .planning/config.json (if exists)
  </files_to_read>

  <success_criteria>
  - [ ] API references verified via Context7 before implementation (or confirmed in RESEARCH.md)
  - [ ] Relevant best-practice skills loaded before writing code
  - [ ] All tasks executed
  - [ ] Each task committed individually
  - [ ] SUMMARY.md created with skills-loaded field
  - [ ] STATE.md updated with position and decisions
  </success_criteria>",
  subagent_type="general-purpose"
)
```

**4c. Wait** for all agents in wave to complete.

**4d. Spot-check** -- For each completed plan:
- Verify first 2 files from SUMMARY.md `key-files.created` exist on disk
- Check `git log --oneline --all --grep="{phase}-{plan}"` returns at least 1 commit
- Check for `## Self-Check: FAILED` marker

If spot-check fails: ask user to retry or continue.

**classifyHandoffIfNeeded bug:** If agent reports "failed" with `classifyHandoffIfNeeded is not defined`, this is a Claude Code bug. Run spot-checks -- if PASS, treat as successful.

**4e. Handle checkpoint plans** (`autonomous: false`): Agent returns structured state. Present to user. Spawn fresh continuation agent with completed tasks table and resume point. **IMPORTANT:** The continuation agent's execution_context MUST include nick-executor-checkpoints.md -- continuation handling is in that file.

**4f. Proceed to next wave.**

## Step 4.5: Parallel Verify + Review

After all waves complete, run verification and review in parallel. This replaces the need for manual `/gsd:review` invocation.

**Check for frontend work:**

```bash
HAS_FRONTEND=$(grep -l "has_frontend: true" "$PHASE_DIR"/*-PLAN.md 2>/dev/null | wc -l | tr -d ' ')
```

**If HAS_FRONTEND > 0:** Spawn BOTH agents in parallel (two Task() calls in the same response):

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

Task B (review) -- spawns the code-reviewer agent DIRECTLY (bypasses review skill orchestrator to avoid nested orchestration and double commits):

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

**If HAS_FRONTEND = 0:** Spawn review ONLY (same Task B as above, no Task A):

Inform user: "No frontend work detected. Running code review..."

**After both tasks complete (or just review if no frontend), handle combined results:**

Read REVIEW.md frontmatter for `result: PASS | FAIL`.
Read RUNTIME-VERIFICATION.md frontmatter for `status: passed | issues_found | inconclusive` (if file exists).

**Result matrix (ALL 5 combinations):**

| Review Result | Verify Status | Action |
|---------------|---------------|--------|
| PASS | passed | Commit REVIEW.md + RUNTIME-VERIFICATION.md. Log "Review PASSED, runtime verified." |
| PASS | issues_found | Commit REVIEW.md + RUNTIME-VERIFICATION.md. Display verify issues. Ask: "Runtime issues found. Address before phase goal verification?" |
| PASS | inconclusive | Commit REVIEW.md only. Log "Review PASSED. Runtime verification inconclusive (Chrome DevTools unavailable)." |
| PASS | N/A (no frontend) | Commit REVIEW.md only. Log "Review PASSED. No frontend -- runtime verification skipped." |
| FAIL | any | Do NOT commit. Display review findings (Critical + High). Offer: "Fix issues and re-execute: `/gsd:execute-phase {X} --gaps`" |

**Commit (when review PASS):**

Build the file list dynamically based on what exists:

```bash
FILES="{phase_dir}/{padded_phase}-REVIEW.md"
if [ -f "{phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md" ]; then
  FILES="$FILES {phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md"
fi

node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit \
  "docs(phase-{X}): review PASSED -- code quality verified" \
  --files $FILES
```

**IMPORTANT details:**
- The review agent is spawned DIRECTLY using nick-code-reviewer.md (not the review skill orchestrator), to avoid nested orchestration and double commits. The `<critical>DO NOT COMMIT</critical>` instruction prevents the agent from committing.
- Both Task() calls MUST appear in the same response block to run in parallel.
- Step 5 (Verify Phase Goal) remains unchanged and sequential AFTER Step 4.5.

## Step 5: Verify Phase Goal

Spawn verifier:

```
Task(
  prompt="First, read ~/.claude/skills/nick-execute-phase/references/agents/nick-verifier.md for your role.

  <objective>
  Verify phase {phase_number} achieved its goal.
  Phase directory: {phase_dir}
  Phase goal: {goal}
  </objective>

  <files_to_read>
  - {phase_dir}/*-PLAN.md (all plans for must_haves)
  - {phase_dir}/*-SUMMARY.md (all summaries)
  - .planning/ROADMAP.md (phase goal)
  </files_to_read>

  <output>Create {phase_dir}/{padded_phase}-VERIFICATION.md</output>",
  subagent_type="general-purpose"
)
```

Read status from VERIFICATION.md:
- **passed:** Proceed to update roadmap.
- **human_needed:** Present items for human testing. Get approval or feedback.
- **gaps_found:** Present gap summary. Offer `/gsd:plan-phase {phase} --gaps`.

## Step 6: Update Roadmap

Mark phase complete in ROADMAP.md (date, status).

```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "docs(phase-{X}): complete phase execution" --files .planning/ROADMAP.md .planning/STATE.md .planning/phases/{phase_dir}/*-VERIFICATION.md .planning/REQUIREMENTS.md
```

## Step 7: Offer Next

- If more phases: offer `/gsd:plan-phase {X+1}` (suggest `/clear` first for fresh context).
- If milestone complete: offer `/gsd:complete-milestone`.
