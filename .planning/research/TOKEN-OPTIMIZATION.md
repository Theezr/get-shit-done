# Token Optimization Research for GSD Multi-Agent Workflows

**Date:** 2026-02-15
**Scope:** Strategies to drastically reduce token consumption in GSD's multi-agent orchestrator/executor architecture while maintaining output quality.

---

## Key Findings

- **Current workflow files are 300-440 lines each.** plan-phase.md is 377 lines, execute-phase.md is 339 lines, execute-plan.md is 438 lines. These are loaded into orchestrator context before any agent is spawned.
- **Content-passing is the single largest token sink.** The plan-phase orchestrator extracts `state_content`, `roadmap_content`, `requirements_content`, `research_content`, `verification_content`, `uat_content`, and `context_content` from the init call, then embeds them verbatim into agent prompts. A typical ROADMAP.md is 200-400 lines, REQUIREMENTS.md is 300-600 lines, STATE.md is 80-150 lines. This means each planner agent receives 800-1500 lines of injected content.
- **The same content is loaded multiple times across agents.** The researcher, planner, checker, and revision agents all receive `requirements_content` and `context_content`. In a single plan-phase run with 1 revision, that is 4x duplication of the same content.
- **The lean skills (nick:plan, nick:execute) are 75-85% smaller** than their workflow equivalents and use path-based loading ("if CLAUDE.md exists, read it") instead of content injection, demonstrating the pattern works.
- **Progressive disclosure (from the skills guide) has three levels:** frontmatter (always loaded, ~50 tokens), SKILL.md body (loaded when relevant, ~500-2000 tokens), references/ (loaded on-demand). This maps directly to an optimization strategy for GSD workflows.
- **Estimated total savings: 60-80% token reduction** across a full plan+execute cycle by applying the strategies below.

### Token Budget Estimates (Current vs Optimized)

| Component | Current (tokens) | Optimized (tokens) | Savings |
|-----------|-----------------|-------------------|---------|
| Orchestrator workflow load | ~8,000 | ~2,500 | 69% |
| Per-agent content injection | ~12,000-20,000 | ~2,000-4,000 | 75-80% |
| Agent role prompt (per agent) | ~3,000 | ~1,500 | 50% |
| Redundant cross-agent loading | ~40,000+ (4 agents x same content) | ~12,000 (paths, agent reads once) | 70% |
| Full plan-phase cycle (with revision) | ~100,000-150,000 | ~25,000-40,000 | 70-75% |

---

## Progressive Disclosure Patterns

### The Three-Level System Applied to GSD

**Level 1: Frontmatter / Routing (~50-100 tokens)**
Always loaded. Tells the orchestrator *what this workflow can do* and *when to use it*. This is the skill description equivalent.

```yaml
---
name: gsd:plan-phase
description: Create executable PLAN.md files for a roadmap phase. Orchestrates researcher, planner, and checker agents with revision loop.
argument-hint: "<phase-number> [--research] [--skip-verify] [--gaps]"
---
```

**Level 2: Core Process (~500-1500 tokens)**
Loaded when the workflow is invoked. Contains the orchestration logic but NOT the content of project files. Uses path references instead of content injection.

```markdown
## Process
1. Init: `gsd-tools init plan-phase "$PHASE"`
2. Parse: extract phase_number, flags
3. Research: spawn researcher with paths (not content)
4. Plan: spawn planner with paths (not content)
5. Verify: spawn checker with paths (not content)
6. Revision loop (max 3)
```

**Level 3: References (~200-2000 tokens each, loaded only when needed)**
Detailed instructions that specific agents need. NOT loaded by the orchestrator.

```
references/
  researcher-prompt.md    -- loaded only by researcher agent
  planner-prompt.md       -- loaded only by planner agent
  checker-prompt.md       -- loaded only by checker agent
  deviation-rules.md      -- loaded only by executor agents
  checkpoint-protocol.md  -- loaded only when checkpoints exist
  tdd-execution.md        -- loaded only for tdd plans
```

### Quantified Impact of Progressive Disclosure

**Current pattern (everything in one file):**
- execute-plan.md: 438 lines = ~6,500 tokens
- Every executor agent loads ALL 438 lines even if the plan has no checkpoints, no TDD, no user-setup
- Deviation rules (~40 lines), TDD execution (~30 lines), checkpoint protocol (~30 lines), auth gates (~20 lines) are loaded for every agent regardless of relevance

