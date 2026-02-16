# Phase 1: Core Skills & Token Architecture - Research

**Researched:** 2026-02-15
**Domain:** Claude Code SKILL.md conversion, token optimization, agent architecture
**Confidence:** HIGH

## Summary

Phase 1 converts `plan-phase` and `execute-phase` from the current command+workflow+agent architecture into proper SKILL.md folders with frontmatter, lean bodies, and agent definitions in `references/` directories. The core transformation is switching from content injection (orchestrator loads file contents into agent prompts) to path-based context passing (agents receive file paths and read them with their own fresh 200k context). This is estimated to reduce token consumption by 60-75% per workflow cycle.

The user already has 7 working skills at `docs/nick/` (plan, execute, verify, review, generate-workflow, update-workflow, test-flow) and 3 installed SKILL.md-format skills (`nestjs-best-practices`, `owasp-security`, `security-review`) that demonstrate the target pattern. The existing `security-review` skill is the most relevant precedent: it orchestrates 12+ sub-agents via the Task tool, uses file-based handoffs between phases, and keeps the orchestrator context lean by never reading agent output files.

**Primary recommendation:** Create `gsd-plan-phase/SKILL.md` and `gsd-execute-phase/SKILL.md` as orchestrator skills with lean bodies (~100-150 lines each). Move the 3 agent definitions used by plan-phase (gsd-phase-researcher, gsd-planner, gsd-plan-checker) and the 2 used by execute-phase (gsd-executor, gsd-verifier) into each skill's `references/agents/` directory. Switch all agent prompts from content injection to path-based context passing.

## Standard Stack

### Core

| Component | Location | Purpose | Why Standard |
|-----------|----------|---------|--------------|
| SKILL.md format | Anthropic spec | Skill definition with YAML frontmatter, markdown body, references/ | Official Claude Code skill format; progressive disclosure built-in |
| `allowed-tools` frontmatter | SKILL.md YAML | Restrict tool access per skill | Enforces role separation (PIPE-02, PIPE-03, PIPE-06) |
| Task tool | Claude Code built-in | Spawn sub-agents with fresh 200k context | GSD's existing agent pattern; works from within skills |
| gsd-tools.cjs | `~/.claude/get-shit-done/bin/gsd-tools.cjs` | Init, state mgmt, roadmap parsing, commits | Required -- all workflows depend on it, not being removed |
| .planning/ directory | Project root | Persistent state, phase artifacts | Proven structure, explicitly kept per PROJECT.md |

### Supporting

| Component | Location | Purpose | When to Use |
|-----------|----------|---------|-------------|
| references/ directory | Inside each skill folder | Agent definitions, templates, prompts | Always -- core to progressive disclosure |
| Absolute paths in agent prompts | `~/.claude/skills/gsd-plan-phase/references/agents/...` | Reference files within skill folder from Task prompts | When spawning agents via Task tool |
| Path-based context passing | Agent prompt instructions | Tell agents to Read files themselves | Every agent spawn (TOKEN-01) |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Absolute paths to references/ | Relative paths from skill root | Relative paths are NOT reliable across Task() boundaries; absolute paths are proven (security-review skill uses them) |
| Agent definitions in references/ | Agents as standalone .md files in ~/.claude/agents/ | Standalone agents are the current pattern but violate SKILL-05; references/ is self-contained and portable |
| YAML list for allowed-tools | String format per SKILL.md spec | Both formats work in Claude Code; YAML list is more readable; user's existing skills use both |
| Shared references/ across skills | Duplicated references per skill | Shared avoids duplication but reduces portability; GSD installs to a known path so shared references via absolute paths work |

## Architecture Patterns

### Recommended Skill Folder Structure

```
~/.claude/skills/
  gsd-plan-phase/
    SKILL.md                           # ~100-150 lines, orchestration only
    references/
      agents/
        gsd-phase-researcher.md        # ~469 lines, research agent role
        gsd-planner.md                 # ~1159 lines, planning agent role
        gsd-plan-checker.md            # ~622 lines, verification agent role
      templates/
        researcher-prompt.md           # Prompt template for researcher spawn
        planner-prompt.md              # Prompt template for planner spawn
        checker-prompt.md              # Prompt template for checker spawn

  gsd-execute-phase/
    SKILL.md                           # ~100-150 lines, orchestration only
    references/
      agents/
        gsd-executor.md                # ~419 lines, execution agent role
        gsd-verifier.md                # ~541 lines, verification agent role
      templates/
        executor-prompt.md             # Prompt template for executor spawn
```

