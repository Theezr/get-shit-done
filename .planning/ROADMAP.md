# Roadmap: GSD Workflow Customization

**Created:** 2026-02-15
**Depth:** quick
**Phases:** 4
**Requirements:** 19 mapped

### Phase 1: Core Skills & Token Architecture ✓

**Status:** Complete (2026-02-15)
**Verified:** 11/11 must-haves passed

**Goal:** Convert plan-phase and execute-phase to SKILL.md format with path-based context passing, progressive disclosure, and agent definitions in references/ directories.

**Requirements:** SKILL-01, SKILL-02, SKILL-05, TOKEN-01, TOKEN-02, PIPE-02, PIPE-03, PIPE-06

**Success Criteria:**
1. `gsd-plan-phase/SKILL.md` exists with frontmatter (name, description, allowed-tools), lean body (<1,500 tokens), and agent prompts in `references/`
2. `gsd-execute-phase/SKILL.md` exists with the same structure; executor agents receive file paths (not content) and read files themselves
3. Agent definition files live in skill `references/agents/` directories, not as standalone agent `.md` files
4. Planner skill restricts to read-only + write plan (no code implementation); executor skill restricts to no-commit (allowed-tools enforces separation)
5. A plan-phase invocation uses path-based context passing (agents read STATE.md, ROADMAP.md, REQUIREMENTS.md themselves instead of receiving injected content)

**Plans:** 3 plans

Plans:
- [x] 01-01-PLAN.md -- Create gsd-plan-phase skill (SKILL.md + 3 agent definitions in references/agents/)
- [x] 01-02-PLAN.md -- Create gsd-execute-phase skill (SKILL.md + 2 agent definitions in references/agents/)
- [x] 01-03-PLAN.md -- Gap closure: standardize files_to_read pattern and fix gsd-tools extension in execute-phase

---

### Phase 2: MCP-Powered Planning & Execution ✓

**Status:** Complete (2026-02-16)
**Verified:** 10/10 must-haves passed

**Goal:** Integrate Context7, microsoft-docs, and shadcn MCPs into the plan and execute skills so that every API reference is looked up from current documentation rather than generated from training data.

**Requirements:** MCP-01, MCP-02, MCP-03, MCP-04, MCP-05

**Success Criteria:**
1. Plan skill performs Context7 2-step lookup (resolve-library-id then query-docs) for every library the implementation will touch, and writes findings to RESEARCH.md
2. Plan skill uses microsoft-docs MCP (search, code-sample-search, fetch) when Azure features are involved
3. Plan skill uses shadcn MCP (getComponents, getComponent) when UI component planning is needed
4. Execute skill verifies the plan's API references via Context7 before implementing (catches stale or incorrect API signatures from the plan)
5. Execute skill loads relevant best-practice skills (vercel-react, nestjs, owasp, frontend-design) before writing code

**Plans:** 2 plans

Plans:
- [x] 02-01-PLAN.md -- Add MCP protocols to plan-phase researcher + update SKILL.md allowed-tools + planner/checker awareness
- [x] 02-02-PLAN.md -- Add proactive API verification and best-practice skill loading to execute-phase executor + update SKILL.md

---

### Phase 3: Verification & Review Pipeline ✓

**Status:** Complete (2026-02-16)
**Verified:** 11/11 must-haves passed

**Goal:** Build the verify and review skills that complete the plan-execute-verify-review pipeline, with Chrome DevTools runtime testing and best-practice code review with commit-on-PASS.

**Requirements:** SKILL-03, SKILL-04, MCP-06, MCP-07, PIPE-01, PIPE-04, PIPE-05

**Success Criteria:**
1. Verify skill uses Chrome DevTools MCP to perform snapshot-interact-verify testing on the running application (navigate, snapshot, click/fill, check console/network)
2. Verify skill reports findings without modifying source files (read-only + write report)
3. Review skill loads best-practice skills, runs IDE diagnostics MCP for TypeScript errors, and produces a structured review with severity levels
4. Review skill commits only after review result is PASS; no commit occurs on FAIL
5. Full pipeline runs end-to-end: plan produces PLAN.md, execute produces code + SUMMARY.md, verify produces VERIFICATION.md, review produces REVIEW.md and commits

**Plans:** 2 plans

Plans:
- [x] 03-01-PLAN.md -- Create gsd-verify-work skill (SKILL.md + browser-tester agent for Chrome DevTools runtime testing)
- [x] 03-02-PLAN.md -- Create gsd-review skill (SKILL.md + code-reviewer agent for quality review + commit-on-PASS)

---

### Phase 4: Resilience & Optimization ✓

**Status:** Complete (2026-02-16)
**Verified:** 17/17 must-haves passed

**Goal:** Harden all skills with graceful MCP degradation, conditional agent prompt loading, gsd-tools.js phase-specific extraction, and end-to-end pipeline validation under real project conditions.

**Requirements:** MCP-08, TOKEN-03, TOKEN-04, TOKEN-05