**Optimized pattern (split into references/):**
- Core execute-plan.md: ~150 lines = ~2,200 tokens (always loaded)
- deviation-rules.md: ~40 lines = ~600 tokens (always loaded -- all plans need this)
- checkpoint-protocol.md: ~30 lines = ~450 tokens (loaded only for non-autonomous plans)
- tdd-execution.md: ~30 lines = ~450 tokens (loaded only for TDD plans)
- auth-gates.md: ~20 lines = ~300 tokens (loaded only when auth errors occur)
- user-setup-guide.md: ~25 lines = ~375 tokens (loaded only when user_setup in frontmatter)

**Result:** A standard autonomous plan execution loads 2,800 tokens instead of 6,500 tokens -- **57% reduction per executor agent**. Since execute-phase may spawn 3-8 executor agents per phase, this saves 11,000-30,000 tokens per phase execution.

---

## Context Passing Strategies

### Path-Based vs Content-Based: The Real Impact

**Current approach (content-based):**
```markdown
<planning_context>
**Project State:** {state_content}        <!-- 80-150 lines injected -->
**Roadmap:** {roadmap_content}            <!-- 200-400 lines injected -->
**Requirements:** {requirements_content}  <!-- 300-600 lines injected -->
**Research:** {research_content}           <!-- 100-300 lines injected -->
**Phase Context:** {context_content}       <!-- 50-200 lines injected -->
</planning_context>
```

Total injected: **730-1650 lines (~10,000-25,000 tokens)** per agent prompt.

**Optimized approach (path-based):**
```markdown
<files_to_read>
Read these files at execution start:
- State: .planning/STATE.md
- Roadmap: .planning/ROADMAP.md (read only Phase {X} section)
- Requirements: .planning/REQUIREMENTS.md (read only Phase {X} requirements)
- Research: {phase_dir}/{phase}-RESEARCH.md
- Context: {phase_dir}/{phase}-CONTEXT.md
</files_to_read>
```

Total in prompt: **~8 lines (~120 tokens)**. Agent reads files itself with its fresh 200k context.

**Why path-based works:**
1. Each subagent gets a fresh 200k context window -- it has plenty of room to read files
2. The `@` syntax does not work across Task() boundaries (already noted in plan-phase.md line 135), so content must be either injected or the agent must read it
3. File reading costs the same tokens whether the orchestrator reads it or the agent reads it -- but with path-based, the orchestrator does NOT pay the token cost
4. The execute-phase.md already uses path-based for executors (line 99: "Pass paths only -- executors read files themselves")

**The key insight: the orchestrator's context is the bottleneck, not the agent's.** The orchestrator persists across all agent spawns. Every token in its context accumulates. Agent contexts are ephemeral -- they spawn, do work, and terminate.

### Selective Reading: Targeted Sections Instead of Whole Files

Instead of reading entire ROADMAP.md (200-400 lines), agents should read only their relevant section:

```bash
# Current: reads entire file
ROADMAP_CONTENT=$(echo "$INIT" | jq -r '.roadmap_content // empty')

# Optimized: reads only the relevant phase
PHASE_SECTION=$(node gsd-tools.js roadmap get-phase "${PHASE}" --format section)
```

The `gsd-tools.js roadmap get-phase` command already exists and returns just the phase section. This pattern should be extended to requirements (extract only phase-relevant requirements) and state (extract only current position + recent decisions).

### What to Include vs Exclude in Agent Prompts

**Always include (in prompt text):**
- Phase number and name (5 tokens)
- Phase goal from roadmap (10-20 tokens)
- Mode flags (standard/revision/gap_closure) (5 tokens)
- Output location (file path) (10 tokens)
- Success criteria (20-40 tokens)

**Include as path references (agent reads itself):**
- STATE.md
- ROADMAP.md (agent extracts relevant section)
- REQUIREMENTS.md (agent extracts relevant section)
- RESEARCH.md
- CONTEXT.md
- Existing PLAN.md files (for revision/checking)

**Exclude entirely from most agents:**
- Full roadmap (only need current phase section)
- Full requirements (only need current phase requirements)
- UAT content (only checker needs this)
- Verification content (only gap-closure planner needs this)
- Other phases' research/plans

---

## Agent Prompt Optimization

### Lean Prompt Structure

The nick:plan and nick:execute skills demonstrate an effective lean prompt pattern. Compare:

**Current GSD planner prompt (~250 lines of injected content):**
```markdown
<planning_context>
**Phase:** {phase_number}
**Mode:** {standard | gap_closure}
**Project State:** {state_content}        <!-- 150 lines -->
**Roadmap:** {roadmap_content}            <!-- 300 lines -->
**Requirements:** {requirements_content}  <!-- 400 lines -->
**Phase Context:** {context_content}       <!-- 100 lines -->
**Research:** {research_content}           <!-- 200 lines -->
</planning_context>

<downstream_consumer>
Output consumed by /gsd:execute-phase. Plans need:
- Frontmatter (wave, depends_on, files_modified, autonomous)
- Tasks in XML format
- Verification criteria
- must_haves for goal-backward verification
</downstream_consumer>

<quality_gate>
- [ ] PLAN.md files created in phase directory
- [ ] Each plan has valid frontmatter
- [ ] Tasks are specific and actionable
- [ ] Dependencies correctly identified
- [ ] Waves assigned for parallel execution
- [ ] must_haves derived from phase goal
</quality_gate>
```

**Optimized planner prompt (~30 lines + path references):**
```markdown
<objective>
Plan Phase {phase_number}: {phase_name}
Goal: {goal}
Mode: {standard | gap_closure}
Output: {phase_dir}/
</objective>

<files_to_read>
- .planning/STATE.md (current position + recent decisions only)
- .planning/ROADMAP.md (Phase {X} section only)
- .planning/REQUIREMENTS.md (Phase {X} requirements only)
- {phase_dir}/*-CONTEXT.md (if exists -- contains locked decisions)
- {phase_dir}/*-RESEARCH.md (if exists)
</files_to_read>

<role_instructions>
Read /path/to/agents/gsd-planner.md for full planning instructions.
</role_instructions>
```

The agent's role file (gsd-planner.md) already contains the quality gates, output format, and downstream consumer information. There is no need to duplicate it in the prompt.

### Estimated Savings Per Agent Spawn

| Agent | Current prompt tokens | Optimized prompt tokens | Savings |
|-------|----------------------|------------------------|---------|
| Researcher | ~8,000 | ~800 | 90% |
| Planner | ~18,000 | ~1,200 | 93% |
| Checker | ~15,000 | ~1,000 | 93% |
| Revision planner | ~16,000 | ~1,500 | 91% |
| Executor (per plan) | ~2,000 | ~800 | 60% |

Note: executor prompts are already relatively lean (execute-phase.md uses path-based passing). The biggest wins are in the planning pipeline.

---

## Workflow File Sizing

### Optimal Sizes

Based on the progressive disclosure model and the contrast between GSD workflows and lean skills:

| File Type | Optimal Size | Rationale |
|-----------|-------------|-----------|
| Orchestrator workflow | 100-200 lines | Coordination logic only. No agent-specific content. |
| Agent role file | 80-150 lines | Core instructions + quality gates. Deviation rules, TDD, etc. in references/. |
| Agent prompt (injected) | 20-40 lines | Objective, paths, mode flags. No file content. |
| Reference file | 30-80 lines | Single concern. Loaded only when relevant. |
| Skill frontmatter | 5-15 lines | Name, description, argument-hint, allowed-tools. |
| Skill body | 50-150 lines | Core process. Defer detail to references/. |

### When to Split into references/

**Split when:**
1. Content is only needed by a subset of executions (checkpoints, TDD, auth gates)
2. Content exceeds 40 lines and is self-contained
3. Multiple agents share the same reference (deviation rules used by all executors)
4. Content changes independently from the core workflow

**Keep inline when:**
1. Content is always needed (every execution path uses it)
2. Content is under 20 lines
3. Content requires tight coupling with surrounding logic

### Current Files That Should Be Split

**execute-plan.md (438 lines) should become:**
- execute-plan.md (~150 lines) -- core process
- references/deviation-rules.md (~45 lines) -- always loaded by executors
- references/checkpoint-protocol.md (~35 lines) -- loaded when non-autonomous
- references/tdd-execution.md (~35 lines) -- loaded when tdd plan
- references/auth-gates.md (~25 lines) -- loaded on auth errors
- references/task-commit.md (~40 lines) -- always loaded by executors

