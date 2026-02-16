---
phase: quick-5
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - skills/nick-plan-phase/SKILL.md
  - skills/nick-execute-phase/SKILL.md
  - skills/nick-verify-work/SKILL.md
  - skills/nick-review/SKILL.md
autonomous: true
must_haves:
  truths:
    - "Each skill captures a start timestamp at initialization and computes wall time at its final step"
    - "Each skill tracks how many Task() agent spawns it made during execution"
    - "Plan-phase and execute-phase skills track Context7 MCP call count (resolve-library-id + query-docs)"
    - "Each skill appends a metrics row to an Execution Metrics table in STATE.md at completion"
    - "Files modified count is derived from git diff between start and end of skill execution"
  artifacts:
    - path: "skills/nick-plan-phase/SKILL.md"
      provides: "Plan-phase orchestrator with metrics collection"
      contains: "START_TIME"
    - path: "skills/nick-execute-phase/SKILL.md"
      provides: "Execute-phase orchestrator with metrics collection"
      contains: "START_TIME"
    - path: "skills/nick-verify-work/SKILL.md"
      provides: "Verify-work orchestrator with metrics collection"
      contains: "START_TIME"
    - path: "skills/nick-review/SKILL.md"
      provides: "Review orchestrator with metrics collection"
      contains: "START_TIME"
  key_links:
    - from: "Step 1 (all skills)"
      to: "Final step (all skills)"
      via: "START_TIME bash variable captured at init, used at end to compute wall time"
      pattern: "START_TIME"
    - from: "Final step (all skills)"
      to: ".planning/STATE.md"
      via: "Append metrics row to Execution Metrics table"
      pattern: "Execution Metrics"
---

<objective>
Add execution metrics tracking to all 4 custom skills (nick-plan-phase, nick-execute-phase, nick-verify-work, nick-review).

Purpose: Provide visibility into how long each skill invocation takes, how many agents it spawns, how many Context7 queries it makes, and how many files it modifies. This data accumulates in STATE.md for the user to review execution efficiency over time.

Output: All 4 SKILL.md files updated with start-time capture at init, counter tracking during execution, and metrics append to STATE.md at the final step.
</objective>

<execution_context>
@/Users/nick/.claude/get-shit-done/workflows/execute-plan.md
@/Users/nick/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@skills/nick-plan-phase/SKILL.md
@skills/nick-execute-phase/SKILL.md
@skills/nick-verify-work/SKILL.md
@skills/nick-review/SKILL.md
@.planning/STATE.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Add metrics to nick-plan-phase and nick-execute-phase</name>
  <files>skills/nick-plan-phase/SKILL.md, skills/nick-execute-phase/SKILL.md</files>
  <action>
Make three changes to each skill file. The pattern is identical but the specifics differ per skill.

**Pattern applied to BOTH files:**

**Change 1: Capture start time and initialize counters at Step 1**

In each skill's Step 1 (Initialize), immediately AFTER the `INIT=$(node ...)` bash block and BEFORE the "Parse from JSON" line, add:

```markdown
**Metrics tracking:** Capture start time and initialize counters for end-of-skill metrics.

\```bash
START_TIME=$(date +%s)
START_COMMIT=$(git rev-parse HEAD 2>/dev/null || echo "none")
\```

Initialize mental counters (track these as you orchestrate -- do NOT use bash variables since shell state does not persist between Bash calls):
- `agents_spawned`: 0 (increment each time you call Task())
- `c7_queries`: 0 (increment each time you call mcp__context7__resolve-library-id or mcp__context7__query-docs)
```

**Change 2: Track counts inline**

Add a brief reminder comment at each Task() spawn point and Context7 call site:

- For nick-plan-phase SKILL.md:
  - Step 3 researcher Task() spawn(s): Add comment `<!-- metrics: +1 agent (or +N agents if parallel) -->`
  - Step 4 planner Task() spawn: Add comment `<!-- metrics: +1 agent -->`
  - Step 5 checker Task() spawn: Add comment `<!-- metrics: +1 agent -->`
  - Step 6 revision loop planner + checker: Add comment `<!-- metrics: +1 agent each iteration -->`
  - Note: nick-plan-phase has Context7 in its allowed-tools but the orchestrator itself does not make C7 calls (agents do). So c7_queries for the orchestrator should be tracked only if the orchestrator itself calls Context7 directly (unlikely but possible). Agents' C7 usage is not tracked here -- that would require agent-side reporting which is out of scope.

- For nick-execute-phase SKILL.md:
  - Step 4b executor Task() spawns: Add comment `<!-- metrics: +N agents (one per plan in wave) -->`
  - Step 5 verifier Task() spawn: Add comment `<!-- metrics: +1 agent -->`
  - Step 4e checkpoint continuation: Add comment `<!-- metrics: +1 agent per continuation -->`
  - The orchestrator may call Context7 directly -- track if it does.

**Change 3: Append metrics to STATE.md at the final step**

For nick-plan-phase, at the END of Step 7 (Present Results), add a new subsection:

```markdown
**Metrics:** Append execution metrics to STATE.md.

\```bash
END_TIME=$(date +%s)
WALL_TIME=$((END_TIME - START_TIME))
FILES_MODIFIED=$(git diff --name-only $START_COMMIT HEAD 2>/dev/null | wc -l | tr -d ' ')
\```

Format wall time as `Xm Ys` (e.g., `3m 45s` or `0m 30s`). Construct a metrics row:

```
| plan-phase | Phase {X} | {YYYY-MM-DD} | {Xm Ys} | {agents_spawned} | {c7_queries} | {files_modified} |
```

Read STATE.md. If it does not contain `### Execution Metrics`, append this section:

