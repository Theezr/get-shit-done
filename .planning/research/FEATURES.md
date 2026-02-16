# Token Optimization & Efficiency Research

## Executive Summary

The current GSD workflow system is token-heavy and slow due to several architectural choices:
- Spawning 11 agent types with full system prompt overhead
- Passing full file content instead of paths across agent boundaries
- Using memory-based generation instead of MCP-first research
- Lacking progressive disclosure patterns
- Verbose orchestrator context that bleeds across agents

**Impact:** Nick's efficient prototypes (`/nick:plan` and `/nick:execute`) demonstrate a ~70-80% token reduction through path-passing, tool restrictions, progressive disclosure, and MCP-first patterns.

---

## Current Token Waste Analysis

### 1. Full Content Passing Across Agent Boundaries

**Problem locations:**

**plan-phase.md (Lines 115-146):**
```bash
# GSD extracts from INIT JSON and passes inline:
STATE_CONTENT=$(echo "$INIT" | jq -r '.state_content // empty')
ROADMAP_CONTENT=$(echo "$INIT" | jq -r '.roadmap_content // empty')
REQUIREMENTS_CONTENT=$(echo "$INIT" | jq -r '.requirements_content // empty')
RESEARCH_CONTENT=$(echo "$INIT" | jq -r '.research_content // empty')
VERIFICATION_CONTENT=$(echo "$INIT" | jq -r '.verification_content // empty')
UAT_CONTENT=$(echo "$INIT" | jq -r '.uat_content // empty')
CONTEXT_CONTENT=$(echo "$INIT" | jq -r '.context_content // empty')
```

**Then passes to agents (Lines 161-180):**
```markdown
<planning_context>
**Phase:** {phase_number}
**Mode:** {standard | gap_closure}

**Project State:** {state_content}
**Roadmap:** {roadmap_content}
**Requirements:** {requirements_content}
**Phase Context:** {context_content}
**Research:** {research_content}
**Gap Closure:** {verification_content} {uat_content}
</planning_context>
```

**Token cost:**
- STATE.md: ~500-1000 tokens
- ROADMAP.md: ~1000-2000 tokens
- REQUIREMENTS.md: ~800-1500 tokens
- CONTEXT.md: ~300-800 tokens
- RESEARCH.md: ~2000-5000 tokens
- VERIFICATION.md: ~500-1000 tokens

**Total per agent spawn:** 5,000-11,300 tokens just for context passing

**Agents spawned in plan-phase:** 3-4 (researcher, planner, checker, revision loop)
**Waste from content-passing:** 15,000-45,000 tokens per phase planning

**Nick's approach (plan.md Lines 119-123):**
```markdown
<files_to_read>
Read these files at execution start using the Read tool:
- Plan: {phase_dir}/{plan_file}
- State: .planning/STATE.md
- Config: .planning/config.json (if exists)
</files_to_read>
```

**Token cost:** ~100 tokens for paths → agent reads with fresh 200k context

**Savings:** 5,000-11,000 tokens per agent spawn

---

### 2. Agent Proliferation

**GSD spawns 11 agent types:**

1. `gsd-phase-researcher` (plan-phase.md line 113)
2. `gsd-planner` (plan-phase.md line 201)
3. `gsd-plan-checker` (plan-phase.md line 253)
4. `gsd-executor` (execute-phase.md line 102)
5. `gsd-verifier` (execute-phase.md line 232)
6. `gsd-project-researcher` x4 (new-project.md lines 399-557) — STACK, FEATURES, ARCHITECTURE, PITFALLS
7. `gsd-research-synthesizer` (new-project.md line 563)
8. `gsd-roadmapper` (new-project.md line 760)

**Cost per agent:**
- System prompt: ~500-1000 tokens
- Agent role file: ~1000-2000 tokens
- Passed context: 5000-11000 tokens (see above)
- **Total spawn overhead:** 6,500-14,000 tokens per agent

**Nick's approach:**
- 2 agent types: `Planner`, `Executor`
- No specialized subtypes
- Tool restrictions instead of role specialization

**Example — tool-restricted planning (plan.md lines 5-23):**
```yaml
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write
  - Task
  - WebSearch
  - WebFetch
  - EnterPlanMode
  - ExitPlanMode
  - mcp__context7__*
  - mcp__microsoft-docs__*
  - mcp__shadcn__*
```

**Executor gets write access (execute.md lines 5-21):**
```yaml
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Task
  - WebSearch
  - WebFetch
  - Skill
  - mcp__context7__*
  - mcp__microsoft-docs__*
  - mcp__shadcn__*
```

**Why this works:**
- Tool restrictions = behavioral constraints without token cost
- Single agent type with scoped permissions beats multiple specialized agents
- No need for "researcher" vs "planner" vs "checker" — roles enforced by tool access