**plan-phase.md (377 lines) should become:**
- plan-phase.md (~120 lines) -- orchestration flow
- references/researcher-prompt-template.md (~30 lines)
- references/planner-prompt-template.md (~30 lines)
- references/checker-prompt-template.md (~30 lines)
- references/revision-prompt-template.md (~20 lines)
- references/plan-phase-banners.md (~30 lines) -- display formatting

**execute-phase.md (339 lines) should become:**
- execute-phase.md (~120 lines) -- orchestration flow
- references/wave-execution.md (~40 lines)
- references/checkpoint-handling.md (~40 lines)
- references/failure-handling.md (~30 lines)
- references/verification-step.md (~30 lines)

---

## Caching and State Patterns

### Problem: Redundant File Reads Across Agent Spawns

In a typical plan-phase run:
1. Orchestrator reads ROADMAP.md, REQUIREMENTS.md, STATE.md via init
2. Researcher agent reads ROADMAP.md, REQUIREMENTS.md (again)
3. Planner agent reads ROADMAP.md, REQUIREMENTS.md, STATE.md (again)
4. Checker agent reads REQUIREMENTS.md (again)
5. If revision: planner reads everything again

That is 4-5 reads of REQUIREMENTS.md in a single workflow invocation.

### Strategy 1: Pre-Extract and Pass Summaries

Instead of each agent reading full files and extracting what it needs, the orchestrator extracts minimal context once and passes it:

```bash
# Orchestrator extracts once (10 tokens in prompt):
PHASE_GOAL=$(node gsd-tools.js roadmap get-phase "$PHASE" | jq -r '.goal')
PHASE_REQS=$(node gsd-tools.js requirements get-phase "$PHASE" --summary)
CURRENT_POSITION=$(node gsd-tools.js state get-position)
RECENT_DECISIONS=$(node gsd-tools.js state get-decisions --last 5)
```

Then passes these short summaries to agents instead of full file paths. This trades orchestrator context (small) for agent read operations (saved).

### Strategy 2: gsd-tools.js Context Extraction Commands

Extend gsd-tools.js with targeted extraction commands:

```bash
# Instead of: agent reads 400-line REQUIREMENTS.md
# Use: gsd-tools extracts phase-specific requirements (~30-50 lines)
gsd-tools.js requirements get-phase 3 --format markdown

# Instead of: agent reads 300-line ROADMAP.md
# Use: gsd-tools extracts phase section (~20-30 lines)
gsd-tools.js roadmap get-phase 3 --format section

# Instead of: agent reads 150-line STATE.md
# Use: gsd-tools extracts position + recent decisions (~10-20 lines)
gsd-tools.js state snapshot --compact
```

This is not caching in the traditional sense -- it is **extraction at the source** so agents never load unnecessary content.

### Strategy 3: Orchestrator State Accumulation Guard

The orchestrator's context grows with each agent spawn/return cycle. Strategies to keep it lean:

1. **Do not store agent return content in variables.** Parse the return signal (PLANNING COMPLETE, ISSUES FOUND) and discard the content.
2. **Use structured return signals.** Agents return a status code + file path, not content.
3. **The 10-15% rule** (already documented in execute-phase.md): orchestrator should use no more than 10-15% of its context window for coordination. This means ~20k tokens max for a 200k window.

### Strategy 4: State File Size Caps

- STATE.md: cap at 150 lines (already documented in execute-plan.md line 374)
- ROADMAP.md: one phase section should be 20-40 lines
- REQUIREMENTS.md: consider splitting into per-phase requirement files if the master file exceeds 400 lines

---

## Smart Addons

These are techniques that add a small token cost but provide disproportionate quality or speed gains.

### 1. Compact Context Headers (~20 tokens, saves 500+ tokens of confusion)

Add a structured header to every agent prompt that establishes context without narrative:

```markdown
Phase: 3 | Plan: 02 | Mode: standard | Wave: 1
Goal: "User authentication with JWT refresh rotation"
Prior: Phase 2 complete (data models, API scaffold)
```

This replaces paragraphs of context explanation with a scannable summary.

### 2. Must-Have Checklist (~30 tokens, prevents expensive revision loops)

Include an explicit must-have list derived from the phase goal. This prevents the checker from flagging minor issues and forcing revision loops that cost 30,000+ tokens each:

```markdown
must_haves:
- JWT access + refresh token endpoints exist
- Token rotation on refresh
- Protected route middleware
- Login/logout flows
```

A single avoided revision loop saves more tokens than 100 must-have lists.

