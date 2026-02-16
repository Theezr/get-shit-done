---
phase: quick-1
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - agents/nick-planner.md
  - skills/nick-plan-phase/references/agents/nick-planner.md
  - agents/nick-executor.md
  - skills/nick-execute-phase/references/agents/nick-executor.md
autonomous: true

must_haves:
  truths:
    - "Planner always does Context7 verification for key APIs even when RESEARCH.md exists"
    - "Executor always does Context7 verification before implementation even when RESEARCH.md exists"
    - "RESEARCH.md is treated as helpful context, not a skip condition"
  artifacts:
    - path: "agents/nick-planner.md"
      provides: "Planner agent definition"
      contains: "Context7.*always"
    - path: "skills/nick-plan-phase/references/agents/nick-planner.md"
      provides: "Installed planner agent definition"
      contains: "Context7.*always"
    - path: "agents/nick-executor.md"
      provides: "Executor agent definition"
      contains: "Context7.*always"
    - path: "skills/nick-execute-phase/references/agents/nick-executor.md"
      provides: "Installed executor agent definition"
      contains: "Context7.*always"
  key_links:
    - from: "agents/nick-planner.md"
      to: "skills/nick-plan-phase/references/agents/nick-planner.md"
      via: "Same content synced to skill copy"
      pattern: "Context7 verification is ALWAYS performed"
    - from: "agents/nick-executor.md"
      to: "skills/nick-execute-phase/references/agents/nick-executor.md"
      via: "Same content synced to skill copy"
      pattern: "Context7 verification is ALWAYS performed"
---

<objective>
Make nick-plan-phase and nick-execute-phase use Context7 more proactively by removing the
"RESEARCH.md-first" skip pattern. RESEARCH.md becomes helpful context (what was previously
found), not a reason to skip fresh Context7 verification. This ensures plans and execution
always reference current API signatures even if the researcher already looked at the library.

Purpose: RESEARCH.md can be stale or incomplete. Always verifying via Context7 catches API
changes, fills gaps, and produces more accurate plans and implementations.

Output: Updated agent definitions in both repo source and skill copies.
</objective>