### Pattern 1: Lean Orchestrator with Path-Based Agent Spawning

**What:** SKILL.md body contains ONLY orchestration logic (init, validate, spawn agents, handle returns, present results). All detailed instructions live in references/.

**When to use:** Every GSD skill that spawns agents.

**Example (plan-phase body excerpt):**

```markdown
## Step 5: Spawn Researcher

Task(
  prompt="First, read ~/.claude/skills/gsd-plan-phase/references/agents/gsd-phase-researcher.md for your role and instructions.

<objective>
Research how to implement Phase {phase_number}: {phase_name}
Answer: 'What do I need to know to PLAN this phase well?'
</objective>

<files_to_read>
- .planning/STATE.md
- .planning/ROADMAP.md (Phase {X} section)
- .planning/REQUIREMENTS.md (Phase {X} requirements)
- {phase_dir}/*-CONTEXT.md (if exists)
</files_to_read>

<output>
Write to: {phase_dir}/{padded_phase}-RESEARCH.md
</output>",
  subagent_type="general-purpose",
  description="Research Phase {phase}"
)
```

**Key aspects:**
- Agent reads its role instructions via Read tool from references/agents/
- Agent reads project files via Read tool from .planning/ (path-based, not injected)
- Orchestrator passes ~20-30 lines of context, not ~1,000 lines of content
- Agent gets fresh 200k context to read everything itself

### Pattern 2: Frontmatter-Enforced Role Separation

**What:** The `allowed-tools` field in SKILL.md frontmatter restricts what each skill can do. This is the mechanism for PIPE-02 (planner never implements), PIPE-03 (executor never commits), and PIPE-06 (explicit tool restrictions).

**When to use:** Every skill that has role constraints.

**Example (plan-phase frontmatter):**

```yaml
---
name: gsd-plan-phase
description: >
  Create detailed execution plans for a roadmap phase. Orchestrates research,
  planning, and verification agents. Use when user says "plan phase",
  "create plan for phase", "/gsd:plan-phase", or wants to plan the next
  phase of their project. Requires .planning/ directory with ROADMAP.md.
argument-hint: "<phase-number> [--research] [--skip-research] [--gaps] [--skip-verify]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - WebFetch
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
---
```

**Note:** `allowed-tools` restricts the orchestrator skill itself. Sub-agents spawned via Task get their own tool access defined by their agent definition, NOT inherited from the parent skill's allowed-tools.

**Example (execute-phase frontmatter):**

```yaml
---
name: gsd-execute-phase
description: >
  Execute all plans in a phase with wave-based parallel agents. Use when user
  says "execute phase", "run phase", "build phase", or "/gsd:execute-phase".
  Requires planned phase with PLAN.md files.
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
```

**Enforced restrictions:**
- Plan-phase: Read + Write (for plan docs) + Task (for agent spawning). NO Edit, NO code implementation tools. This satisfies PIPE-02.
- Execute-phase orchestrator: Read + Write + Edit + Bash + Task. The executor agents commit per-task (current pattern). The reviewer (Phase 3) will commit after PASS. For Phase 1, PIPE-03 is satisfied by ensuring the execute-phase ORCHESTRATOR never commits -- the executor agents do. Full PIPE-03 (executor never commits, reviewer does) requires Phase 3's review skill.

### Pattern 3: Agent Definitions as references/ Files

**What:** Agent .md files (currently in `~/.claude/agents/`) move to skill `references/agents/` directories. This satisfies SKILL-05.

**When to use:** Every agent used by a skill.

**Current pattern (agents as standalone files):**
```
~/.claude/agents/
  gsd-planner.md          # Standalone, loaded by plan-phase
  gsd-executor.md         # Standalone, loaded by execute-phase
  gsd-verifier.md         # Standalone, loaded by execute-phase
```

**Target pattern (agents in skill references/):**
```
~/.claude/skills/gsd-plan-phase/references/agents/
  gsd-planner.md          # Bundled with plan-phase skill
  gsd-phase-researcher.md # Bundled with plan-phase skill
  gsd-plan-checker.md     # Bundled with plan-phase skill

~/.claude/skills/gsd-execute-phase/references/agents/
  gsd-executor.md         # Bundled with execute-phase skill
  gsd-verifier.md         # Bundled with execute-phase skill
```

**Path reference change in Task prompts:**
```
# Before (standalone agent):
"First, read ~/.claude/agents/gsd-planner.md for your role..."

# After (skill-bundled agent):
"First, read ~/.claude/skills/gsd-plan-phase/references/agents/gsd-planner.md for your role..."
```