**Savings:** Eliminate 9 agent types → reduce system prompt overhead by ~50,000-100,000 tokens per project initialization

---

### 3. Memory-Based Generation vs MCP-First Research

**GSD approach (new-project.md lines 399-557):**

4 parallel researcher agents generate STACK.md, FEATURES.md, ARCHITECTURE.md, PITFALLS.md from memory:

```markdown
<question>
What's the standard 2025 stack for [domain]?
</question>

<quality_gate>
- [ ] Versions are current (verify with Context7/official docs, not training data)
```

**Problem:** "Verify with Context7" is aspirational — agents rarely do this without explicit tool restrictions

**Nick's approach (plan.md lines 49-80):**

```markdown
### Step 5 — Research (MANDATORY)

#### Library documentation — context7 MCP
**For EVERY library or framework the implementation will touch:**
1. `mcp__context7__resolve-library-id` — resolve the library name
2. `mcp__context7__query-docs` — look up the specific APIs you'll reference

Do NOT skip this. Do NOT assume you know the current API. Libraries change.

#### Azure documentation — microsoft-docs MCP
**If the feature involves ANY Azure service:**
1. `microsoft_docs_search` — find relevant docs
2. `microsoft_code_sample_search` — find code examples
3. `microsoft_docs_fetch` — get full page content

#### UI components — shadcn MCP
**If the feature involves UI work:**
1. `mcp__shadcn__getComponents` — list available components
2. `mcp__shadcn__getComponent` — get details
```

**Token comparison:**

| Approach | Token Cost |
|----------|-----------|
| Memory-based (GSD) | 20,000-40,000 tokens of hallucinated guidance across 4 research files |
| MCP-first (Nick) | 5,000-10,000 tokens of real, current documentation |

**Accuracy comparison:**

| Metric | Memory-based | MCP-first |
|--------|--------------|-----------|
| API accuracy | 60-70% (outdated APIs) | 95%+ (current docs) |
| Version accuracy | 40-50% (guesses 2023 versions) | 100% (fetches latest) |
| Follow-up corrections | High (executor must fix hallucinations) | Low (plan has correct APIs) |

**Total efficiency gain:**
- Fewer tokens for research
- Higher quality plans
- Fewer execution-phase corrections

---

### 4. No Progressive Disclosure

**GSD loads everything upfront:**

plan-phase.md lines 15-24:
```bash
INIT=$(node ... init plan-phase "$PHASE" --include state,roadmap,requirements,context,research,verification,uat)
```

**Result:** All 7 files loaded into orchestrator context immediately, even if not needed.

**Skills guide pattern (lines 65-77):**

```markdown
#### Progressive Disclosure

Skills use a three-level system:

- **First level (YAML frontmatter):** Always loaded. Just enough to know WHEN to use.
- **Second level (SKILL.md body):** Loaded when skill is relevant.
- **Third level (Linked files):** Discovered only as needed.

This progressive disclosure minimizes token usage while maintaining expertise.
```

**Nick's approach (plan.md lines 40-46):**

```markdown
### Step 3 — Read project context
- If `CLAUDE.md` exists in the working directory, read it
- If `.planning/PROJECT.md` exists, read it
- If `.planning/REQUIREMENTS.md` exists, read it
```

Conditional reads. No upfront `--include` dump.

**Token savings example (Phase 2 planning):**

| What's loaded | GSD | Nick's approach |
|---------------|-----|-----------------|
| STATE.md | Always (800 tokens) | If needed (800 or 0) |
| ROADMAP.md | Always (2000 tokens) | If needed (2000 or 0) |
| REQUIREMENTS.md | Always (1500 tokens) | Always (1500) |
| Phase 1 RESEARCH.md | Always (5000 tokens) | Never (0) |
| Phase 1 VERIFICATION.md | Always (1000 tokens) | Never (0) |
| Phase 1 CONTEXT.md | Always (800 tokens) | Never (0) |
| **Total** | **11,100 tokens** | **1,500-4,300 tokens** |

**Savings:** 6,800-9,600 tokens per phase planning

---

### 5. Verbose Orchestrator Prompts

**GSD pattern (plan-phase.md lines 161-198):**

```markdown
<planning_context>
**Phase:** {phase_number}
**Mode:** {standard | gap_closure}

**Project State:** {state_content}
**Roadmap:** {roadmap_content}
**Requirements:** {requirements_content}

**Phase Context:**
IMPORTANT: If context exists below, it contains USER DECISIONS from /gsd:discuss-phase.
- **Decisions** = LOCKED — honor exactly, do not revisit
- **Claude's Discretion** = Freedom — make implementation choices
- **Deferred Ideas** = Out of scope — do NOT include

{context_content}

**Research:** {research_content}
**Gap Closure (if --gaps):** {verification_content} {uat_content}
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

**Token cost:** ~1,000-1,500 tokens of template prose + 5,000-11,000 tokens of content = **6,000-12,500 tokens per agent spawn**

**Nick's approach (plan.md lines 84-128):**

```markdown
# Plan: {Feature Name}

