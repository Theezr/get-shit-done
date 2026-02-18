---
phase: quick-7
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  # Agent files (repo source + skills reference copies)
  - agents/nick-executor.md
  - agents/nick-planner.md
  - agents/nick-phase-researcher.md
  - agents/nick-debugger.md
  - agents/nick-research-synthesizer.md
  - skills/nick-execute-phase/references/agents/nick-executor.md
  - skills/nick-plan-phase/references/agents/nick-planner.md
  - skills/nick-plan-phase/references/agents/nick-phase-researcher.md
  # Workflow/skill files
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
  # Reference docs
  - get-shit-done/references/git-integration.md
autonomous: true
must_haves:
  truths:
    - "When commit_docs is false, no agent or workflow attempts to git-commit .planning files"
    - "When commit_docs is true, behavior is unchanged from current"
    - "Agents never see 'skipped' commit results because they skip the call entirely"
  artifacts:
    - path: "agents/nick-executor.md"
      provides: "Executor agent with conditional .planning commit"
      contains: "commit_docs"
    - path: "agents/nick-planner.md"
      provides: "Planner agent with conditional .planning commits"
      contains: "commit_docs"
    - path: "get-shit-done/workflows/quick.md"
      provides: "Quick workflow with conditional .planning commit"
      contains: "commit_docs"
  key_links:
    - from: "all agents and workflows"
      to: "gsd-tools.cjs commit"
      via: "commit_docs conditional guard"
      pattern: "commit_docs.*(true|false)"
---

<objective>
Add `commit_docs` conditional guards around every `gsd-tools.cjs commit` call that commits .planning files, across all agent and workflow files.

Purpose: When .planning is gitignored (commit_docs=false), agents currently call `gsd-tools.cjs commit`, get back a "skipped" JSON response, and may interpret this as failure -- leading to retry loops or raw git command fallbacks. The fix is to skip the commit call entirely when commit_docs is false.

Output: All 26 files updated with conditional commit guards.
</objective>

<execution_context>
@/Users/nick/.claude/get-shit-done/workflows/execute-plan.md
@/Users/nick/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/STATE.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Add commit_docs guards to all agent files (8 files)</name>
  <files>
    agents/nick-executor.md
    agents/nick-planner.md
    agents/nick-phase-researcher.md
    agents/nick-debugger.md
    agents/nick-research-synthesizer.md
    skills/nick-execute-phase/references/agents/nick-executor.md
    skills/nick-plan-phase/references/agents/nick-planner.md
    skills/nick-plan-phase/references/agents/nick-phase-researcher.md
  </files>
  <action>
For each agent file, wrap every `gsd-tools.cjs commit` call that commits .planning files with a `commit_docs` conditional. The pattern is:

**Before (current):**
```
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "docs(...): ..." --files .planning/...
```
```

**After (fixed):**
```
If `commit_docs` is true (from init JSON):
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "docs(...): ..." --files .planning/...
```
```

Specific files and locations:

1. **agents/nick-executor.md** (~line 473-476, `<final_commit>` section): Wrap the single commit call. The executor already extracts `commit_docs` from init JSON at line 116.

2. **agents/nick-planner.md** (2 commits):
   - ~line 799 (revision mode Step 6): Wrap commit call. Planner extracts `commit_docs` at line 841.
   - ~line 1064 (git_commit step): Wrap commit call.

3. **agents/nick-phase-researcher.md** (~line 387, Step 6): The step is already titled "Commit Research (optional)" -- change the guard text. Currently says just the command. Add: `If commit_docs is true (from init JSON):` before the bash block. The researcher extracts `commit_docs` at line 317.

4. **agents/nick-debugger.md** (~line 1000-1003): Already has text "Then commit planning docs via CLI (respects commit_docs config automatically):" -- change this to an explicit conditional: `If commit_docs is true (from state load):` before the bash block.

5. **agents/nick-research-synthesizer.md** (~line 131): Wrap commit call. Note: this agent does NOT currently extract commit_docs from init. Add a note: `If commit_docs is true (from orchestrator context):` before the bash block.

6-8. **skills/nick-execute-phase/references/agents/nick-executor.md**, **skills/nick-plan-phase/references/agents/nick-planner.md**, **skills/nick-plan-phase/references/agents/nick-phase-researcher.md**: Apply the exact same changes as their agents/ counterparts (these are paired copies).

IMPORTANT: Do NOT change the bash command itself. Only add the conditional text wrapper. Do NOT change any other content in these files. Keep the existing warning notes (like "commit_docs controls git only, NOT file writing") -- they are still correct and complementary.
  </action>
  <verify>
