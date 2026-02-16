# Gap Analysis: GSD Current State vs. Target Customized Workflow

**Date:** 2026-02-15
**Author:** Research Agent
**Scope:** Full inventory of GSD workflows (30), agents (11), CLI tool, templates, and references mapped against the user's target skills (nick:plan, nick:execute, nick:verify, nick:review, generate-workflow, update-workflow, test-flow)

---

## Key Findings

- The user's target skills collapse GSD's 30 workflows and 11 agents into **4 core skills** (plan, execute, verify, review) plus **3 supporting skills** (generate-workflow, update-workflow, test-flow). This is a radical simplification.
- GSD's phase-based project structure (init -> roadmap -> phases) is preserved in the target but the **file system moves from `.planning/` to `.pipeline/`** with a simpler layout: `plans/`, `plans/done/`, `builds/`, `reviews/`.
- The target skills introduce **MCP integration** as a first-class concern (context7, microsoft-docs, shadcn, chrome-devtools, IDE diagnostics) which GSD has none of.
- The target adds **Chrome DevTools runtime verification** via MCP, which is entirely absent from GSD's static code verification approach.
- The target adds a **dedicated code review skill** (nick:review) with skills-based best-practices loading (vercel-react, nestjs, owasp, frontend-design), which GSD lacks entirely.
- GSD's verification is static (grep-based stub detection, artifact existence checks). The target's verification is **runtime** (browser automation, snapshot-interact-verify pattern, network inspection, performance traces).
- GSD's `gsd-tools.cjs` CLI is not referenced in any target skill. The target uses standard Claude Code tools and MCP servers exclusively.
- GSD's elaborate state management (STATE.md, progress bars, metrics, session continuity) has **no equivalent** in the target skills. The pipeline is stateless between skill invocations.
- The target skills enforce a **strict separation of concerns**: planner never implements, executor never commits, verifier never modifies source, reviewer commits only after PASS. GSD blurs these boundaries (executor commits per-task).
- GSD's token-heavy features (4 parallel researchers, research synthesizer, plan checker revision loops, goal-backward derivation) are **removed** in favor of direct MCP lookups during planning.

---

## Workflow Mapping

### GSD Workflows -> Target State