```markdown

### Execution Metrics

| Skill | Phase/Task | Date | Wall Time | Agents | C7 Queries | Files Modified |
|-------|-----------|------|-----------|--------|------------|----------------|
```

Then append the metrics row after the table header (or after the last existing row).

Write the updated STATE.md.
```

For nick-execute-phase, do the same at the END of Step 7 (Offer Next), with `execute-phase` as the Skill column value.

**IMPORTANT implementation notes:**
- `START_TIME` and `START_COMMIT` are captured via Bash but shell state does not persist between Bash calls. The orchestrator must re-capture END_TIME in the same Bash block that computes the diff, OR store START_TIME/START_COMMIT values in the prompt context (the orchestrator remembers them as text from the output of the first Bash call).
- The orchestrator tracks `agents_spawned` and `c7_queries` mentally as it goes -- these are just integers it increments and remembers.
- `FILES_MODIFIED` uses git diff between the start commit and current HEAD at the end.
  </action>
  <verify>
Read both modified SKILL.md files. For each, confirm:
1. Step 1 has START_TIME and START_COMMIT capture bash block after INIT
2. Step 1 has mental counter initialization instructions
3. Task() spawn points have metrics tracking comments
4. Final step (Step 7) has metrics computation bash block and STATE.md append logic
5. Metrics table format matches: `| Skill | Phase/Task | Date | Wall Time | Agents | C7 Queries | Files Modified |`
6. Existing skill logic is completely preserved -- only additions, no removals
  </verify>
  <done>Both nick-plan-phase/SKILL.md and nick-execute-phase/SKILL.md capture start time at init, track agent/C7 counts during orchestration, and append a metrics row to STATE.md at completion.</done>
</task>

<task type="auto">
  <name>Task 2: Add metrics to nick-verify-work and nick-review</name>
  <files>skills/nick-verify-work/SKILL.md, skills/nick-review/SKILL.md</files>
  <action>
Apply the same 3-change pattern from Task 1 to both files, with these skill-specific differences:

**nick-verify-work/SKILL.md:**

- **Change 1 (Step 1):** Add START_TIME/START_COMMIT capture and counter init after the INIT bash block. Note: verify-work does NOT have Context7 in its allowed-tools, so `c7_queries` will always be 0. Still include the counter for table consistency.
- **Change 2:** Step 4 browser-tester Task() spawn: Add comment `<!-- metrics: +1 agent -->`
- **Change 3 (Step 6 -- Present Results and Offer Next):** Append metrics to STATE.md with `verify-work` as the Skill column value. Add the metrics computation and STATE.md append logic at the END of Step 6, after the "Offer /gsd:review" line.

**nick-review/SKILL.md:**

- **Change 1 (Step 1):** Add START_TIME/START_COMMIT capture and counter init after the INIT bash block. Note: nick-review does NOT have Context7 in its allowed-tools, so `c7_queries` will always be 0. Still include the counter for table consistency.
- **Change 2:** Step 3 code-reviewer Task() spawn: Add comment `<!-- metrics: +1 agent -->`
- **Change 3 (Step 6 -- Present Results):** Append metrics to STATE.md with `review` as the Skill column value. Add the metrics computation and STATE.md append logic at the END of Step 6, after the "Offer gap closure path" line.

**Same implementation notes as Task 1:**
- START_TIME captured in first Bash call, remembered by orchestrator as text
- END_TIME and git diff computed in a single Bash block at the final step
- Agent count is trivial for these skills (typically 1 agent each)
- C7 queries will be 0 for both (neither skill has Context7 tools)
- FILES_MODIFIED from git diff START_COMMIT..HEAD
  </action>
  <verify>
Read both modified SKILL.md files. For each, confirm:
1. Step 1 has START_TIME and START_COMMIT capture bash block after INIT
2. Step 1 has mental counter initialization instructions
3. Task() spawn points have metrics tracking comments
4. Final step has metrics computation bash block and STATE.md append logic
5. Metrics table format matches the same format as Task 1
6. Existing skill logic is completely preserved -- only additions, no removals
  </verify>
  <done>Both nick-verify-work/SKILL.md and nick-review/SKILL.md capture start time at init, track agent count during orchestration, and append a metrics row to STATE.md at completion.</done>
</task>

</tasks>

<verification>
- Read all 4 modified SKILL.md files and verify each has the 3 changes (start capture, inline tracking, end metrics append)
- Verify the metrics table format is identical across all 4 skills
- Verify no existing orchestration logic was altered or removed
- Verify the STATE.md append logic handles both cases: creating the Execution Metrics section if missing, and appending to existing table
- Check that START_TIME capture appears before any Task() spawns in each skill
</verification>

<success_criteria>
- All 4 SKILL.md files have START_TIME/START_COMMIT capture at initialization
- All 4 SKILL.md files have agent spawn tracking comments at Task() call sites
- All 4 SKILL.md files append a metrics row to STATE.md at their final step
- Metrics row format is consistent: `| {skill} | Phase {X} | {date} | {wall_time} | {agents} | {c7_queries} | {files_modified} |`
- STATE.md Execution Metrics table is created if it does not exist, appended to if it does
- Existing skill orchestration logic is 100% preserved
</success_criteria>

<output>
After completion, create `.planning/quick/5-add-execution-metrics-to-state-md-via-sk/5-SUMMARY.md`
</output>
