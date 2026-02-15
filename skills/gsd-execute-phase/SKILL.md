---
name: gsd-execute-phase
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
---

# Execute Phase

You orchestrate phase execution. You spawn agents -- you do NOT execute plans yourself.

Orchestrator stays lean (~10-15% context). Each executor agent gets fresh 200k context.

## Step 1: Initialize

```bash
INIT=$(node /Users/nick/.claude/get-shit-done/bin/gsd-tools.cjs init execute-phase "$PHASE_ARG")
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
PLAN_INDEX=$(node /Users/nick/.claude/get-shit-done/bin/gsd-tools.cjs phase-plan-index "$PHASE_NUMBER")
```

Parse: `plans[]` (each with `id`, `wave`, `autonomous`, `objective`, `has_summary`), `waves`, `incomplete`.

**Filtering:** Skip plans where `has_summary: true`. If `--gaps-only`: skip non-gap_closure plans. If all filtered: exit.

Display execution plan table: Wave | Plans | What it builds.

## Step 4: Execute Waves

For each wave in sequence:

**4a. Describe what's being built** -- Read plan objectives, summarize technical approach.

**4b. Spawn executor agents** -- Parallel if `parallelization=true`, sequential if `false`:

```
Task(
  prompt="First, read ~/.claude/skills/gsd-execute-phase/references/agents/gsd-executor.md for your role.

  <objective>
  Execute plan {plan_id} of phase {phase_number}-{phase_name}.
  Commit each task atomically. Create SUMMARY.md. Update STATE.md.
  </objective>

  <execution_context>
  Read these workflow files for execution instructions:
  @/Users/nick/.claude/get-shit-done/workflows/execute-plan.md
  @/Users/nick/.claude/get-shit-done/templates/summary.md
  @/Users/nick/.claude/get-shit-done/references/checkpoints.md
  @/Users/nick/.claude/get-shit-done/references/tdd.md
  </execution_context>

  <files_to_read>
  Read these files at execution start using the Read tool:
  - Plan: {phase_dir}/{plan_file}
  - State: .planning/STATE.md
  - Config: .planning/config.json (if exists)
  </files_to_read>

  <success_criteria>
  - [ ] All tasks executed
  - [ ] Each task committed individually
  - [ ] SUMMARY.md created in plan directory
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

**4e. Handle checkpoint plans** (`autonomous: false`): Agent returns structured state. Present to user. Spawn fresh continuation agent with completed tasks table and resume point.

**4f. Proceed to next wave.**

## Step 5: Verify Phase Goal

Spawn verifier:

```
Task(
  prompt="First, read ~/.claude/skills/gsd-execute-phase/references/agents/gsd-verifier.md for your role.

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
node /Users/nick/.claude/get-shit-done/bin/gsd-tools.cjs commit "docs(phase-{X}): complete phase execution" --files .planning/ROADMAP.md .planning/STATE.md .planning/phases/{phase_dir}/*-VERIFICATION.md .planning/REQUIREMENTS.md
```

## Step 7: Offer Next

- If more phases: offer `/gsd:plan-phase {X+1}` (suggest `/clear` first for fresh context).
- If milestone complete: offer `/gsd:complete-milestone`.