**Success Criteria:**
1. All MCP-using skills handle unavailability gracefully: pre-flight availability check, fallback to WebSearch for docs, confidence tagging (HIGH = MCP-verified, MEDIUM = WebSearch, LOW = training data)
2. Research findings are written to RESEARCH.md once; downstream agents (planner, executor) read the file instead of re-querying MCPs
3. `gsd-tools.js` supports phase-specific extraction (e.g., `requirements get-phase 3` returns only that phase's requirements, `roadmap get-phase 3 --format section` returns only that section)
4. Agent prompts split into core + conditional sections: TDD execution, checkpoint protocol, auth gates loaded only when the plan requires them (not on every invocation)

**Plans:** 3 plans

Plans:
- [x] 04-01-PLAN.md -- Add phase-specific extraction to gsd-tools.js (requirements get-phase, roadmap --format, plan index type)
- [x] 04-02-PLAN.md -- Add MCP degradation to all agent definitions + RESEARCH.md-first consistency in planner
- [x] 04-03-PLAN.md -- Split executor into core + checkpoints + TDD, conditional loading in orchestrator

---

## Coverage

| Requirement | Phase | Description |
|-------------|-------|-------------|
| SKILL-01 | Phase 1 | plan-phase as proper SKILL.md |
| SKILL-02 | Phase 1 | execute-phase as proper SKILL.md |
| SKILL-03 | Phase 3 | verify skill with Chrome DevTools |
| SKILL-04 | Phase 3 | review skill with best-practice loading |
| SKILL-05 | Phase 1 | Agent defs in references/ directories |
| MCP-01 | Phase 2 | Context7 in plan skill |
| MCP-02 | Phase 2 | microsoft-docs in plan skill |
| MCP-03 | Phase 2 | shadcn in plan skill |
| MCP-04 | Phase 2 | Context7 verification in execute skill |
| MCP-05 | Phase 2 | Best-practice skills in execute skill |
| MCP-06 | Phase 3 | Chrome DevTools in verify skill |
| MCP-07 | Phase 3 | IDE diagnostics in review skill |
| MCP-08 | Phase 4 | Graceful MCP degradation |
| TOKEN-01 | Phase 1 | Path-based context passing |
| TOKEN-02 | Phase 1 | Lean SKILL.md bodies + references/ |
| TOKEN-03 | Phase 4 | Research caching in RESEARCH.md |
| TOKEN-04 | Phase 4 | gsd-tools.js phase-specific extraction |
| TOKEN-05 | Phase 4 | Conditional agent prompt sections |
| PIPE-01 | Phase 3 | Full plan-execute-verify-review pipeline |
| PIPE-02 | Phase 1 | Planner never implements code |
| PIPE-03 | Phase 1 | Executor never commits |
| PIPE-04 | Phase 3 | Verifier never modifies source |
| PIPE-05 | Phase 3 | Reviewer commits only after PASS |
| PIPE-06 | Phase 1 | allowed-tools enforces role separation |

**Coverage: 19/19 (100%)**

*Note: PIPE-01 through PIPE-06 total 6 requirements but only 19 unique v1 requirements exist. PIPE-02, PIPE-03, and PIPE-06 are naturally addressed in Phase 1 as part of the skills conversion (allowed-tools in frontmatter enforces role separation). PIPE-01, PIPE-04, and PIPE-05 require the verify and review skills from Phase 3 to be fully realized.*

### Phase 5: Convert commands, agents, and skills from gsd- to nick- prefix ✓

**Status:** Complete (2026-02-16)
**Verified:** 16/16 must-haves passed

**Goal:** Rename all user-owned skills, agents, and hooks from gsd- to nick- prefix. Package-managed files (/gsd: commands, gsd-tools.js, workflows) remain unchanged.
**Depends on:** Phase 4
**Plans:** 3 plans

Plans:
- [x] 05-01-PLAN.md -- Rename repo skill directories and files from gsd- to nick- prefix with content updates
- [x] 05-02-PLAN.md -- Rename global agents and hooks from gsd- to nick- prefix, update settings.json
- [x] 05-03-PLAN.md -- Install nick-prefixed skills, delete old gsd- files, verify in Claude Code

### Phase 6: Fix nick skill paths: use nick: commands in flow recommendations, fix gsd-tools.cjs path and relative @-reference resolution

**Goal:** Fix all gsd-tools.js references to gsd-tools.cjs in nick-prefixed skills so that bash invocations resolve to the actual installed file. Research confirmed /gsd: command recommendations are correct (package commands exist) and @-reference resolution is likely working (test before changing).
**Depends on:** Phase 5
**Plans:** 1 plan

Plans:
- [ ] 06-01-PLAN.md -- Fix gsd-tools.js to gsd-tools.cjs extension in all nick skill files (repo + installed)

### Phase 7: Prototype-driven design in plan phase and mandatory frontend verification after execution

**Goal:** Add prototype-driven design to the planner (static HTML/CSS mockups alongside PLAN.md for frontend plans) and mandatory auto-triggered runtime verification in the execute-phase orchestrator when any plan has frontend work.
**Depends on:** Phase 6
**Plans:** 2 plans

Plans:
- [ ] 07-01-PLAN.md -- Add prototype creation to planner + frontend detection + prototype consistency check in plan checker
- [ ] 07-02-PLAN.md -- Add prototype-guided execution to executor + auto-verify frontend step in execute-phase orchestrator

### Phase 8: Combine review and verify-work skills into parallel execution

**Goal:** Combine the review and verify-work skills so they run in parallel after execution, reducing wall-clock time and eliminating the manual review step.
**Depends on:** Phase 7
**Plans:** 1 plan

Plans:
- [ ] 08-01-PLAN.md -- Parallel verify + review in execute-phase Step 4.5 with result matrix and decoupled review skill

---
*Roadmap created: 2026-02-15*