Run: `grep -c "commit_docs.*true" agents/nick-executor.md agents/nick-planner.md agents/nick-phase-researcher.md agents/nick-debugger.md agents/nick-research-synthesizer.md skills/nick-execute-phase/references/agents/nick-executor.md skills/nick-plan-phase/references/agents/nick-planner.md skills/nick-plan-phase/references/agents/nick-phase-researcher.md`

Each file should show at least 1 match for the new conditional guard text. Total should be 10 (nick-planner has 2 each = 4, plus 6 other files = 10).

Also verify no commit call in these files is left unguarded:
`grep -B1 "gsd-tools.cjs commit" agents/nick-*.md skills/*/references/agents/nick-*.md`
Every commit line should have a conditional guard on the line(s) before it.
  </verify>
  <done>All 8 agent files have conditional `commit_docs` guards around every .planning commit call. No unguarded commit calls remain in agent files.</done>
</task>

<task type="auto">
  <name>Task 2: Add commit_docs guards to all workflow and skill files (17 files)</name>
  <files>
    get-shit-done/workflows/quick.md
    get-shit-done/workflows/execute-phase.md
    get-shit-done/workflows/execute-plan.md
    get-shit-done/workflows/new-project.md
    get-shit-done/workflows/discuss-phase.md
    get-shit-done/workflows/verify-work.md
    get-shit-done/workflows/map-codebase.md
    get-shit-done/workflows/new-milestone.md
    get-shit-done/workflows/pause-work.md
    get-shit-done/workflows/add-todo.md
    get-shit-done/workflows/complete-milestone.md
    get-shit-done/workflows/remove-phase.md
    get-shit-done/workflows/plan-milestone-gaps.md
    get-shit-done/workflows/check-todos.md
    get-shit-done/workflows/diagnose-issues.md
    skills/nick-execute-phase/SKILL.md
    skills/nick-review/SKILL.md
  </files>
  <action>
For each workflow/skill file, wrap every `gsd-tools.cjs commit` call that commits .planning files with a `commit_docs` conditional guard. Same pattern as Task 1:

```
If `commit_docs` is true (from init JSON):
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "..." --files .planning/...
```
```

All these workflows already parse `commit_docs` from their init JSON (confirmed via grep). The specific files and commit counts:

1. **get-shit-done/workflows/quick.md** (line 194): 1 commit -- wrap it.
2. **get-shit-done/workflows/execute-phase.md** (lines 274, 355): 2 commits -- wrap both.
3. **get-shit-done/workflows/execute-plan.md** (lines 392, 407): 2 commits -- wrap both. Note line 407 uses `--amend` for codebase map update; still needs the guard.
4. **get-shit-done/workflows/new-project.md** (lines 215, 365, 763, 896): 4 commits -- wrap all. These are project initialization commits (PROJECT.md, config.json, REQUIREMENTS.md, ROADMAP.md). They're part of project setup so they SHOULD typically be committed, but the guard still applies since the user has explicitly set commit_docs=false.
5. **get-shit-done/workflows/discuss-phase.md** (lines 392, 410): 2 commits -- wrap both.
6. **get-shit-done/workflows/verify-work.md** (line 295): 1 commit -- wrap it.
7. **get-shit-done/workflows/map-codebase.md** (line 265): 1 commit -- wrap it.
8. **get-shit-done/workflows/new-milestone.md** (lines 74, 249, 324): 3 commits -- wrap all.
9. **get-shit-done/workflows/pause-work.md** (line 95): 1 commit -- wrap it.
10. **get-shit-done/workflows/add-todo.md** (line 121): 1 commit -- wrap it. Remove or update the existing "Tool respects commit_docs config and gitignore automatically." note to instead say the guard handles it.
11. **get-shit-done/workflows/complete-milestone.md** (line 565): 1 commit -- wrap it.
12. **get-shit-done/workflows/remove-phase.md** (line 105): 1 commit -- wrap it.
13. **get-shit-done/workflows/plan-milestone-gaps.md** (line 135): 1 commit -- wrap it.
14. **get-shit-done/workflows/check-todos.md** (line 157): 1 commit -- wrap it. Same note update as add-todo.md.
15. **get-shit-done/workflows/diagnose-issues.md** (line 161): 1 commit -- wrap it.
16. **skills/nick-execute-phase/SKILL.md** (line 173): 1 commit -- wrap it.
17. **skills/nick-review/SKILL.md** (line 95): 1 commit -- wrap it.

IMPORTANT: Do NOT change the bash commands themselves. Only add the conditional text wrapper. Do NOT change any other content in these files.

For add-todo.md and check-todos.md, update the existing "Tool respects commit_docs config and gitignore automatically." note to: "Skipped when commit_docs is false."
  </action>
  <verify>
Run: `grep -c "commit_docs.*true" get-shit-done/workflows/quick.md get-shit-done/workflows/execute-phase.md get-shit-done/workflows/execute-plan.md get-shit-done/workflows/new-project.md get-shit-done/workflows/discuss-phase.md get-shit-done/workflows/verify-work.md get-shit-done/workflows/map-codebase.md get-shit-done/workflows/new-milestone.md get-shit-done/workflows/pause-work.md get-shit-done/workflows/add-todo.md get-shit-done/workflows/complete-milestone.md get-shit-done/workflows/remove-phase.md get-shit-done/workflows/plan-milestone-gaps.md get-shit-done/workflows/check-todos.md get-shit-done/workflows/diagnose-issues.md skills/nick-execute-phase/SKILL.md skills/nick-review/SKILL.md`

Each file should show at least 1 match. Files with multiple commits (execute-phase: 2, execute-plan: 2, new-project: 4, discuss-phase: 2, new-milestone: 3) should show correspondingly higher counts.

Total expected: 24 new conditional guards across 17 files.

Also verify no commit call is left unguarded:
`grep -B1 "gsd-tools.cjs commit" get-shit-done/workflows/*.md skills/*/SKILL.md`
Every commit line should have a conditional guard on the line(s) before it.
  </verify>
  <done>All 17 workflow/skill files have conditional `commit_docs` guards around every .planning commit call. No unguarded commit calls remain.</done>
</task>

<task type="auto">
  <name>Task 3: Update git-integration reference docs and verify completeness</name>
  <files>
    get-shit-done/references/git-integration.md
  </files>
  <action>
Update `get-shit-done/references/git-integration.md` to document the commit_docs guard pattern. This file is the authoritative reference for git commit conventions in GSD.

Add a new section after `</commit_strategy_rationale>` and before the closing content (or at an appropriate location near the commit formats):

```markdown
<commit_docs_guard>

