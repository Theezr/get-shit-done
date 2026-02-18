---
phase: quick-7
plan: 01
status: complete
commits:
  - hash: eba8659
    scope: "fix(quick-7): add commit_docs guards to all 8 agent files"
  - hash: 8a74758
    scope: "fix(quick-7): add commit_docs guards to all 17 workflow and skill files"
  - hash: 4ff2640
    scope: "docs(quick-7): document commit_docs guard pattern in git-integration reference"
files_modified:
  - agents/nick-executor.md
  - agents/nick-planner.md
  - agents/nick-phase-researcher.md
  - agents/nick-debugger.md
  - agents/nick-research-synthesizer.md
  - skills/nick-execute-phase/references/agents/nick-executor.md
  - skills/nick-plan-phase/references/agents/nick-planner.md
  - skills/nick-plan-phase/references/agents/nick-phase-researcher.md
  - get-shit-done/workflows/quick.md
  - get-shit-done/workflows/execute-phase.md
  - get-shit-done/workflows/execute-plan.md
  - get-shit-done/workflows/new-project.md
  - get-shit-done/workflows/discuss-phase.md
  - get-shit-done/workflows/verify-work.md
  - get-shit-done/workflows/map-codebase.md
  - get-shit-done/workflows/new-milestone.md
  - get-shit-done/workflows/pause-work.md
  - get-shit-done/workflows/add-todo.md
  - get-shit-done/workflows/complete-milestone.md
  - get-shit-done/workflows/remove-phase.md
  - get-shit-done/workflows/plan-milestone-gaps.md
  - get-shit-done/workflows/check-todos.md
  - get-shit-done/workflows/diagnose-issues.md
  - skills/nick-execute-phase/SKILL.md
  - skills/nick-review/SKILL.md
  - get-shit-done/references/git-integration.md
key-decisions:
  - "commit_docs guards at prompt level prevent LLM agents from misinterpreting skipped commit results"
  - "Reference doc examples (git-integration format blocks, planning-config) left as-is since they are documentation templates not execution paths"
---

## Summary

Added `If commit_docs is true (from init JSON):` conditional guards around every `gsd-tools.cjs commit` call that commits `.planning/` files across 26 files (8 agent files, 17 workflow/skill files, 1 reference doc).

## Problem

When `.planning/` is gitignored, agents called `gsd-tools.cjs commit` and received `{"committed": false, "reason": "skipped_gitignored"}`. LLM agents could misinterpret "skipped" as failure and attempt workarounds (raw git commands, retries).

## Solution

Pre-guard every `.planning/` commit call with a `commit_docs` conditional check. Agents already extract `commit_docs` from init JSON but never used it to gate commit calls. Now they skip the call entirely when `commit_docs` is false, so they never see a "skipped" response to misinterpret.

## Files Changed

- **8 agent files** — wrapped final_commit/git_commit sections with `commit_docs` conditional
- **17 workflow/skill files** — wrapped all `gsd-tools.cjs commit` calls for `.planning/` files
- **1 reference doc** — added `<commit_docs_guard>` section to git-integration.md documenting the pattern