**Duplication consideration:** gsd-verifier.md is used by both execute-phase and (in Phase 3) verify-phase. Options: (a) duplicate in both skills' references/agents/, (b) keep one canonical copy and reference it by absolute path. Recommendation: duplicate for now (self-contained skills), refactor to shared references if maintenance burden becomes real.

### Anti-Patterns to Avoid

- **Injecting file contents into agent prompts:** This is the entire problem being solved. Each agent has 200k context -- let it read files itself. The orchestrator should pass paths, not content.
- **Putting agent logic in the SKILL.md body:** The body is orchestration. Agent instructions belong in references/agents/. Body should be under 1,500 tokens (TOKEN-02).
- **Using relative paths in Task prompts:** Relative paths do not resolve reliably across Task() boundaries. The security-review skill confirmed this: it uses absolute paths (`~/.claude/skills/security-review/phases/...`) in all agent prompts. GSD skills should do the same.
- **Making SKILL.md body a copy of the workflow file:** The workflow files are 377-440 lines. SKILL.md bodies should be 100-150 lines of orchestration steps only.
- **Omitting allowed-tools from frontmatter:** Without explicit allowed-tools, the skill has access to all tools, defeating role separation (PIPE-06).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Phase initialization + context loading | Custom init logic in SKILL.md | `gsd-tools.cjs init plan-phase "$PHASE"` | Already handles 15+ fields, model resolution, file discovery; tested and proven |
| Roadmap/state parsing | Inline Bash parsing in skill body | `gsd-tools.cjs roadmap get-phase`, `state-snapshot` | Complex format parsing with edge cases already handled |
| Agent role definitions | Inline role text in Task prompts | Separate .md files in references/agents/ | Agent roles are 400-1100+ lines; embedding them in prompts defeats progressive disclosure |
| PLAN.md structure validation | Manual grep checks | `gsd-tools.cjs verify plan-structure` | Existing validation with frontmatter schema checks |
| Git commits for planning docs | Raw git commands | `gsd-tools.cjs commit` | Handles secret scanning, file staging, message formatting |

**Key insight:** gsd-tools.cjs is NOT being removed. It handles atomic operations that all skills depend on. The change is in how the orchestration layer calls it (from a skill body instead of a workflow file) and how context gets passed to agents (paths instead of content).

## Common Pitfalls

### Pitfall 1: Body Size Creep

**What goes wrong:** The SKILL.md body grows beyond 1,500 tokens because developers move workflow logic into the body verbatim instead of refactoring.
**Why it happens:** Current workflow files are 377-440 lines. Copy-pasting them into SKILL.md and trimming only slightly results in a 3,000+ token body.
**How to avoid:** The body should be a step list with bash commands and Task spawning only. Detailed agent prompts, templates, display banners, error handling guides go into references/.
**Warning signs:** Body exceeds 150 lines; body contains agent role descriptions; body has multi-paragraph contextual instructions.

### Pitfall 2: Path Resolution in Task Prompts

**What goes wrong:** Agent is told to "read references/agents/gsd-planner.md" (relative path) but the Task agent's working directory is the user's project, not the skill folder.
**Why it happens:** Skills and their sub-agents run in different contexts. The skill's body runs within the skill's context, but Task agents run in the user's project context.
**How to avoid:** Always use absolute paths in Task prompts: `~/.claude/skills/gsd-plan-phase/references/agents/gsd-planner.md`. This is confirmed by the security-review skill's pattern.
**Warning signs:** "File not found" errors when agents try to read their role instructions; agents proceeding without reading their role file.

### Pitfall 3: Orchestrator Context Bloat from Agent Returns

**What goes wrong:** The orchestrator reads agent output (RESEARCH.md, PLAN.md, VERIFICATION.md) to "check" results, consuming 5,000-15,000 tokens per read. After 3-4 agents, orchestrator context is bloated.
**Why it happens:** Desire to validate agent work before proceeding.
**How to avoid:** Follow the security-review skill pattern: agents return a one-line structured signal (e.g., "PLANNING COMPLETE: 3 plans created"). Orchestrator spot-checks by verifying files exist on disk (`ls`, `grep` for status markers), never by reading full content. The 10-15% orchestrator context rule is already documented in execute-phase.md.
**Warning signs:** Orchestrator reading PLAN.md or RESEARCH.md into variables; growing orchestrator context across agent spawns.