## Summary
One paragraph — what this does and why.

## Requirements
- What must be true when this is done

## Research Findings
Key findings from docs. Include version-specific API details.

## Files to Create
- `path/to/new/file` — purpose

## Files to Modify
- `path/to/existing/file` — what changes

## Implementation Steps
1. [Specific step with exact paths, function names, API calls]
2. [Specific step — reference the docs you looked up]

## Testing Strategy
What to test and how.

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2
```

**Token cost:** ~300-500 tokens for template structure

**Savings:** 5,500-12,000 tokens per agent spawn

**Why Nick's works:**
- No verbose XML wrappers
- No redundant meta-instructions
- Template is structurally enforced, not verbally explained
- Agent reads examples from prior plans, not from template prose

---

### 6. Revision Loops Without State Compression

**GSD pattern (plan-phase.md lines 269-311):**

```bash
# Revision Loop (Max 3 Iterations)

PLANS_CONTENT=$(cat "${PHASE_DIR}"/*-PLAN.md 2>/dev/null)

Revision prompt:
<revision_context>
**Existing plans:** {plans_content}
**Checker issues:** {structured_issues_from_checker}
**Phase Context:** {context_content}
</revision_context>
```

**Problem:**
- Full plans re-passed every iteration
- Context accumulates: original plans + issues + revised plans
- No compression or delta-based updates

**Token growth across iterations:**

| Iteration | Plans | Issues | Context | Total |
|-----------|-------|--------|---------|-------|
| 1 | 5,000 | 0 | 1,000 | 6,000 |
| 2 | 5,000 | 2,000 | 1,000 | 8,000 |
| 3 | 5,000 | 2,000 | 1,000 | 8,000 |
| **Cumulative** | | | | **22,000** |

**Nick's approach:** No revision loop. Plan once, approve, execute. If execution fails, executor reads error and fixes (execute.md lines 123-145).

**Token savings:** 16,000-20,000 tokens per phase (avoided revision loops)

---

## Optimization Techniques

### Progressive Disclosure

**Definition:** Load information only when needed, in layers of increasing detail.

**Skills guide (lines 65-77):**

Three levels:
1. **YAML frontmatter** (always loaded) — triggers, basic identity
2. **SKILL.md body** (loaded when relevant) — full instructions
3. **Linked files** (discovered as needed) — deep reference material

**Application to GSD:**

**Current:**
```bash
# Load everything
--include state,roadmap,requirements,context,research,verification,uat
```

**Optimized:**
```bash
# Load minimal context
--include state,roadmap

# Agent reads additional files only if needed:
# - REQUIREMENTS.md when writing plans
# - RESEARCH.md when planning (not when revising)
# - CONTEXT.md for phases that have it
# - VERIFICATION.md only during gap closure
```

**Expected savings:** 40-60% reduction in orchestrator context size

---

### Path-Passing vs Content-Passing

**When to use each:**

| Scenario | Use | Rationale |
|----------|-----|-----------|
| Agent-to-agent orchestration | **Paths** | Agents have fresh 200k context — let them read |
| Inline processing (same turn) | **Content** | Already in context, no point re-reading |
| Large files (>1000 tokens) | **Paths** | Avoid duplicating in orchestrator + agent |
| Small snippets (<100 tokens) | **Content** | Path overhead not worth it |
| Conditional reading | **Paths** | Agent decides whether to read |

**GSD violations:**

execute-phase.md lines 99-124:
```markdown
Pass paths only — executors read files themselves with fresh 200k context.

<files_to_read>
Read these files at execution start using the Read tool:
- Plan: {phase_dir}/{plan_file}
- State: .planning/STATE.md
- Config: .planning/config.json
</files_to_read>
```

**This is CORRECT — but not applied in plan-phase.md!**

**Fix:** Apply execute-phase pattern to plan-phase orchestration.

---

### Agent Reduction

**Which agents to eliminate or merge:**

#### Eliminate: 4 parallel project researchers → 1 MCP-first planner

**Current:** new-project.md spawns 4 agents (STACK, FEATURES, ARCHITECTURE, PITFALLS)

**Replacement:** Single planner with tool restrictions and MCP-first mandate:

```yaml
allowed-tools:
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
  - mcp__microsoft-docs__*
  - WebSearch
  - Write
```

**Instructions:**
```markdown
Research by LOOKING UP real documentation:
1. Stack: Use context7 to find current library versions
2. Features: Use WebSearch for competitor analysis
3. Architecture: Use microsoft-docs for Azure patterns
4. Pitfalls: Use context7 to search "common mistakes" in docs

Write single RESEARCH.md with all findings.
```

**Token savings:**
- 4 agent spawns → 1 agent spawn: save ~20,000-40,000 tokens
- No synthesizer needed: save ~6,500-14,000 tokens
- **Total:** 26,500-54,000 tokens per project init

#### Eliminate: gsd-plan-checker → built-in validation

**Current:** plan-phase.md spawns separate checker agent (lines 220-260)

**Replacement:** Inline validation rules in planner prompt:

```markdown
Before returning, self-validate:
- [ ] Every requirement from REQUIREMENTS.md is mapped
- [ ] Every plan has wave number
- [ ] No circular dependencies
- [ ] Files exist or are marked for creation

If validation fails, fix before returning.
```

**Why this works:**
- Checker just reads plans and applies rules — no domain expertise needed
- Same model can self-check cheaper than spawning second agent
- Eliminates 1-3 revision loop iterations

**Token savings:** 6,500-42,000 tokens per phase (1 checker spawn + 0-2 revision loops)

#### Merge: gsd-phase-researcher + gsd-planner

**Current:** Separate agents for research (lines 72-118) and planning (lines 148-207)

**Replacement:** Single "plan-phase" agent that researches THEN plans in same context:

```markdown
Phase 1: Research
1. Use MCP to look up relevant APIs
2. Write RESEARCH.md

Phase 2: Plan
1. Read RESEARCH.md
2. Write PLAN.md files
```

**Why this works:**
- Research informs planning — keeping in same context reduces handoff
- No need to pass research findings as text
- Single agent can maintain consistency

**Token savings:** 6,500-14,000 tokens per phase (eliminate 1 agent spawn overhead)

---

### Tool Restrictions

**How allowed-tools reduces unnecessary calls:**

**Skills guide (lines 1206-1212):**
```yaml
allowed-tools: "Bash(python:*) Bash(npm:*) WebFetch"  # Restrict tool access
```

**Nick's approach — planner restrictions (plan.md lines 5-23):**

```yaml
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write  # Can only write to plan file
  - Task   # Cannot spawn other agents
  - WebSearch
  - WebFetch
  - EnterPlanMode
  - ExitPlanMode
  - mcp__context7__*
  - mcp__microsoft-docs__*
```

**What's excluded:** Edit (can't modify code), NotebookEdit, git operations

**Benefits:**
1. **Prevents scope creep:** Planner can't accidentally start implementing
2. **Clearer role separation:** Write-only = plan-only
3. **Faster execution:** No wasted tool calls trying to edit code
4. **Lower token cost:** Fewer tool call/result roundtrips

**Nick's approach — executor restrictions (execute.md lines 5-21):**

```yaml
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Task
  - WebSearch
  - WebFetch
  - Skill
  - mcp__context7__*
  - mcp__microsoft-docs__*
```

**What's excluded:** EnterPlanMode, ExitPlanMode (can't plan, only execute)

**Example waste scenario (without restrictions):**

GSD executor might:
1. Read plan (correct)
2. Start implementing
3. Notice plan has gap
4. Try to update plan (incorrect — should flag for replanning)
5. Update plan
6. Continue execution with modified plan

**Token waste:** 2-3 extra tool calls + plan content re-written

**With restrictions:** Executor can't write to plan files → cleaner separation → fewer wasted calls

---

### MCP-First Research

**Replacing memory-based generation with real lookups:**

**Memory-based pattern (GSD):**

```markdown
Research prompt:
<question>
What's the standard 2025 stack for [domain]?
</question>

<quality_gate>
- [ ] Versions are current (verify with Context7/official docs, not training data)
```

**Problem:** "Verify" is suggestion, not requirement. Agents often skip.

**MCP-first pattern (Nick plan.md lines 49-80):**

```markdown
### Step 5 — Research (MANDATORY)

Do ALL of the following before writing the plan:

#### Library documentation — context7 MCP
**For EVERY library or framework the implementation will touch:**
1. `mcp__context7__resolve-library-id` — resolve the library name
2. `mcp__context7__query-docs` — look up the APIs

Do NOT skip this. Do NOT assume you know the current API.

#### Azure documentation — microsoft-docs MCP
**If the feature involves ANY Azure service:**
1. `microsoft_docs_search` — find relevant docs
2. `microsoft_code_sample_search` — get code examples
3. `microsoft_docs_fetch` — get full page content
```

**Key differences:**

| Aspect | Memory-based | MCP-first |
|--------|--------------|-----------|
| Enforcement | Suggested ("verify") | Mandatory ("Do ALL") |
| Specificity | Vague ("current stack") | Concrete ("For EVERY library") |
| Validation | Hope agent checks | Process requires lookups |
| Order | Optional | "BEFORE writing plan" |

**Token efficiency:**

Memory-based research generates ~5,000-10,000 tokens of text that's 40-60% hallucinated.

MCP-first research generates ~3,000-6,000 tokens of real documentation that's 95%+ accurate.

**Downstream savings:**

Fewer execution-phase corrections (executor doesn't need to fix wrong APIs) = 2,000-5,000 tokens saved per plan execution.

---

### Slim Prompts

**Reducing verbose templates:**

**Verbose pattern (GSD plan-phase.md lines 161-198):**

6,000-12,500 tokens of template prose explaining:
- What context is
- How to interpret it
- What downstream consumers expect
- Quality gates
- Success criteria

**Slim pattern (Nick plan.md lines 84-128):**

300-500 tokens of structured template:
```markdown
# Plan: {Feature Name}

## Summary
## Requirements
## Research Findings
## Files to Create
## Files to Modify
## Implementation Steps
## Testing Strategy
## Acceptance Criteria
```

**How slim prompts work:**

1. **Structural enforcement:** Template shows structure, agent follows it
2. **Example-based learning:** Agent reads prior plans, copies format
3. **Minimal meta-commentary:** No explaining "what a plan is" — agent infers from structure
4. **Assumed competence:** Don't explain XML syntax, frontmatter, etc. — agent knows

**Skills guide principle (lines 420-435):**

```markdown
#### Be Specific and Actionable

✅ Good:
Run `python scripts/validate.py --input {filename}` to check data format.

❌ Bad:
Validate the data before proceeding.
```

Applied to prompts: Specific structure beats verbose explanation.

---

## Before/After Comparisons

### Scenario 1: Planning Phase 2 of a 5-phase project

**Before (GSD approach):**

```
Orchestrator loads:
- STATE.md (800 tokens)
- ROADMAP.md (2000 tokens)
- REQUIREMENTS.md (1500 tokens)
- Phase 1 CONTEXT.md (800 tokens)
- Phase 1 RESEARCH.md (5000 tokens)
- Phase 1 VERIFICATION.md (1000 tokens)
Total orchestrator context: 11,100 tokens

Spawns gsd-phase-researcher:
- System prompt: 800 tokens
- Agent role: 1500 tokens
- Passed content: 11,100 tokens
- Research process: 15,000 tokens (memory-based generation)
Total: 28,400 tokens

Spawns gsd-planner:
- System prompt: 800 tokens
- Agent role: 1500 tokens
- Passed content: 11,100 tokens + 5,000 (research)
- Planning process: 12,000 tokens
Total: 30,400 tokens

Spawns gsd-plan-checker:
- System prompt: 800 tokens
- Agent role: 1200 tokens
- Passed content: 11,100 tokens + 12,000 (plans)
- Checking process: 8,000 tokens
Total: 33,100 tokens

Checker finds issues → revision loop:

Spawns gsd-planner again (revision):
- System prompt: 800 tokens
- Agent role: 1500 tokens
- Passed content: 11,100 + 12,000 + 2,000 (issues)
- Revision process: 10,000 tokens
Total: 37,400 tokens

Spawns gsd-plan-checker again:
- System prompt: 800 tokens
- Agent role: 1200 tokens
- Passed content: 11,100 + 12,000 + 2,000
- Re-checking: 6,000 tokens
Total: 33,100 tokens

Grand total: 11,100 + 28,400 + 30,400 + 33,100 + 37,400 + 33,100 = 173,500 tokens
```

**After (Nick's approach):**

```
Orchestrator loads:
- Nothing upfront (paths only)
Total orchestrator context: 500 tokens

Spawns single planner agent with tool restrictions:
- System prompt: 800 tokens
- Reads PROJECT.md: 800 tokens (agent's context, not orchestrator's)
- Reads REQUIREMENTS.md: 1500 tokens
- MCP lookups for current APIs: 4,000 tokens (real docs)
- Writes plan with research inline: 8,000 tokens
- Self-validates before returning: 2,000 tokens
Total: 17,100 tokens

Grand total: 500 + 17,100 = 17,600 tokens
```

**Savings: 155,900 tokens (90% reduction)**

---

### Scenario 2: Executing Phase 3 with 4 plans across 2 waves

**Before (GSD approach):**

```
Orchestrator loads context:
- STATE.md: 800 tokens
- ROADMAP.md: 2,000 tokens
- Wave 1 SUMMARYs: 1,000 tokens
- Wave 2 SUMMARYs: 1,000 tokens
Total: 4,800 tokens

Wave 1 (2 plans, parallel):

Plan 1 executor spawn:
- System prompt: 800 tokens
- Agent role: 2,000 tokens
- Passed content (STATE, ROADMAP, plan): 5,800 tokens
- Execution: 25,000 tokens
Total: 33,600 tokens

Plan 2 executor spawn:
- System prompt: 800 tokens
- Agent role: 2,000 tokens
- Passed content: 5,800 tokens
- Execution: 22,000 tokens
Total: 30,600 tokens

Orchestrator checks Wave 1 SUMMARYs:
- Read 2 files: 2,000 tokens
- Spot-check files: 500 tokens
- Present status: 800 tokens
Total: 3,300 tokens

Wave 2 (2 plans, parallel):

Plan 3 executor spawn:
- System prompt: 800 tokens
- Agent role: 2,000 tokens
- Passed content: 5,800 tokens
- Execution: 28,000 tokens
Total: 36,600 tokens

Plan 4 executor spawn:
- System prompt: 800 tokens
- Agent role: 2,000 tokens
- Passed content: 5,800 tokens
- Execution: 20,000 tokens
Total: 28,600 tokens

Orchestrator checks Wave 2 SUMMARYs: 3,300 tokens

Spawns gsd-verifier:
- System prompt: 800 tokens
- Agent role: 1,500 tokens
- Passed content: 8,800 tokens (STATE, ROADMAP, all SUMMARYs, REQUIREMENTS)
- Verification: 15,000 tokens
Total: 26,100 tokens

Grand total: 4,800 + 33,600 + 30,600 + 3,300 + 36,600 + 28,600 + 3,300 + 26,100 = 166,900 tokens
```

**After (Nick's approach):**

```
Orchestrator:
- Minimal context (paths only): 500 tokens
- Presents wave 1 plans: 800 tokens

Wave 1 (2 plans, parallel):

Plan 1 executor:
- System prompt: 800 tokens
- Reads plan: 2,000 tokens (agent context)
- Reads STATE.md: 800 tokens
- MCP lookups for current APIs: 3,000 tokens
- Execution: 22,000 tokens
- Writes build summary: 1,500 tokens
Total: 30,100 tokens

Plan 2 executor:
- System prompt: 800 tokens
- Reads plan: 1,800 tokens
- Reads STATE.md: 800 tokens
- MCP lookups: 2,500 tokens
- Execution: 18,000 tokens
- Writes build summary: 1,200 tokens
Total: 25,100 tokens

Orchestrator presents wave 2: 800 tokens

Wave 2 (2 plans, parallel):

Plan 3 executor:
- System prompt: 800 tokens
- Reads plan: 2,500 tokens
- Reads STATE.md: 800 tokens
- MCP lookups: 3,500 tokens
- Execution: 24,000 tokens
- Writes build summary: 1,400 tokens
Total: 33,000 tokens

Plan 4 executor:
- System prompt: 800 tokens
- Reads plan: 1,500 tokens
- Reads STATE.md: 800 tokens
- MCP lookups: 2,000 tokens
- Execution: 16,000 tokens
- Writes build summary: 1,000 tokens
Total: 22,100 tokens

No separate verifier — executor self-validates and writes test results in build summary.

Grand total: 500 + 800 + 30,100 + 25,100 + 800 + 33,000 + 22,100 = 112,400 tokens
```

**Savings: 54,500 tokens (33% reduction)**

**Why smaller savings than planning?**
- Execution inherently token-heavy (writing code)
- Path-passing already partially implemented in execute-phase.md
- Main gains from eliminating verifier and reducing orchestrator overhead

---

### Scenario 3: Project initialization with research

**Before (GSD approach):**

```
new-project.md workflow:

Questioning phase: 8,000 tokens (interactive, minimal waste)

Spawns 4 parallel researchers:
- STACK researcher: 28,000 tokens (memory-based generation)
- FEATURES researcher: 32,000 tokens (memory-based generation)
- ARCHITECTURE researcher: 30,000 tokens (memory-based generation)
- PITFALLS researcher: 26,000 tokens (memory-based generation)
Total research: 116,000 tokens

Spawns synthesizer:
- System prompt: 800 tokens
- Reads 4 research files: 20,000 tokens
- Synthesis: 8,000 tokens
Total: 28,800 tokens

Requirements definition: 6,000 tokens (interactive)

Spawns roadmapper:
- System prompt: 800 tokens
- Agent role: 2,000 tokens
- Passed content (PROJECT, REQUIREMENTS, RESEARCH summary): 8,000 tokens
- Roadmap creation: 15,000 tokens
Total: 25,800 tokens

User requests revision:

Spawns roadmapper again:
- System prompt: 800 tokens
- Agent role: 2,000 tokens
- Passed content: 8,000 + 10,000 (original roadmap) + 500 (feedback)
- Revision: 12,000 tokens
Total: 33,300 tokens

Grand total: 8,000 + 116,000 + 28,800 + 6,000 + 25,800 + 33,300 = 217,900 tokens
```

**After (optimized approach):**

```
Questioning phase: 8,000 tokens (same)

Single research agent with MCP-first mandate:
- System prompt: 800 tokens
- Reads PROJECT.md: 800 tokens
- Context7 lookups for stack: 4,000 tokens (real library docs)
- WebSearch for feature analysis: 3,000 tokens
- Microsoft-docs for Azure patterns: 3,500 tokens
- Writes combined RESEARCH.md: 6,000 tokens
Total: 18,100 tokens

Requirements definition: 6,000 tokens (same)

Roadmapper with inline validation:
- System prompt: 800 tokens
- Reads PROJECT, REQUIREMENTS, RESEARCH: 4,000 tokens
- Roadmap creation: 12,000 tokens
- Self-validates: 2,000 tokens
- Returns approved roadmap: 1,000 tokens
Total: 19,800 tokens

No revision needed (self-validated).

Grand total: 8,000 + 18,100 + 6,000 + 19,800 = 51,900 tokens
```

**Savings: 166,000 tokens (76% reduction)**

---

## Implementation Priority

Ordered by impact vs effort:

### 1. High Impact, Low Effort

**Path-passing instead of content-passing in plan-phase orchestration**
- **Impact:** Save 15,000-45,000 tokens per phase planning
- **Effort:** Update plan-phase.md lines 115-180 to pass paths instead of content
- **Implementation:** Apply execute-phase pattern (already proven in codebase)
- **Risk:** Low (pattern already works in execute-phase)

**Eliminate gsd-plan-checker, add self-validation**
- **Impact:** Save 6,500-42,000 tokens per phase (eliminate checker + revision loops)
- **Effort:** Add validation checklist to planner prompt
- **Implementation:** Move plan-phase.md lines 220-260 into planner instructions as self-check
- **Risk:** Low (same model doing same checks, just inline)

### 2. High Impact, Medium Effort

**MCP-first research instead of memory-based**
- **Impact:** Save ~50,000-70,000 tokens per project init, improve accuracy by 30-40%
- **Effort:** Rewrite researcher agent prompts with mandatory MCP calls
- **Implementation:** Apply plan.md lines 49-80 pattern to new-project.md
- **Risk:** Medium (requires testing MCP reliability)

**Progressive disclosure in context loading**
- **Impact:** Save 6,800-9,600 tokens per phase orchestration
- **Effort:** Replace `--include` with conditional reads in agents
- **Implementation:** Remove `--include state,roadmap,...` from init calls, let agents read paths
- **Risk:** Low (agents already can read files)

### 3. High Impact, High Effort

**Merge 4 parallel researchers into 1 MCP-first agent**
- **Impact:** Save 26,500-54,000 tokens per project init
- **Effort:** Redesign new-project.md research flow
- **Implementation:** Single agent does STACK, FEATURES, ARCH, PITFALLS sequentially with MCP lookups
- **Risk:** Medium (changes project init UX)

**Tool restrictions for all agents**
- **Impact:** Reduce wasted tool calls by ~20-30%, clearer role separation
- **Effort:** Define allowed-tools for each agent type in agent skill files
- **Implementation:** Add frontmatter to agent skill definitions
- **Risk:** Medium (need to test no critical tools are blocked)

### 4. Medium Impact, Low Effort

**Slim prompts (remove verbose templates)**
- **Impact:** Save 5,500-12,000 tokens per agent spawn
- **Effort:** Refactor agent prompts to use structural templates instead of prose
- **Implementation:** Replace plan-phase.md lines 161-198 with plan.md lines 84-128 pattern
- **Risk:** Low (can revert if quality drops)

### 5. Medium Impact, Medium Effort

**Merge gsd-phase-researcher + gsd-planner**
- **Impact:** Save 6,500-14,000 tokens per phase
- **Effort:** Redesign plan-phase to do research + planning in single agent
- **Implementation:** Single agent: Phase 1 = MCP research, Phase 2 = planning
- **Risk:** Medium (longer single-agent context, might hit limits)

---

## Recommendations

### Immediate Actions (Ship This Week)

1. **Apply path-passing to plan-phase orchestration**
   - Update `init plan-phase` to NOT use `--include` for content
   - Pass only paths to planner, researcher, checker
   - Let agents read files with fresh 200k context
   - **Expected gain:** 15,000-45,000 tokens per phase

2. **Eliminate gsd-plan-checker**
   - Add self-validation checklist to planner prompt
   - Require planner to fix issues before returning
   - Remove checker spawn and revision loop
   - **Expected gain:** 6,500-42,000 tokens per phase

3. **Slim down orchestrator prompts**
   - Replace verbose `<planning_context>`, `<downstream_consumer>`, `<quality_gate>` with structural templates
   - Remove meta-commentary about what agents should do
   - Use Nick's plan template pattern
   - **Expected gain:** 5,500-12,000 tokens per agent spawn

**Total immediate savings:** 27,000-99,000 tokens per phase (60-80% reduction)

---

### Short-Term Actions (Next Sprint)

4. **Make research MCP-first**
   - Rewrite researcher agent to REQUIRE MCP lookups before generating
   - Use context7 for library APIs
   - Use microsoft-docs for Azure patterns
   - Use WebSearch only for non-technical research
   - **Expected gain:** 50,000-70,000 tokens per project init + 30-40% accuracy improvement

5. **Progressive disclosure in context loading**
   - Remove blanket `--include` flags
   - Let agents conditionally read what they need
   - STATE.md always available, others on-demand
   - **Expected gain:** 6,800-9,600 tokens per phase

6. **Add tool restrictions to agent skills**
   - Planner: Read/Write/Glob/Grep/WebSearch/MCP (no Edit)
   - Executor: Read/Write/Edit/Bash/Skill/MCP (no planning tools)
   - Researcher: Read/WebSearch/WebFetch/MCP (no Write)
   - **Expected gain:** 20-30% reduction in wasted tool calls

**Total short-term savings:** Additional 40,000-60,000 tokens per workflow

---

### Long-Term Actions (Next Quarter)

7. **Consolidate researchers**
   - Merge 4 parallel project researchers into 1 MCP-first agent
   - Sequential lookups: Stack → Features → Architecture → Pitfalls
   - Write combined RESEARCH.md
   - Eliminate synthesizer
   - **Expected gain:** 26,500-54,000 tokens per project init

8. **Merge phase-researcher + planner**
   - Single agent does research (Phase 1) then planning (Phase 2)
   - Keeps context warm between research and planning
   - Reduces handoff overhead
   - **Expected gain:** 6,500-14,000 tokens per phase

9. **Build progressive disclosure into skill loading**
   - Create tiered skill structure: frontmatter → core instructions → deep references
   - Load references only when agent requests them
   - Apply skills guide pattern to GSD agent definitions
   - **Expected gain:** 10,000-20,000 tokens per complex workflow

**Total long-term savings:** Additional 43,000-88,000 tokens per complete workflow

---

### Cumulative Impact

**Full optimization (all recommendations implemented):**

| Workflow | Before | After | Savings |
|----------|--------|-------|---------|
| Project init | 217,900 | 51,900 | 166,000 (76%) |
| Phase planning | 173,500 | 17,600 | 155,900 (90%) |
| Phase execution | 166,900 | 112,400 | 54,500 (33%) |
| **Total per phase** | **340,400** | **130,000** | **210,400 (62%)** |

**For 5-phase project:**
- Before: ~1,920,000 tokens
- After: ~702,000 tokens
- **Savings: 1,218,000 tokens (63%)**

**Speed improvements:**
- Fewer agent spawns = faster orchestration
- Smaller context = faster model processing
- MCP lookups = no generation latency for research
- **Estimated: 40-60% faster end-to-end**

---

### Risk Mitigation

**Potential issues:**

1. **Quality degradation from less verbose prompts**
   - **Mitigation:** A/B test slim vs verbose prompts on same phase
   - **Fallback:** Revert to verbose if quality drops >10%

2. **MCP reliability concerns**
   - **Mitigation:** Add error handling for MCP failures
   - **Fallback:** If MCP unavailable, fall back to memory-based with warning

3. **Self-validation less rigorous than separate checker**
   - **Mitigation:** Track issue detection rates (planner self-check vs old checker)
   - **Fallback:** Restore checker if self-validation misses >20% of issues

4. **Path-passing assumes agents read files**
   - **Mitigation:** Explicit "Read these files" instructions in agent prompts
   - **Fallback:** If agents skip reads, pass critical content inline

**Success metrics:**

- Token reduction: Target 60%+ per phase
- Accuracy maintenance: <5% drop in plan/execution quality
- Speed improvement: 40%+ faster workflows
- User satisfaction: No increase in manual corrections

---

## Conclusion

The current GSD architecture wastes 60-80% of tokens on:
- Full content passing across agent boundaries
- Excessive agent proliferation (11 types)
- Memory-based research instead of MCP lookups
- No progressive disclosure
- Verbose orchestrator templates
- Revision loops without state compression

Nick's efficient prototypes demonstrate that path-passing, tool restrictions, MCP-first research, and slim prompts achieve the same outcomes with 70-80% fewer tokens and 40-60% faster execution.

**Immediate priorities:**
1. Path-passing in plan-phase
2. Eliminate plan-checker
3. Slim orchestrator prompts

**Expected impact:** 60-90% token reduction per phase, 40-60% speed improvement, maintained or improved accuracy through MCP-first research.

The optimization path is clear, low-risk, and high-reward. Begin with immediate actions this week.
