---
phase: quick-2
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - skills/nick-execute-phase/SKILL.md
  - ~/.claude/skills/nick-execute-phase/SKILL.md
autonomous: true

must_haves:
  truths:
    - "Execute-phase runs plan waves and verifies phase goal -- nothing else"
    - "No auto verify-work or review agents spawn during execute-phase"
    - "Step 7 suggests manual /gsd:verify-work and /gsd:review as optional next steps"
  artifacts:
    - path: "skills/nick-execute-phase/SKILL.md"
      provides: "Execute-phase skill without auto verify+review"
      contains: "Step 5: Verify Phase Goal"
    - path: "~/.claude/skills/nick-execute-phase/SKILL.md"
      provides: "Installed copy synced with repo"
  key_links:
    - from: "skills/nick-execute-phase/SKILL.md"
      to: "~/.claude/skills/nick-execute-phase/SKILL.md"
      via: "file copy sync"
      pattern: "identical content"
---

<objective>
Remove auto verify+review (Step 4.5) from execute-phase, restoring manual invocation of /gsd:verify-work and /gsd:review.

Purpose: Phase 8 added automatic verify+review spawning after wave execution. This couples too many concerns into execute-phase. Revert to manual invocation -- user runs /gsd:verify-work and /gsd:review when they want them.
Output: Updated SKILL.md (repo + installed) with Step 4.5 removed and Step 7 updated.
</objective>

<execution_context>
@/Users/nick/.claude/get-shit-done/workflows/execute-plan.md
@/Users/nick/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@skills/nick-execute-phase/SKILL.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Remove Step 4.5 and update Step 7 to suggest manual verify/review</name>
  <files>skills/nick-execute-phase/SKILL.md, ~/.claude/skills/nick-execute-phase/SKILL.md</files>
  <action>
  In `skills/nick-execute-phase/SKILL.md`:

  1. DELETE the entire "## Step 4.5: Parallel Verify + Review" section (lines 125-223).
     This includes everything from `## Step 4.5:` up to but NOT including `## Step 5: Verify Phase Goal`.

  2. UPDATE "## Step 7: Offer Next" (currently the final section) to suggest manual verify and review.
     Replace the current Step 7 content with:

     ```
     ## Step 7: Offer Next

     Present options based on phase state:

     - **Optional quality checks** (suggest these first):
       - If phase has frontend work (`has_frontend: true` in any plan): suggest `/gsd:verify-work {X}` for runtime verification
       - Always suggest `/gsd:review {X}` for code quality review
     - If more phases: offer `/gsd:plan-phase {X+1}` (suggest `/clear` first for fresh context).
     - If milestone complete: offer `/gsd:complete-milestone`.
     ```

  3. SYNC to installed location:
     ```bash
     cp skills/nick-execute-phase/SKILL.md ~/.claude/skills/nick-execute-phase/SKILL.md
     ```

  Do NOT modify any other steps. Do NOT touch nick-review/SKILL.md (the review decoupling from Phase 8 Task 1 is a good change to keep).
  </action>
  <verify>
  - `grep -c "Step 4.5" skills/nick-execute-phase/SKILL.md` returns 0
  - `grep -c "Parallel Verify" skills/nick-execute-phase/SKILL.md` returns 0
  - `grep -c "verify-work" skills/nick-execute-phase/SKILL.md` returns 1 (only in Step 7 suggestion)
  - `grep -c "review" skills/nick-execute-phase/SKILL.md` returns 1 (only in Step 7 suggestion)
  - `diff skills/nick-execute-phase/SKILL.md ~/.claude/skills/nick-execute-phase/SKILL.md` shows no differences
  - Steps flow: 1 (Initialize) -> 2 (Branching) -> 3 (Discover) -> 4 (Execute Waves) -> 5 (Verify Phase Goal) -> 6 (Update Roadmap) -> 7 (Offer Next)
  </verify>
  <done>
  Execute-phase SKILL.md has no Step 4.5, no auto-spawning of verify-work or review agents.
  Step 7 suggests /gsd:verify-work and /gsd:review as optional manual next steps.
  Repo and installed copies are identical.
  </done>
</task>

</tasks>

<verification>
- Execute-phase flow goes directly from Step 4 (Execute Waves) to Step 5 (Verify Phase Goal)
- No references to nick-verify-work or nick-code-reviewer spawning remain in execute-phase
- Step 7 provides clear guidance for manual verify/review invocation
- Installed copy at ~/.claude/skills/nick-execute-phase/SKILL.md matches repo copy
</verification>

<success_criteria>
- Step 4.5 completely removed from skills/nick-execute-phase/SKILL.md
- Step 7 updated with manual verify/review suggestions
- Repo and installed SKILL.md are identical
- All other steps (1-6) unchanged
</success_criteria>

<output>
After completion, create `.planning/quick/2-revert-phase-8-remove-auto-verify-review/2-SUMMARY.md`
</output>