### 3. Negative Instructions (~15 tokens, prevents scope creep)

Explicitly tell agents what NOT to do. Scope creep is a major source of wasted tokens:

```markdown
Do NOT: add UI, write migration scripts, implement email verification, or set up monitoring.
```

### 4. Format Constraints (~10 tokens, reduces verbose output)

Constrain agent output format to avoid verbose prose:

```markdown
Output: structured PLAN.md only. No explanatory text. No alternatives discussion.
```

### 5. Early Termination Signals (~5 tokens, saves entire agent spawns)

Add conditions that skip unnecessary agents:

```markdown
Skip checker if: only 1 plan AND plan has <= 3 tasks AND no architectural changes.
```

This saves an entire checker agent spawn (~15,000 tokens in prompts + response) for simple phases.

---

## Recommendations

Prioritized by estimated impact (token savings per workflow cycle):

### Tier 1: Highest Impact (implement first)

1. **Switch planner/researcher/checker to path-based context passing** (estimated savings: 40,000-60,000 tokens per plan-phase cycle)
   - Currently these agents get full file contents injected into their prompts
   - Switch to the pattern already used by execute-phase for executors
   - Each agent reads files itself from its fresh 200k context
   - Orchestrator prompt shrinks from ~1200 lines of content to ~40 lines of paths

2. **Split execute-plan.md into core + references/** (estimated savings: 15,000-30,000 tokens per execute-phase cycle)
   - Move checkpoints, TDD, auth gates, user-setup to references/
   - Executor agents only load what is relevant to their specific plan
   - With 3-8 executors per phase, savings multiply

3. **Extract phase-specific sections instead of full files** (estimated savings: 10,000-20,000 tokens per agent spawn)
   - Extend gsd-tools.js with `requirements get-phase`, `roadmap get-phase --format section`, `state snapshot --compact`
   - Agents receive 30-50 lines instead of 200-600 lines of requirements/roadmap

### Tier 2: Medium Impact (implement second)

4. **Restructure plan-phase.md and execute-phase.md as lean orchestrators** (estimated savings: 5,000-10,000 tokens per cycle)
   - Move prompt templates to references/
   - Move display banners to references/
   - Keep only coordination logic in the main file

5. **Add early termination conditions** (estimated savings: 15,000-30,000 tokens when triggered)
   - Skip checker for simple phases (1 plan, few tasks)
   - Skip researcher when research already exists and no --research flag (already partially implemented)
   - Skip revision loop when checker returns PASSED on first try (already implemented)

6. **Compact context headers for all agent prompts** (estimated savings: 2,000-5,000 tokens per cycle)
   - Replace verbose `<planning_context>` blocks with structured one-liners
   - Add must-have checklists to prevent unnecessary revision loops

### Tier 3: Structural (implement when refactoring)

7. **Convert GSD workflows to skill format** (progressive disclosure built-in)
   - nick:plan and nick:execute already demonstrate this pattern
   - Frontmatter provides routing, body provides core process, references/ provides detail
   - Natural fit with Claude Code's skill loading mechanism

8. **Per-phase requirements files** (reduce full-project file loading)
   - Split REQUIREMENTS.md into per-phase files when master exceeds 400 lines
   - Each agent only reads its phase's requirements file
   - Eliminates the need for extraction commands

9. **Agent return protocol optimization** (prevent orchestrator context bloat)
   - Agents return structured signals (status + path) not content
   - Orchestrator reads SUMMARY.md only for spot-checks, does not store in context
   - Enforce the 10-15% orchestrator context rule

### Summary of Expected Total Savings

| Workflow | Current Est. (tokens) | After Tier 1 | After Tier 1+2 | After All |
|----------|----------------------|-------------|----------------|-----------|
| plan-phase (standard) | 100,000-150,000 | 50,000-70,000 | 35,000-50,000 | 25,000-40,000 |
| plan-phase (with revision) | 150,000-200,000 | 75,000-100,000 | 50,000-70,000 | 35,000-55,000 |
| execute-phase (3 plans) | 80,000-120,000 | 50,000-70,000 | 35,000-50,000 | 25,000-40,000 |
| Full phase cycle (plan+execute) | 200,000-300,000 | 110,000-150,000 | 75,000-105,000 | 55,000-85,000 |

**Bottom line: Tier 1 alone cuts token consumption roughly in half. All tiers together achieve 60-75% reduction.**