| GSD Workflow | File | Target State | Target Skill | Rationale |
|---|---|---|---|---|
| `new-project` | `new-project.md` | **Remove** | N/A | Replaced by direct project setup. The user's target has no equivalent of GSD's elaborate questioning -> research -> requirements -> roadmap flow. Projects are initialized by creating `.pipeline/` and a plan file. |
| `new-milestone` | `new-milestone.md` | **Remove** | N/A | Same as new-project. Milestones are a GSD concept not present in the target. |
| `map-codebase` | `map-codebase.md` | **Remove** | N/A | Replaced by the planner's Step 5 (codebase exploration via Glob/Grep/Read). No separate codebase mapping phase. |
| `discuss-phase` | `discuss-phase.md` | **Remove** | N/A | The target planner uses plan mode (EnterPlanMode/ExitPlanMode) for user interaction. No separate discussion phase. |
| `research-phase` | `research-phase.md` | **Merge** | `nick:plan` | Research is integrated into the planner skill (Step 5: mandatory research via context7, microsoft-docs, shadcn MCPs). No separate research phase. |
| `list-phase-assumptions` | `list-phase-assumptions.md` | **Remove** | N/A | Plan mode in the target replaces this. User reviews the plan in plan mode before approving. |
| `plan-phase` | `plan-phase.md` | **Modify** | `nick:plan` | The orchestration logic (spawn researcher, planner, checker, revision loop) is replaced by a single skill that does research + planning in one pass. Uses MCP for docs, plan mode for user approval. |
| `execute-phase` | `execute-phase.md` | **Modify** | `nick:execute` | Wave-based parallel execution and checkpoint protocols are removed. The target executor reads a plan file and implements it linearly. No per-task commits (verifier/reviewer commits). |
| `execute-plan` | `execute-plan.md` | **Merge** | `nick:execute` | This was the per-plan executor reference. Merged into the single execute skill. |
| `verify-phase` | `verify-phase.md` | **Modify** | `nick:verify` | Static goal-backward verification replaced by Chrome DevTools runtime testing. Completely different verification approach. |
| `verify-work` | `verify-work.md` | **Modify** | `nick:verify` | Conversational UAT replaced by automated workflow tests (snapshot-interact-verify) plus Chrome DevTools inspection. |
| `diagnose-issues` | `diagnose-issues.md` | **Remove** | N/A | The target has no equivalent. Issues found during verify are reported. Fixes go back through execute. |
| `discovery-phase` | `discovery-phase.md` | **Merge** | `nick:plan` | Discovery is part of the planner's research step. |
| `help` | `help.md` | **Modify** | N/A | Needs new help content reflecting the 4+3 skill structure. Much simpler. |
| `progress` | `progress.md` | **Remove** | N/A | No STATE.md in the target. Progress is tracked by the existence of files in `.pipeline/`. |
| `quick` | `quick.md` | **Remove** | N/A | The target's plan->execute->verify->review IS the quick path already. No separate quick mode needed. |
| `settings` | `settings.md` | **Remove** | N/A | No config.json in the target. Skills are self-contained. |
| `set-profile` | `set-profile.md` | **Remove** | N/A | No model profiles in the target. Claude Code handles model selection. |
| `pause-work` | `pause-work.md` | **Remove** | N/A | No session management in the target. Pipeline files persist state naturally. |
| `resume-project` | `resume-project.md` | **Remove** | N/A | Same as pause-work. Pipeline files serve as resume points. |
| `transition` | `transition.md` | **Remove** | N/A | No inter-command routing needed. Users invoke skills directly. |
| `add-phase` | `add-phase.md` | **Remove** | N/A | No roadmap management in the target. |
| `insert-phase` | `insert-phase.md` | **Remove** | N/A | No roadmap management in the target. |
| `remove-phase` | `remove-phase.md` | **Remove** | N/A | No roadmap management in the target. |
| `add-todo` | `add-todo.md` | **Remove** | N/A | No todo management in the target. Use GitHub issues or plain notes. |
| `check-todos` | `check-todos.md` | **Remove** | N/A | No todo management in the target. |
| `complete-milestone` | `complete-milestone.md` | **Remove** | N/A | No milestone lifecycle in the target. |
| `audit-milestone` | `audit-milestone.md` | **Remove** | N/A | No milestone auditing in the target. |
| `plan-milestone-gaps` | `plan-milestone-gaps.md` | **Remove** | N/A | No gap closure workflow in the target. |
| `update` | `update.md` | **Remove** | N/A | No self-update mechanism needed for custom skills. |

**Summary:** Of 30 workflows, **4 are modified/merged** into target skills, **26 are removed**.

---

## Agent Mapping

### GSD Agents -> Target State

| GSD Agent | File | Target State | Rationale |
|---|---|---|---|
| `gsd-planner` | `gsd-planner.md` | **Modify** | Becomes `nick:plan`. Loses: wave computation, dependency graphs, gap closure mode, revision mode, XML task format, must_haves derivation. Gains: MCP integration (context7, microsoft-docs, shadcn), plan mode workflow, simpler markdown plan output, .pipeline/ file structure. |
| `gsd-executor` | `gsd-executor.md` | **Modify** | Becomes `nick:execute`. Loses: per-task commits, checkpoint protocols, continuation handling, TDD execution, STATE.md updates, gsd-tools.cjs calls, deviation rules (kept conceptually but simplified). Gains: MCP doc lookups, skills loading (react, nestjs, owasp), build summary output, no-commit rule (reviewer commits). |
| `gsd-verifier` | `gsd-verifier.md` | **Modify** | Becomes `nick:verify`. Loses: static goal-backward verification, grep-based stub detection, anti-pattern scanning, VERIFICATION.md output. Gains: Chrome DevTools MCP (full browser automation), workflow test generation (5 explorer agents), test-planner + browser-tester delegation, runtime inspection (console, network, performance). |
| `gsd-phase-researcher` | `gsd-phase-researcher.md` | **Remove** | Absorbed into `nick:plan` Step 5 (mandatory research via MCP). No separate researcher agent. |
| `gsd-project-researcher` | `gsd-project-researcher.md` | **Remove** | No project-level research phase in the target. |
| `gsd-research-synthesizer` | `gsd-research-synthesizer.md` | **Remove** | No synthesis step needed when research is inline in the planner. |
| `gsd-plan-checker` | `gsd-plan-checker.md` | **Remove** | Plan mode (user reviews plan before approving) replaces automated plan checking. |
| `gsd-codebase-mapper` | `gsd-codebase-mapper.md` | **Remove** | Planner explores codebase directly. No separate mapping agents. |
| `gsd-debugger` | `gsd-debugger.md` | **Remove** | No structured debugging workflow in the target. Standard Claude debugging is used. |
| `gsd-integration-checker` | `gsd-integration-checker.md` | **Remove** | Cross-phase integration checking is replaced by runtime verification (nick:verify). |
| `gsd-roadmapper` | `gsd-roadmapper.md` | **Remove** | No roadmap creation in the target. Projects start with a plan file. |