<execution_context>
@/Users/nick/.claude/get-shit-done/workflows/execute-plan.md
@/Users/nick/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@agents/nick-planner.md
@skills/nick-plan-phase/references/agents/nick-planner.md
@agents/nick-executor.md
@skills/nick-execute-phase/references/agents/nick-executor.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Remove RESEARCH.md-first skip pattern from nick-planner</name>
  <files>
    agents/nick-planner.md
    skills/nick-plan-phase/references/agents/nick-planner.md
  </files>
  <action>
  In both planner files, make these targeted text edits:

  **Edit A -- discovery_levels section (around line 112 in skill copy, line 110 in agents/ copy):**
  Replace the "RESEARCH.md-first rule" paragraph:
  ```
  **RESEARCH.md-first rule:** Before any Context7 query, check if `{phase_dir}/*-RESEARCH.md` already covers the library/topic. If RESEARCH.md has a HIGH-confidence entry for that library, skip the Context7 lookup entirely. This avoids double-querying what the researcher already verified.
  ```
  With:
  ```
  **Context7-always rule:** Context7 verification is ALWAYS performed for key APIs referenced in the plan, even when RESEARCH.md exists. RESEARCH.md provides useful context (what the researcher found, confidence levels, known patterns) but is NOT a reason to skip fresh verification. APIs change, RESEARCH.md may be stale or incomplete, and a quick Context7 check costs minimal context compared to implementing against a wrong API.
  ```

  **Edit B -- gather_phase_context step (around line 1035 in skill copy, line 937 in agents/ copy):**
  Replace the "MCP-verified findings" paragraph:
  ```
  **MCP-verified findings:** RESEARCH.md now contains Context7-verified API details with confidence tags (HIGH/MEDIUM/LOW). When RESEARCH.md has HIGH-confidence findings for a library, trust those findings for planning rather than re-querying Context7. Only use Context7 directly for Level 1 quick verification of libraries NOT covered in RESEARCH.md.
  ```
  With:
  ```
  **MCP-verified findings:** RESEARCH.md contains Context7-verified API details with confidence tags (HIGH/MEDIUM/LOW). Use these as useful context when planning -- they show what the researcher found and at what confidence. However, ALWAYS do a quick Context7 verification for key APIs you reference in task actions, regardless of RESEARCH.md coverage. This catches staleness, fills gaps the researcher missed, and ensures plan accuracy. The cost of a quick Context7 check is negligible compared to an executor implementing against a wrong API.
  ```

  NOTE: The agents/ copy does NOT have Edit B (it has a simpler version without the MCP-verified paragraph). For agents/nick-planner.md, only apply Edit A. Check that the text exists before editing -- the two copies have minor differences (e.g., "GSD planner" vs "planner").
  </action>
  <verify>
  Grep both files for "RESEARCH.md-first" -- should return zero matches.
  Grep both files for "Context7-always" -- should return matches.
  Grep both files for "skip the Context7 lookup entirely" -- should return zero matches.
  Grep both files for "trust those findings for planning rather than re-querying" -- should return zero matches.
  </verify>
  <done>
  Both planner files (agents/ and skills/) no longer instruct skipping Context7 when RESEARCH.md
  exists. Instead they instruct always performing Context7 verification with RESEARCH.md as context.
  </done>
</task>

<task type="auto">
  <name>Task 2: Remove RESEARCH.md-first skip pattern from nick-executor</name>
  <files>
    agents/nick-executor.md
    skills/nick-execute-phase/references/agents/nick-executor.md
  </files>
  <action>
  The executor file has the mcp_protocol section with the skip pattern. The agents/ repo copy
  does NOT have an mcp_protocol section (it was only added to the skill copy). Edit only the
  skill copy, then sync the mcp_protocol section to the agents/ copy.

  **In skills/nick-execute-phase/references/agents/nick-executor.md:**

  **Edit A -- mcp_protocol Step 1 (lines 22-27):**
  Replace:
  ```
  ### Step 1: Read RESEARCH.md (if exists)

  Before any Context7 calls, check if the researcher already documented the libraries:
  - Read `$PHASE_DIR/*-RESEARCH.md` (the phase's research file)
  - If RESEARCH.md covers the library's API you need, skip Context7 for that library
  - This avoids double-querying what the researcher already verified
  ```
  With:
  ```
  ### Step 1: Read RESEARCH.md (if exists)

  Read `$PHASE_DIR/*-RESEARCH.md` for context on what the researcher found:
  - Note confidence levels (HIGH/MEDIUM/LOW) and documented API patterns
  - Use this as helpful context for Step 2, NOT as a reason to skip verification
  - RESEARCH.md may be stale or incomplete -- always verify via Context7
  ```

  **Edit B -- mcp_protocol Step 2 (lines 29-31):**
  Replace:
  ```
  For each library referenced in the task's `<action>` that is NOT covered in RESEARCH.md:
  ```
  With:
  ```
  For each library referenced in the task's `<action>`, ALWAYS verify via Context7 (RESEARCH.md is context, not a skip condition):
  ```

  **Edit C -- mcp_degradation Fallback Behavior (lines 67-71):**
  Replace:
  ```
  When Context7 is unavailable:
  - Step 1 (Read RESEARCH.md) is unchanged -- always do this
  - Step 2 (Context7 verify) is SKIPPED -- proceed with plan instructions
  - Step 3 (Load best-practice skills) is unchanged -- skill files are local, not MCP-dependent
  ```
  With:
  ```
  When Context7 is unavailable:
  - Step 1 (Read RESEARCH.md) becomes PRIMARY source -- rely on researcher findings since live verification is unavailable
  - Step 2 (Context7 verify) is SKIPPED -- proceed with RESEARCH.md findings and plan instructions
  - Step 3 (Load best-practice skills) is unchanged -- skill files are local, not MCP-dependent
  - Tag unverified APIs in SUMMARY.md: "API not verified (Context7 unavailable, used RESEARCH.md)"
  ```

  **For agents/nick-executor.md:** This file currently has NO mcp_protocol section. Add the
  entire updated mcp_protocol section (from the skill copy, after edits) immediately after
  the closing `</role>` tag and before `<execution_flow>`. This brings the repo copy in sync.
  Also add `mcp__context7__*` to the tools line in frontmatter (currently missing from agents/ copy).
  </action>
  <verify>
  Grep skill copy for "skip Context7 for that library" -- should return zero matches.
  Grep skill copy for "NOT covered in RESEARCH.md" -- should return zero matches.
  Grep skill copy for "NOT as a reason to skip" -- should return a match.
  Grep agents/ copy for "mcp_protocol" -- should return matches (newly added section).
  Grep agents/ copy for "mcp__context7" -- should appear in tools line.
  </verify>
  <done>
  Both executor files instruct always verifying via Context7 before implementation. RESEARCH.md
  is used as context (not skip condition) when Context7 is available, and as primary fallback
  when Context7 is unavailable. The agents/ repo copy now has the mcp_protocol section matching
  the skill copy.
  </done>
</task>

<task type="auto">
  <name>Task 3: Sync updated files to installed location</name>
  <files>
    ~/.claude/skills/nick-plan-phase/references/agents/nick-planner.md
    ~/.claude/skills/nick-execute-phase/references/agents/nick-executor.md
  </files>
  <action>
  Copy the updated skill reference files to the installed location where Claude Code actually
  loads them at runtime:

  ```bash
  cp skills/nick-plan-phase/references/agents/nick-planner.md ~/.claude/skills/nick-plan-phase/references/agents/nick-planner.md
  cp skills/nick-execute-phase/references/agents/nick-executor.md ~/.claude/skills/nick-execute-phase/references/agents/nick-executor.md
  ```

  Verify the installed copies match the repo copies:
  ```bash
  diff skills/nick-plan-phase/references/agents/nick-planner.md ~/.claude/skills/nick-plan-phase/references/agents/nick-planner.md
  diff skills/nick-execute-phase/references/agents/nick-executor.md ~/.claude/skills/nick-execute-phase/references/agents/nick-executor.md
  ```
  </action>
  <verify>
  diff commands return no output (files are identical).
  </verify>
  <done>
  Installed skill copies at ~/.claude/skills/ match the updated repo copies. Changes are
  live for next nick-plan-phase or nick-execute-phase invocation.
  </done>
</task>

</tasks>

<verification>
After all tasks complete:
1. Grep entire repo for "skip the Context7 lookup entirely" -- zero matches
2. Grep entire repo for "skip Context7 for that library" -- zero matches
3. Grep entire repo for "trust those findings for planning rather than re-querying" -- zero matches
4. Grep entire repo for "Context7-always" -- matches in planner files
5. Grep entire repo for "NOT as a reason to skip" -- matches in executor files
6. Installed copies match repo copies (diff returns clean)
</verification>

<success_criteria>
- No agent file tells Claude to skip Context7 when RESEARCH.md exists
- All agent files instruct always performing Context7 verification for key APIs
- RESEARCH.md is described as "helpful context" or "additional context", never as a replacement for live verification
- Changes are live in both repo and installed locations
- Fallback behavior still works correctly when Context7 is unavailable (RESEARCH.md becomes primary in that case)
</success_criteria>

<output>
After completion, create `.planning/quick/1-make-nick-plan-phase-and-nick-execute-ph/1-01-SUMMARY.md`
</output>