### Pitfall 4: allowed-tools Confusion Between Skill and Agents

**What goes wrong:** Developers assume the skill's `allowed-tools` restricts sub-agents spawned via Task.
**Why it happens:** The frontmatter `allowed-tools` field suggests global scope.
**How to avoid:** Understand that `allowed-tools` restricts the skill itself (the orchestrator). Sub-agents spawned via Task get tool access based on their own agent type and the Task tool parameters. If you need an agent restricted, define those restrictions in the agent's role file (references/agents/gsd-executor.md) as instructions, not as a frontmatter enforcement.
**Warning signs:** Agent performing actions the skill's allowed-tools should prevent; confusion about why executor can commit when execute-phase skill doesn't list git tools.

### Pitfall 5: Duplicate Agent Files Without Synchronization

**What goes wrong:** gsd-verifier.md exists in both execute-phase/references/agents/ and (eventually) verify-phase/references/agents/. An update to one copy is not propagated to the other.
**Why it happens:** Self-contained skills require bundled agent definitions, leading to duplication.
**How to avoid:** For Phase 1, accept duplication and document which agents are shared. Consider a shared references location at `~/.claude/get-shit-done/references/agents/` that both skills reference by absolute path. Alternatively, the installation script could copy from a canonical source.
**Warning signs:** Different behavior from the same agent type in different skills; bug fixes applied to one copy but not the other.

### Pitfall 6: Breaking the Installation Path

**What goes wrong:** Skills are placed directly in `~/.claude/skills/` but the current installation (`npx get-shit-done-cc`) copies to `~/.claude/get-shit-done/`, `~/.claude/commands/gsd/`, and `~/.claude/agents/`. New skill folders need to be installed alongside or in place of these.
**Why it happens:** Skills use a different installation mechanism than commands/agents.
**How to avoid:** During Phase 1, install skills to `~/.claude/skills/gsd-plan-phase/` etc. Keep existing commands/agents working in parallel until migration is complete. Update `bin/install.js` to also copy skill folders. Consider: do the new skills REPLACE the old commands or COEXIST? Recommendation: coexist during development, replace once validated.
**Warning signs:** `/gsd:plan-phase` still triggering old command instead of new skill; users needing to manually install skill folders.

## Code Examples

### Example 1: Complete gsd-plan-phase/SKILL.md Body Structure

```markdown
---
name: gsd-plan-phase
description: >
  Create detailed execution plans for a roadmap phase. Orchestrates research,
  planning, and verification agents. Use when user says "plan phase",
  "create plan for phase", "/gsd:plan-phase", or wants to plan the next
  phase of their project. Requires .planning/ directory with ROADMAP.md.
argument-hint: "<phase-number> [--research] [--skip-research] [--gaps] [--skip-verify]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - WebFetch
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
---

# Plan Phase

You orchestrate phase planning. You spawn agents -- you do NOT do the planning work yourself.

## Step 1: Initialize

[bash: gsd-tools.cjs init plan-phase "$PHASE" -- no --include flag, no content loading]
[Parse: phase_dir, models, flags, has_research, has_plans, etc.]
[Validate: planning_exists, phase_found]

## Step 2: Parse Arguments

[Extract phase number, --research, --skip-research, --gaps, --skip-verify]
[Auto-detect next phase if not specified]

## Step 3: Handle Research (conditional)

[Skip if: --gaps, --skip-research, research_enabled=false, or RESEARCH.md exists]
[If needed: spawn researcher from references/agents/gsd-phase-researcher.md]
[Pass: phase description, CONTEXT.md path, output path -- NOT file contents]

## Step 4: Spawn Planner

[Spawn gsd-planner from references/agents/gsd-planner.md]
[Pass: phase number, goal, mode, file PATHS (STATE, ROADMAP, REQUIREMENTS, CONTEXT, RESEARCH)]
[Agent reads files itself with fresh 200k context]

## Step 5: Verify Plans (conditional)

[Skip if: --skip-verify or plan_checker_enabled=false]
[Spawn gsd-plan-checker from references/agents/gsd-plan-checker.md]
[Pass: phase_dir path, requirements path -- NOT plan contents]

## Step 6: Revision Loop (max 3)

[If checker returns ISSUES FOUND: re-spawn planner with issue list + plan paths]
[Increment counter, re-check, max 3 iterations]

## Step 7: Present Results

[Display: plan count, wave breakdown, research/verification status]
[Offer: /gsd:execute-phase {X}]
```

### Example 2: Path-Based Agent Prompt (vs Current Content-Based)