**Summary:** Of 11 agents, **3 are modified** into target skills, **8 are removed**.

---

## Missing Capabilities

Things present in the user's target skills that GSD completely lacks:

### 1. MCP Integration (Critical Gap)

The target skills depend heavily on MCP servers:

| MCP Server | Used By | Purpose |
|---|---|---|
| `context7` | plan, execute | Library documentation lookup (resolve-library-id, query-docs) |
| `microsoft-docs` | plan, execute | Azure docs search, code samples, page fetch |
| `shadcn` | plan, execute | UI component details (getComponents, getComponent) |
| `chrome-devtools` | verify | Full browser automation (snapshot, click, fill, navigate, console, network, performance) |
| `ide` | review | getDiagnostics for TypeScript error checking |

**GSD has zero MCP integration.** It uses gsd-tools.cjs for everything and WebSearch/WebFetch for research. This is the single largest architectural gap.

### 2. Chrome DevTools Runtime Verification

The target's `nick:verify` skill can:
- Navigate to the running app and take page snapshots
- Click, fill, hover, press keys on the live UI
- Check console for errors/warnings
- Inspect network requests and responses
- Run performance traces and analyze insights
- Generate workflow test documents from codebase analysis (5 parallel explorer agents)
- Execute automated test plans via test-planner + browser-tester agents

GSD verification is entirely static: grep for patterns, check file existence, count lines. It cannot interact with the running app.

### 3. Dedicated Code Review Skill

The target's `nick:review` skill is a standalone step that:
- Reads every created/modified file
- Loads best-practice skills (vercel-react-best-practices, nestjs-best-practices, owasp-security, web-design-guidelines)
- Runs IDE diagnostics via MCP
- Runs npm typecheck, test, build
- Produces a structured review with severity levels
- Commits ONLY if review passes

GSD has no code review step. The executor self-verifies and commits per-task.

### 4. Skills-Based Best Practices Loading

The target executor and reviewer load domain-specific skills before writing/reviewing code:
- `vercel-react-best-practices` -- mandatory for React/Next.js code
- `nestjs-best-practices` -- mandatory for NestJS code
- `frontend-design` -- mandatory for UI components
- `owasp-security` -- for security-sensitive code

GSD has no skills system. Best practices are embedded in agent prompts or absent entirely.

### 5. Plan Mode (EnterPlanMode/ExitPlanMode)

The target planner enters plan mode, does all research and writing there, then exits for user approval. This is a Claude Code native feature that GSD does not use.

### 6. Pipeline File Structure

The target uses `.pipeline/` with a simple structure:
```
.pipeline/
  plans/         -- active plans
  plans/done/    -- completed plans
  builds/        -- build summaries
  reviews/       -- review reports
```

This replaces GSD's complex `.planning/` structure with phases, numbered directories, frontmatter schemas, etc.

### 7. Workflow Test Documents

The target includes `generate-workflow` and `update-workflow` skills that create detailed test documents by analyzing the codebase with 5 parallel explorer agents. These documents drive automated browser testing. GSD has nothing equivalent.

### 8. Feature Slug as Primary Key

The target uses a simple kebab-case feature slug (e.g., `jwt-api-auth`) as the primary key for tracking a feature through plan -> build -> review. GSD uses phase numbers + plan numbers (e.g., `03-02-PLAN.md`).

---

## Simplification Opportunities

### 1. Eliminate State Management Entirely