## Planning File Commits

All `.planning/` commit calls are conditional on `commit_docs` being true. This is a user config setting (`config.json`) that controls whether planning artifacts are tracked in git.

**Pattern for agents and workflows:**

```
If `commit_docs` is true (from init JSON):
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "..." --files .planning/...
```
```

**When `commit_docs` is false:** Skip the commit call entirely. Do NOT call `gsd-tools.cjs commit` and then handle the "skipped" response -- the call should never be made.

**Why guard at the prompt level:** `gsd-tools.cjs commit` already returns `skipped_commit_docs_false` or `skipped_gitignored` when .planning is not committable. But LLM agents may interpret "skipped" as failure and attempt workarounds (raw git commands, retries). Pre-guarding prevents this entirely.

**Note:** `commit_docs` controls git only, NOT file writing. Always write .planning files to disk regardless of this setting.

</commit_docs_guard>
```

Also verify completeness across the entire codebase:
- Run a final grep for ALL `gsd-tools.cjs commit` calls that reference `.planning` across the repo
- Confirm every single one now has a `commit_docs` guard
- Report any stragglers
  </action>
  <verify>
Run: `grep -rn "gsd-tools.cjs commit" --include="*.md" . | grep -i planning`
Every line returned should have a `commit_docs` guard within 1-3 lines before it in the source file. Spot-check 5 random files to confirm.

Also: `grep "commit_docs_guard" get-shit-done/references/git-integration.md` should return matches confirming the new section exists.
  </verify>
  <done>git-integration.md documents the commit_docs guard pattern. Final audit confirms no unguarded .planning commit calls exist anywhere in the codebase.</done>
</task>

</tasks>

<verification>
1. `grep -rn "gsd-tools.cjs commit" --include="*.md" agents/ skills/ get-shit-done/workflows/ get-shit-done/references/` -- list all commit calls
2. For each commit call that references .planning files, verify `commit_docs` guard exists within 1-3 lines before it
3. Zero unguarded .planning commit calls across all 26 files
</verification>

<success_criteria>
- Every `gsd-tools.cjs commit` call in agents and workflows that commits .planning files is wrapped with `If commit_docs is true` guard
- git-integration.md documents the pattern
- No behavioral change when commit_docs is true (commands run as before)
- When commit_docs is false, agents/workflows skip the commit call entirely instead of calling it and handling "skipped"
</success_criteria>

<output>
After completion, create `.planning/quick/7-fix-planners-and-executors-that-try-to-c/7-SUMMARY.md`
</output>
