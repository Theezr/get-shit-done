---
name: nick-review
description: >
  Review code quality for a phase's executed work. Loads best-practice skills,
  runs IDE diagnostics, and produces a structured review with severity-graded
  findings. Commits review artifacts only after PASS result. Use when user says
  "review phase", "review code", "/gsd:review", or after verify-work completes.
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

# Review

You orchestrate code review for phase work. You spawn a code-reviewer agent -- you do NOT review code yourself. You commit review artifacts ONLY after the review result is PASS.

Orchestrator stays lean. The code-reviewer agent gets fresh 200k context.

## Step 1: Initialize

```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs init phase-op "$PHASE_ARG")
```

Parse: `phase_dir`, `phase_number`, `phase_name`, `padded_phase`, `commit_docs`.

- If phase not found: Error -- phase directory does not exist.

## Step 2: Load Phase Context

Read from the phase directory to build review context:

- Read all `*-SUMMARY.md` files to identify ALL files created/modified (key-files sections)
- Read all `*-PLAN.md` files for plan adherence checking

If no SUMMARY.md files found: Error -- "No execution summaries found. Run `/gsd:execute-phase` first."

## Step 3: Spawn Code Reviewer

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

  <output>Create {phase_dir}/{padded_phase}-REVIEW.md</output>",
  subagent_type="general-purpose"
)
```

## Step 4: Read Review Result

Parse `{phase_dir}/{padded_phase}-REVIEW.md` frontmatter for `result: PASS | FAIL`.

If REVIEW.md does not exist: Error -- "Code reviewer did not produce REVIEW.md."

## Step 5: Handle Result

**If PASS:**

- Run `git status` to identify all modified/untracked files in the phase directory
- Cross-reference with SUMMARY.md key-files to ensure nothing is missed
- Commit review artifacts:

```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit \
  "docs(phase-{X}): review PASSED -- code quality verified" \
  --files {phase_dir}/{padded_phase}-REVIEW.md
```

- Display: "Review PASSED. Phase {X} review artifacts committed."
- Note: This commits the REVIEW and VERIFICATION reports only. The executor's code commits are already in git (per-task commits from the executor). The review GATE validates those commits are quality-approved.

**If FAIL:**

- Do NOT commit anything
- Display critical and high findings from REVIEW.md
- Offer: "Fix issues and re-execute, then re-review: `/gsd:execute-phase {X} --gaps` then `/gsd:review {X}`"

## Step 6: Present Results

Display review summary with findings count by severity.

- If PASS: Offer next phase planning `/gsd:plan-phase {X+1}` (suggest `/clear` first for fresh context).
- If FAIL: Offer gap closure path.