GSD's STATE.md, progress bars, session continuity, metrics tracking, and position management can all be removed. The `.pipeline/` directory structure is self-documenting:
- Plans in `plans/` = not started
- Plans in `plans/done/` = completed
- Builds in `builds/` = implemented, needs review
- Reviews in `reviews/` = reviewed

**Token savings: ~2000 tokens per agent invocation** (no STATE.md reading/writing).

### 2. Eliminate gsd-tools.cjs

The CLI tool handles: init, state management, frontmatter validation, plan structure verification, commit helpers, roadmap parsing, phase operations, websearch, history digest. Most of these are unnecessary in the target:

| gsd-tools function | Target replacement |
|---|---|
| `init` | Skills read project files directly |
| `state *` | No state management |
| `frontmatter *` | No frontmatter schemas |
| `verify *` | MCP-based verification |
| `commit` | git commands directly |
| `roadmap *` | No roadmap |
| `phase-plan-index` | No phase/wave system |
| `websearch` | WebSearch tool |
| `history-digest` | No history tracking |

**However:** gsd-tools.cjs could be retained as a thin helper for common operations (mkdir, file moves, etc.) if desired. The target skills don't reference it at all.

### 3. Collapse Research into Planning

GSD spawns up to 4 parallel researchers + 1 synthesizer (5 subagent invocations) before planning begins. The target planner does inline research via MCP lookups in Step 5. This eliminates ~5x subagent overhead.

### 4. Eliminate Plan Checking Revision Loop

GSD runs a plan checker after the planner, then potentially 3 revision iterations (planner <-> checker). The target uses plan mode: the user reviews and approves the plan themselves. Zero revision loop overhead.

### 5. Eliminate Wave/Dependency Computation

GSD computes execution waves, dependency graphs, and parallel execution groups. The target executes plans linearly, one at a time. This eliminates all wave computation logic.

### 6. Merge 30 Workflows into 7 Skills

The mapping above shows 26 of 30 workflows are removed outright. The remaining 4 are merged/modified into simpler skills. The 3 new workflow-test skills are net-new.

---

## Migration Risk Assessment

### High Risk

| Risk | Description | Mitigation |
|---|---|---|
| **Loss of project structure** | GSD's init -> roadmap -> phases provides a complete project lifecycle. The target has no equivalent. Users lose guided project setup. | Accept this as a feature, not a bug. The target assumes developers know what they want to build and just need plan -> execute -> verify -> review. |
| **Loss of state persistence** | GSD's STATE.md survives context resets and provides continuity. The target has no session state. If context resets mid-build, there's no automatic resume. | Build summaries and plan files provide implicit state. The user can always re-read build summary and continue. |
| **MCP server dependency** | The target skills require chrome-devtools, context7, microsoft-docs, shadcn, and ide MCP servers to be running. If any are disconnected, skills degrade. | Skills should gracefully degrade when MCP servers are unavailable (skip that step, note it was skipped). |
| **gsd-tools.cjs removal** | GSD commands depend heavily on gsd-tools.cjs for init, state, commit, validation. Removing it requires rewriting all those operations inline in skills. | Keep gsd-tools.cjs as an optional utility. Skills can work without it but use it when available. |

### Medium Risk

| Risk | Description | Mitigation |
|---|---|---|
| **Verification coverage gap** | GSD's static verification catches stub code and missing wiring via grep patterns. The target's runtime verification only catches issues visible in the browser. Unreachable code, unused exports, and type errors may slip through. | The target's `nick:review` skill with IDE diagnostics partially covers this. The combination of review (static) + verify (runtime) should provide adequate coverage. |
| **No gap closure loop** | GSD has an automated gap closure cycle: verify -> find gaps -> plan fixes -> execute fixes -> re-verify. The target has no equivalent. | The target's executor has a "Handling Fix Requests" section that reads review files and fixes issues. This is a manual loop but achieves the same result. |
| **Parallel execution loss** | GSD can execute independent plans in parallel via waves. The target executes linearly. This is slower for large projects. | Accept the tradeoff. Linear execution is simpler and more predictable. Most features are sequential anyway. |

### Low Risk