**Current (content injection -- ~18,000 tokens per planner spawn):**
```markdown
<planning_context>
**Phase:** 3
**Mode:** standard
**Project State:** [150 lines of STATE.md content]
**Roadmap:** [300 lines of ROADMAP.md content]
**Requirements:** [400 lines of REQUIREMENTS.md content]
**Research:** [200 lines of RESEARCH.md content]
**Context:** [100 lines of CONTEXT.md content]
</planning_context>
```

**Target (path-based -- ~120 tokens per planner spawn):**
```markdown
<objective>
Plan Phase 3: API Authentication
Goal: Implement JWT auth with refresh rotation
Mode: standard
Output: .planning/phases/03-api-authentication/
</objective>

<files_to_read>
Read these files at execution start:
- Role: ~/.claude/skills/gsd-plan-phase/references/agents/gsd-planner.md
- State: .planning/STATE.md
- Roadmap: .planning/ROADMAP.md (read Phase 3 section)
- Requirements: .planning/REQUIREMENTS.md (read Phase 3 requirements)
- Research: .planning/phases/03-api-authentication/03-RESEARCH.md
- Context: .planning/phases/03-api-authentication/03-CONTEXT.md (if exists)
</files_to_read>
```

### Example 3: Execute-Phase Executor Spawn (Already Path-Based)

The execute-phase orchestrator ALREADY uses path-based passing (from current execute-phase.md line 99). This pattern is preserved:

```markdown
Task(
  prompt="
    <objective>
    Execute plan {plan_number} of phase {phase_number}-{phase_name}.
    Commit each task atomically. Create SUMMARY.md. Update STATE.md.
    </objective>

    <role>
    Read ~/.claude/skills/gsd-execute-phase/references/agents/gsd-executor.md for your role.
    </role>

    <files_to_read>
    - Plan: {phase_dir}/{plan_file}
    - State: .planning/STATE.md
    - Config: .planning/config.json (if exists)
    </files_to_read>
  ",
  subagent_type="general-purpose",
  description="Execute Plan {plan_id}"
)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Commands in commands/gsd/*.md | Skills in ~/.claude/skills/*/SKILL.md | 2025 (Claude Code skills) | Progressive disclosure, frontmatter-based routing, allowed-tools enforcement |
| Agent .md files in ~/.claude/agents/ | Agent definitions in skill references/agents/ | Phase 1 (this work) | Self-contained skills, portable agent definitions |
| Content injection in agent prompts | Path-based context passing | Phase 1 (this work) | 60-75% token reduction |
| XML tags in workflow files | Markdown in SKILL.md body | Phase 1 (this work) | Compliance with SKILL.md spec (no XML in frontmatter) |
| Workflow .md files in workflows/ | SKILL.md body + references/ | Phase 1 (this work) | Progressive disclosure (body loads on relevance, references on demand) |

**Deprecated/outdated:**
- `@` file reference syntax: Does NOT work across Task() boundaries. Agents must use Read tool to load files.
- `--include` flag on gsd-tools.cjs init: Returns file contents in JSON. Replace with path-only init (agents read files themselves).
- `subagent_type="gsd-planner"` etc.: Custom subagent types may not affect behavior in Claude Code. Standardize on `"general-purpose"` and let the agent definition file (read via prompt) define the role.

## Open Questions

1. **Skill installation path**
   - What we know: Skills go in `~/.claude/skills/`. Current GSD installs to `~/.claude/get-shit-done/`, `~/.claude/commands/gsd/`, `~/.claude/agents/`.
   - What's unclear: Does `bin/install.js` need updating to copy skill folders? Can skills and old commands coexist? Does the new skill override the old command's trigger?
   - Recommendation: Install skills to `~/.claude/skills/gsd-plan-phase/` etc. Keep old commands during Phase 1 development. Test coexistence. Update installer once validated.

2. **Shared agent references vs duplication**
   - What we know: gsd-verifier.md is used by execute-phase AND will be used by verify-phase (Phase 3). gsd-planner.md is used by plan-phase AND quick.
   - What's unclear: Best strategy for managing shared agents across multiple skills.
   - Recommendation: For Phase 1, duplicate. When Phase 3 adds verify-phase, evaluate whether a shared `~/.claude/get-shit-done/references/agents/` location (referenced by absolute path) is cleaner.

3. **PIPE-03 enforcement in Phase 1**
   - What we know: PIPE-03 says "executor never commits." But current GSD executor agents commit per-task. The full PIPE-03 requires a reviewer (Phase 3) to commit after PASS.
   - What's unclear: Should Phase 1 change the executor to stop committing? Or is this deferred to Phase 3?
   - Recommendation: Phase 1 should NOT change the executor's commit behavior. The success criteria says "executor skill restricts to no-commit (allowed-tools enforces separation)" but this requires the review skill from Phase 3 to actually commit. Defer full PIPE-03 to Phase 3. In Phase 1, document the intent and prepare the allowed-tools structure.

4. **Token count for SKILL.md body**
   - What we know: TOKEN-02 says "SKILL.md bodies are lean orchestration only (~500-1,500 tokens)." The current workflow files are 377-440 lines (~5,000-6,500 tokens).
   - What's unclear: Can the plan-phase orchestration fit in 1,500 tokens? It has 14 steps including research handling, planning, verification, revision loop, auto-advance.
   - Recommendation: Target 1,000-1,500 tokens. Move prompt templates, display banners, error handling guides, and detailed agent instructions to references/. The body should be a numbered step list with bash commands and Task tool calls only.

5. **Coexistence of nick:* and gsd:* skills**
   - What we know: The user has nick:plan, nick:execute, nick:verify, nick:review. Phase 1 creates gsd-plan-phase, gsd-execute-phase. Both are skills with different architectures.
   - What's unclear: Will both sets of skills trigger simultaneously? Will the user maintain both?
   - Recommendation: The gsd:* skills are for GSD project workflows (multi-phase, roadmap-driven). The nick:* skills are for ad-hoc feature work (single plan, .pipeline/). They serve different purposes. Both can coexist because their trigger descriptions are different. Document this distinction.

## Sources

### Primary (HIGH confidence)
- `/Users/nick/Documents/git/get-shit-done/docs/claude-code-skill-guide.md` -- Official Anthropic skills guide; defines SKILL.md format, frontmatter fields, progressive disclosure, allowed-tools
- `/Users/nick/.claude/skills/security-review/SKILL.md` -- Working production skill with 12+ agents, file-based handoffs, absolute path references; confirms pattern viability
- `/Users/nick/.claude/skills/nestjs-best-practices/SKILL.md` -- Working production skill with references/ directory pattern
- `/Users/nick/Documents/git/get-shit-done/docs/nick/plan.md` and `execute.md` -- User's existing skills demonstrating allowed-tools, MCP integration, path-based context
- `/Users/nick/Documents/git/get-shit-done/get-shit-done/workflows/plan-phase.md` -- Current plan-phase workflow (source of truth for conversion)
- `/Users/nick/Documents/git/get-shit-done/get-shit-done/workflows/execute-phase.md` -- Current execute-phase workflow (already uses path-based passing for executors)
- `/Users/nick/Documents/git/get-shit-done/commands/gsd/plan-phase.md` -- Current plan-phase command entry point
- `/Users/nick/Documents/git/get-shit-done/commands/gsd/execute-phase.md` -- Current execute-phase command entry point

### Secondary (MEDIUM confidence)
- `/Users/nick/Documents/git/get-shit-done/.planning/research/SKILLS-ARCHITECTURE.md` -- Prior research on skills conversion strategy
- `/Users/nick/Documents/git/get-shit-done/.planning/research/TOKEN-OPTIMIZATION.md` -- Prior research on token savings with quantified estimates
- `/Users/nick/Documents/git/get-shit-done/.planning/research/GAP-ANALYSIS.md` -- Analysis of what to keep/remove/modify
- `/Users/nick/Documents/git/get-shit-done/.planning/codebase/ARCHITECTURE.md` -- Current system architecture documentation
- `/Users/nick/Documents/git/get-shit-done/.planning/codebase/STRUCTURE.md` -- Current file structure documentation

### Tertiary (LOW confidence)
- Token count estimates (60-75% reduction) -- based on prior research calculations, not measured. Actual savings will depend on implementation choices.
- `subagent_type` behavior in Claude Code -- the claim that custom types may not affect behavior is unverified. Needs testing during implementation.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - verified against existing working skills and official docs
- Architecture: HIGH - multiple production skills (security-review, nestjs-best-practices) demonstrate the pattern
- Pitfalls: HIGH - path resolution, context bloat, and allowed-tools scope confirmed by examining real skill implementations
- Token estimates: MEDIUM - based on line/token counts from existing files, not measured in production

**Research date:** 2026-02-15
**Valid until:** 2026-03-15 (stable domain, skills spec unlikely to change rapidly)