| Risk | Description | Mitigation |
|---|---|---|
| **Commit strategy change** | GSD commits per-task (every 15-60 min of work). The target commits only after review passes (potentially hours of uncommitted work). | This is a deliberate design choice. The reviewer commits with full context. If context resets before review, the code changes are still on disk. |
| **Template/reference loss** | GSD has 15+ templates and 13 reference docs. The target skills are self-contained. | Templates are embedded in skill definitions (plan template in nick:plan, build summary template in nick:execute, etc.). No separate template files needed. |

---

## Recommended Migration Order

### Phase 1: Foundation (Do First)

1. **Create `nick:plan` skill** -- This is the entry point. Port the plan-mode workflow, MCP research integration, and plan template. Test it standalone.

2. **Create `nick:execute` skill** -- Depends on plan output format from Phase 1. Port the execution logic, skills loading, build summary output.

3. **Set up `.pipeline/` directory structure** -- Simple mkdir commands in both skills.

### Phase 2: Verification & Review (Do Second)

4. **Create `nick:review` skill** -- Depends on build summary format from Phase 2. Port the code review logic, skills loading, IDE diagnostics integration.

5. **Create `nick:verify` skill** -- Depends on build summary format. This is the most complex skill due to Chrome DevTools integration. Port the workflow test generation (5 explorers) and browser testing.

### Phase 3: Supporting Skills (Do Third)

6. **Create `generate-workflow` skill** -- Port the 5-explorer pattern for workflow document generation.

7. **Create `update-workflow` skill** -- Port the differential update logic.

8. **Create `test-flow` skill** -- Port the test-planner + browser-tester orchestration.

### Phase 4: Cleanup (Do Last)

9. **Write new help/reference** -- Document the 7 skills.

10. **Evaluate gsd-tools.cjs** -- Decide what to keep (if anything) as a utility.

11. **Remove deprecated GSD files** -- Clean up unused workflows, agents, templates, references.

### Why This Order

- Plan and execute are the core loop. Everything else depends on their output format.
- Review and verify validate the core loop. They need to understand build summaries.
- Supporting skills are optional enhancements. They can be added later.
- Cleanup is last because old files don't hurt anything while migration is in progress.

---

## Appendix: Complete File Inventories

### GSD Workflow Files (30)

Located in `get-shit-done/workflows/`:
```
add-phase.md, add-todo.md, audit-milestone.md, check-todos.md,
complete-milestone.md, diagnose-issues.md, discovery-phase.md,
discuss-phase.md, execute-phase.md, execute-plan.md, help.md,
insert-phase.md, list-phase-assumptions.md, map-codebase.md,
new-milestone.md, new-project.md, pause-work.md, plan-milestone-gaps.md,
plan-phase.md, progress.md, quick.md, remove-phase.md, research-phase.md,
resume-project.md, set-profile.md, settings.md, transition.md,
update.md, verify-phase.md, verify-work.md
```

### GSD Agent Files (11)

Located in `agents/`:
```
gsd-codebase-mapper.md, gsd-debugger.md, gsd-executor.md,
gsd-integration-checker.md, gsd-phase-researcher.md,
gsd-plan-checker.md, gsd-planner.md, gsd-project-researcher.md,
gsd-research-synthesizer.md, gsd-roadmapper.md, gsd-verifier.md
```

### User Target Skill Files (7)

Located in `docs/nick/`:
```
plan.md, execute.md, verify.md, review.md,
generate-workflow.md, update-workflow.md, test-flow.md
```

### GSD Template Files (20+)

Located in `get-shit-done/templates/`:
```
DEBUG.md, UAT.md, context.md, continue-here.md,
debug-subagent-prompt.md, discovery.md, milestone-archive.md,
milestone.md, phase-prompt.md, planner-subagent-prompt.md,
project.md, requirements.md, roadmap.md, state.md,
summary.md, summary-complex.md, summary-minimal.md,
summary-standard.md, user-setup.md, verification-report.md,
codebase/ (7 templates), research-project/ (5 templates)
```

### GSD Reference Files (13)

Located in `get-shit-done/references/`:
```
checkpoints.md, continuation-format.md, decimal-phase-calculation.md,
git-integration.md, git-planning-commit.md, model-profile-resolution.md,
model-profiles.md, phase-argument-parsing.md, planning-config.md,
questioning.md, tdd.md, ui-brand.md, verification-patterns.md
```
