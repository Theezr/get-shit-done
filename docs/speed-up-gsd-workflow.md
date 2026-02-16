# Speed Up GSD Workflow

## Context

The GSD pipeline runs 6 sequential steps per phase: **Research -> Plan -> Check -> Execute -> Verify -> Review**. Each step spawns a Task() agent that must complete before the next starts. A typical phase takes ~20-30 minutes. For a 5-phase project, that's 100-150+ minutes.

Two root causes:
1. **Sequential agent spawns** -- each step blocks on the previous, with agent startup overhead (read definition file, read project files, parse context)
2. **LLM agents doing deterministic work** -- plan validation, file parsing, and state lookups are done by expensive LLM agents when they could be instant code

This plan attacks both with a **custom MCP server** as the centerpiece, plus workflow restructuring.

---

## Strategy: Three Tiers

| Tier | What | Impact | Effort |
|------|------|--------|--------|
| **A. GSD MCP Server** | Replace file reads + deterministic agents with instant tool calls | 40-60% faster per-phase | 2-3 days |
| **B. Pipeline restructuring** | Merge steps, parallelize, auto-skip | 30-50% fewer agent spawns | 1-2 days |
| **C. Prompt trimming** | Slim agents, smarter models | 10-20% less overhead | 0.5 day |

---

## Tier A: Custom GSD MCP Server

### What it does

A persistent Node.js MCP server that:
1. **Watches `.planning/`** via fs.watch -- maintains parsed, cached state in memory
2. **Serves targeted queries** -- `get_phase_context(5)` returns just phase 5's goal + requirements + plan list as structured JSON, instead of agents reading 3-5 full markdown files
3. **Replaces the plan-checker agent entirely** -- deterministic structural validation in code (YAML parsing, cycle detection, task completeness, scope counting) runs in milliseconds instead of spawning a sonnet/haiku agent
4. **Caches research results** -- SQLite-backed cache for Context7/WebSearch/microsoft-docs results. Second+ agents querying the same library get cached results in 10ms instead of re-fetching
5. **Provides workflow intelligence** -- `next_action()` tells the orchestrator what step to run based on current state

### Architecture

```
Claude Code / Skills
       |
  stdio (JSON-RPC)
       |
  GSD MCP Server (Node.js)
  ├── File Watcher (.planning/)
  ├── In-Memory Cache (state, roadmap, phases)
  ├── SQLite (research cache, parse cache)
  ├── Plan Validator (deterministic checks)
  └── Workflow State Tracker
```

### MCP Tools to Build

| Tool | Replaces | Speed Gain |
|------|----------|-----------|
| `gsd_phase_context(N)` | 3-5 file reads per agent (STATE.md, ROADMAP.md, REQUIREMENTS.md, RESEARCH.md) | 95% faster (10ms vs 30-60sec) |
| `gsd_validate_plans(phase_dir)` | Entire plan-checker agent spawn | 99% faster (100ms vs 2-5min) |
| `gsd_check_complexity(phase_dir)` | Manual heuristics in orchestrator | Enables auto-skip |
| `gsd_research_lookup(library, query)` | Redundant Context7/WebSearch calls | 10ms cache hit vs 30-60sec API call |
| `gsd_research_store(library, query, result)` | Nothing (new capability) | Enables cross-agent research sharing |
| `gsd_workflow_status()` | Reading STATE.md + ROADMAP.md to determine next step | Instant JSON vs file parsing |
| `gsd_batch_context(file_paths[])` | Multiple sequential file reads | 1 tool call vs N tool calls |

### What `gsd_validate_plans` checks (replacing plan-checker agent)

These are all deterministic -- no LLM reasoning needed:
- YAML frontmatter parses correctly (wave, autonomous, type fields present)
- Each task has required XML elements (action, files, verify, done)
- Task count per plan <= 3 (scope check)
- No circular dependencies in wave ordering
- Every phase requirement from REQUIREMENTS.md appears in at least one task
- Must-haves section exists and is non-empty
- File paths in tasks reference real files or reasonable new paths

The ~30% of plan-checking that IS semantic (are must-haves truly user-observable? is the scope appropriate for the goal?) stays in the planner as a self-verification step.

### Implementation

**Files to create:**
- `get-shit-done/mcp-server/index.cjs` -- main server (~400 lines)
- `get-shit-done/mcp-server/package.json` -- dependencies
- `get-shit-done/mcp-server/lib/cache.cjs` -- file watcher + in-memory cache
- `get-shit-done/mcp-server/lib/validator.cjs` -- plan validation logic (extracted from plan-checker's 7 dimensions)
- `get-shit-done/mcp-server/lib/research-cache.cjs` -- SQLite research cache

**Files to modify:**
- `.claude/settings.json` or project `.mcp.json` -- register the MCP server
- `skills/nick-plan-phase/SKILL.md` -- replace file reads with `gsd_phase_context()`, replace plan-checker spawn with `gsd_validate_plans()`
- `skills/nick-execute-phase/SKILL.md` -- replace file reads with `gsd_phase_context()`, use `gsd_check_complexity()` for auto-skip
- Agent definitions that reference `<files_to_read>` sections -- agents can call `gsd_phase_context()` instead

**Dependencies:**
- `@modelcontextprotocol/sdk` -- MCP server framework
- `better-sqlite3` -- persistent research cache
- `js-yaml` -- YAML frontmatter parsing
- `chokidar` -- reliable file watching (better than fs.watch)

**Registration:**
```bash
claude mcp add --transport stdio --scope project \
  gsd-planning -- node ~/.claude/get-shit-done/mcp-server/index.cjs
```

### Research Cache Design

```
Researcher agent queries Context7 for "react-router"
  -> result cached in SQLite with 7-day TTL
  -> key: "context7:react-router:routing setup"

Planner agent needs "react-router" docs
  -> calls gsd_research_lookup("react-router", "routing setup")
  -> gets cached result in 10ms (no Context7 API call)

Executor agent needs same docs
  -> same cache hit, 10ms
```

Saves 5-10 redundant API calls per phase, ~30-60 seconds per agent.

---

## Tier B: Pipeline Restructuring

### B1. Merge plan-checker into planner + MCP validation

Two-part replacement:
1. **Structural checks** -> `gsd_validate_plans()` MCP tool (instant, deterministic)
2. **Semantic checks** -> planner self-verification (40-line addition to planner agent)

**Files:**
- `skills/nick-plan-phase/references/agents/nick-planner.md` -- add `<self_verification>` section with the semantic checks that need LLM reasoning
- `skills/nick-plan-phase/SKILL.md` -- replace steps 5-6 (checker spawn + revision loop) with single `gsd_validate_plans()` call. If structural issues found, tell planner to fix. If semantic concerns, planner already self-verified.
- Keep `--strict` flag to re-enable full external checker

**Saves:** 5-15 min/phase (eliminates 1-3 agent spawns + revision loops)

### B2. Parallel verify + review + prefetch

After execution completes, spawn THREE things simultaneously:
1. Verifier agent (goal checking)
2. Reviewer agent (code quality)
3. Next-phase researcher (background, if next phase exists)

**Files:**
- `skills/nick-execute-phase/SKILL.md` -- Step 5 becomes parallel spawn of verifier + reviewer + optional researcher. Wait for verifier + reviewer. Researcher runs in background.

**Saves:** 2-5 min/phase (overlap verify + review) + 2-5 min/transition (prefetch research)

### B3. Auto-skip based on complexity

`gsd_check_complexity()` MCP tool analyzes plans and returns complexity score.

Orchestrators use it:
- **trivial** (1 plan, 1-2 tasks, no UI): skip research + verify + review
- **simple** (1-2 plans, no UI): skip verify-work
- **complex** (3+ plans, UI, checkpoints): full pipeline

**Files:**
- `skills/nick-plan-phase/SKILL.md` -- check complexity, auto-skip research
- `skills/nick-execute-phase/SKILL.md` -- check complexity, auto-skip verify/review
- `.planning/config.json` -- add `"auto_skip": true`

**Saves:** 10-20 min for trivial phases

### B4. Unified command flow

User runs ONE command per phase: `/gsd:plan-phase N` then `/gsd:execute-phase N`. Execute-phase handles everything through review automatically.

**Files:**
- `skills/nick-execute-phase/SKILL.md` -- already handles this via B2 changes
- `skills/nick-review/SKILL.md` -- keep as standalone for `--gaps` scenarios

**Saves:** 2-3 min human latency (no more `/clear` + `/gsd:verify-work` + `/clear` + `/gsd:review`)

---

## Tier C: Prompt Trimming

### C1. Smarter model routing

Downgrade analytical agents that don't need creative reasoning:

```javascript
// In gsd-tools.cjs MODEL_PROFILES:
'gsd-plan-checker':       { quality: 'haiku', balanced: 'haiku', budget: 'haiku' },
'gsd-verifier':           { quality: 'haiku', balanced: 'haiku', budget: 'haiku' },
'gsd-phase-researcher':   { quality: 'sonnet', balanced: 'sonnet', budget: 'haiku' },
'gsd-project-researcher': { quality: 'sonnet', balanced: 'sonnet', budget: 'haiku' },
```

**File:** `get-shit-done/bin/gsd-tools.cjs` lines 126-137

### C2. Extract shared philosophy to reference files

Four agents repeat "Training = Hypothesis" philosophy (~30 lines each). Extract to shared reference.

**Create:** `get-shit-done/references/philosophy-research.md`, `get-shit-done/references/philosophy-solo-dev.md`
**Edit:** 4 agent definitions to @-reference instead of inline

---

## Execution Order

1. **Tier A: Build MCP server** (highest leverage -- one change affects every step)
   - Core server with file watcher + cache
   - Plan validator tool
   - Research cache with SQLite
   - Register with Claude Code

2. **Tier B1 + B2: Restructure pipeline** (uses MCP server tools)
   - Planner self-verification + MCP structural validation
   - Parallel verify + review + prefetch in execute-phase

3. **Tier B3 + B4: Auto-skip + unified flow**
   - Complexity scoring via MCP
   - Execute-phase runs full pipeline automatically

4. **Tier C: Trim** (polish)
   - Model routing updates
   - Shared philosophy extraction

---

## Expected Impact

| Scenario | Before | After | Savings |
|----------|--------|-------|---------|
| Trivial phase (config/rename) | ~20 min | ~3-5 min | 75-85% |
| Simple phase (no UI) | ~25 min | ~10 min | 60% |
| Complex phase (UI + tests) | ~30 min | ~18 min | 40% |
| 5-phase project total | ~130 min | ~55 min | 58% |

The MCP server alone accounts for ~30% of savings (eliminating file I/O overhead + replacing plan-checker agent). Pipeline restructuring accounts for another ~20%.

---

## Verification

1. **MCP server works:** `claude mcp` shows gsd-planning server, tools are callable
2. **Cache works:** Call `gsd_phase_context(N)` twice -- second call returns cached
3. **Validator works:** Create a plan with missing frontmatter -- `gsd_validate_plans()` catches it
4. **Research cache works:** Query Context7 via MCP, second call returns cached
5. **Plan-phase faster:** Run `/gsd:plan-phase` -- no checker agent spawned, MCP validates structurally
6. **Execute-phase unified:** Run `/gsd:execute-phase` -- verify + review run in parallel, results presented together
7. **Auto-skip works:** Trivial phase skips research/verify/review
8. **End-to-end timing:** Compare wall-clock for a full phase before/after

---

## Tier D: MCP-Driven Quality & Intelligence

Tiers A-C focus on speed. Tier D uses the same MCP architecture to improve **quality, token efficiency, structure, research, and context** -- making agents smarter, not just faster.

---

### Token Reduction

#### D1. Surgical context injection

**Problem:** Every agent gets the full ROADMAP.md, STATE.md, REQUIREMENTS.md dumped into its prompt, even when it only needs 10% of it.

**Solution:** `gsd_phase_context(N)` already exists (Tier A) -- extend it with a `fields` parameter:
```
gsd_phase_context(5, fields: ["goal", "requirements", "must_haves"])
```
Returns only the requested fields as structured JSON. Agents declare what they need; the MCP server delivers exactly that.

**Impact:** 40-60% fewer tokens per agent prompt. Planner doesn't need execution state. Verifier doesn't need research results. Each agent gets a tailored context slice.

#### D2. Kill the checker revision loop

**Problem:** Plan-checker finds issues -> planner revises -> checker re-checks -> planner re-revises. This loop sometimes runs 3x, each time re-reading all plans and re-generating full responses.

**Solution:** `gsd_validate_plans()` returns structured issues with exact locations. Planner fixes them in one pass. No re-check needed because structural validation is deterministic -- fix the issue, call validate again in 100ms.

**Impact:** Eliminates 2000-5000 tokens per revision round. Typical phase saves 4000-10000 tokens from killed revision loops.

#### D3. Smart history digest

**Problem:** When context compresses mid-conversation, agents lose important decisions made earlier. When context doesn't compress, old messages waste tokens.

**Solution:** New MCP tool `gsd_session_log(phase, event, data)` that persists key decisions to SQLite:
- Research conclusions ("chose React Router v7 because...")
- Plan revisions ("split plan into 2 because scope too large")
- Execution blockers ("API changed, had to use v2 endpoint")

Agents query `gsd_session_history(phase)` to get a compressed digest of prior decisions instead of relying on conversation history.

**Impact:** 20-30% fewer tokens from conversation history. Decisions survive context compression. New agents (verify, review) get prior context without reading full transcripts.

---

### Better Quality

#### D4. Cross-agent learning & conventions

**Problem:** Each agent discovers project conventions independently. Executor agent 1 figures out "this project uses barrel exports" -- executor agent 2 doesn't know and creates direct imports.

**Solution:** New MCP tool `gsd_conventions_store(convention, source_file)` and `gsd_conventions_get(category)`. Conventions are extracted during research/planning and served to all downstream agents:
- Code style patterns (naming, imports, exports)
- Architecture decisions (state management, routing, API patterns)
- Testing conventions (framework, file naming, assertion style)

**Impact:** Fewer code review findings. Agents produce consistent code on first pass. Review agent can validate against stored conventions instead of inferring them.

#### D5. Plan quality scoring

**Problem:** Plan quality varies. Some plans have vague tasks ("set up the component"), unclear verify steps, or missing edge cases. The only check is binary pass/fail from the checker.

**Solution:** `gsd_score_plan(plan_path)` returns a 0-100 quality score with dimensional breakdown:
- **Specificity** (0-25): Are tasks concrete enough to execute without interpretation?
- **Verifiability** (0-25): Can each verify step be objectively checked?
- **Completeness** (0-25): Do tasks cover all requirements?
- **Scope** (0-25): Is each task appropriately sized (not too large, not too small)?

Planner targets score >= 80 before submitting. Historical scores tracked per phase.

**Impact:** Consistent plan quality. Planner self-corrects before submission. Quality trends visible across project.

#### D6. Must-haves traceability

**Problem:** Must-haves are defined in plans but there's no systematic check that they survive execution. A must-have like "form validates email format" might get lost between plan and implementation.

**Solution:** `gsd_track_must_haves(phase)` returns status of each must-have:
- **planned**: defined in plan
- **implemented**: appears in committed code (grep-based detection)
- **verified**: confirmed by verifier agent

Verifier agent calls this to know exactly what to check. Review agent flags any must-have stuck at "planned".

**Impact:** Zero must-haves silently dropped. Verifier knows exactly what to test. Direct line from requirements -> plan -> code -> verification.

---

### Better Research

#### D7. Research dedup with project usage context

**Problem:** Research agent queries "react-router" generically. But the project already uses react-router v6 with specific patterns. Generic research wastes time on irrelevant versions/approaches.

**Solution:** Extend research cache (Tier A) with project context:
```
gsd_research_lookup("react-router", "routing setup", {
  project_version: "6.28.0",      // from package.json
  existing_patterns: ["createBrowserRouter", "useLoaderData"],  // from codebase grep
  exclude: ["v5", "v7-beta"]
})
```

MCP server pre-populates project context by scanning `package.json` and grepping imports.

**Impact:** Research results are immediately relevant. No time spent on wrong versions. Agents don't need to discover project patterns -- MCP server already knows them.

#### D8. Smart research routing

**Problem:** All research queries go through the same path regardless of type. A "how does React.memo work?" query (stable API, well-documented) gets the same treatment as "Azure Static Web Apps custom auth" (niche, version-sensitive).

**Solution:** `gsd_research_route(query)` classifies the query and routes it:
- **cached**: exact or near-match in research cache -> return immediately
- **stable-api**: well-known library, stable API -> Context7 only, cache aggressively (30-day TTL)
- **version-sensitive**: specific version or recent changes -> Context7 + WebSearch, short TTL (7 days)
- **platform-specific**: Azure/AWS/GCP -> Microsoft Docs MCP / provider docs, medium TTL (14 days)
- **novel**: no good cached or documented answer -> full research with WebSearch, short TTL

**Impact:** 50-70% of research queries answered from cache or single source. Only novel queries trigger expensive multi-source research.

#### D9. Research freshness tracking

**Problem:** Cached research has a fixed TTL but no awareness of whether the underlying library has changed. A 6-day-old cache entry for a library that released a breaking change yesterday is stale.

**Solution:** `gsd_research_freshness(library)` checks:
- Cache age vs TTL
- `npm info <package> time.modified` for last publish date
- If published after cache entry -> mark stale, trigger refresh

Run as background check when research cache is hit. Don't block on it -- serve cached result but flag if potentially stale.

**Impact:** Prevents stale research from causing bugs. Background refresh means no latency hit. Only re-fetches when something actually changed.

---

### More Structure

#### D10. Workflow state machine

**Problem:** Workflow state is tracked via string values in STATE.md ("researching", "planning", "executing"). Transitions are enforced by convention in skill prompts, not by code. Invalid transitions (e.g., "executing" -> "researching") are possible.

**Solution:** `gsd_workflow_transition(from, to, metadata)` enforces valid transitions:
```
researching -> planning -> executing -> verifying -> reviewing -> complete
                  ^                         |
                  +--- (revision needed) ---+
```

Invalid transitions return an error with the valid options. Each transition is logged with timestamp and metadata.

**Impact:** Impossible to skip steps or enter invalid states. Full audit trail of workflow progression. Orchestrators don't need to implement state logic -- MCP handles it.

#### D11. Plan template scaffolding

**Problem:** Planner agent generates plan structure from scratch every time, sometimes forgetting required sections or using inconsistent formatting.

**Solution:** `gsd_scaffold_plan(type, metadata)` generates a plan template:
- `type: "feature"` -> template with UI tasks, API tasks, integration
- `type: "refactor"` -> template with analysis, migration, verification
- `type: "config"` -> minimal template with single task

Planner fills in the template instead of creating from scratch. Template includes all required sections with placeholder comments.

**Impact:** Consistent plan structure. Planner never forgets required sections. Faster plan generation (fill template vs create from scratch).

#### D12. Artifact registry

**Problem:** Agents produce artifacts (research docs, plans, verification reports) scattered across `.planning/` subdirectories. No central index of what exists, what's current, and what's superseded.

**Solution:** `gsd_register_artifact(type, path, metadata)` and `gsd_find_artifacts(type, phase)`:
```json
{
  "type": "research",
  "path": ".planning/phases/05/research/RESEARCH.md",
  "phase": 5,
  "created": "2026-02-16T10:30:00Z",
  "supersedes": null,
  "status": "current"
}
```

Any agent can query "what research exists for phase 5?" and get an instant answer instead of globbing directories.

**Impact:** Agents find artifacts instantly. No orphaned or duplicate documents. Clear lineage of what superseded what.

---

### Better Context

#### D13. Dynamic context budgeting

**Problem:** All agents get roughly the same amount of context regardless of task complexity. A trivial rename task gets the same context dump as a complex multi-file refactor.

**Solution:** `gsd_context_budget(agent_type, task_complexity)` returns a token budget and context priority list:
- **trivial**: 2K tokens -- goal + single plan only
- **simple**: 5K tokens -- goal + plan + key requirements
- **complex**: 15K tokens -- full context with research, conventions, history

Agent prompts use the budget to decide how much context to inject. MCP server truncates/summarizes to fit budget.

**Impact:** Trivial tasks run faster (less context to process). Complex tasks get richer context. Token usage scales with actual need.

#### D14. Codebase-aware planning (file impact analysis)

**Problem:** Planner creates tasks that modify files without understanding the dependency graph. Changing a shared utility might break 15 importers, but the planner doesn't know to add verification for those files.

**Solution:** `gsd_file_impact(file_paths[])` returns:
```json
{
  "shared/utils.ts": {
    "importers": ["pages/Home.tsx", "pages/About.tsx", "components/Header.tsx"],
    "export_count": 12,
    "risk": "high",
    "suggestion": "Add verification for all importers"
  }
}
```

Built from a lightweight import graph (regex-based, cached). Planner uses this to add appropriate verify steps and understand blast radius.

**Impact:** Plans account for ripple effects. Fewer broken imports after execution. Verify steps cover actual dependents, not guesses.

#### D15. Diff-aware re-planning

**Problem:** When execution partially fails and a plan needs revision, the planner re-plans from scratch, potentially undoing completed work or creating conflicts with already-committed code.

**Solution:** `gsd_execution_diff(phase)` returns:
```json
{
  "completed_tasks": ["01-01-task-1", "01-01-task-2"],
  "modified_files": ["src/App.tsx", "src/utils.ts"],
  "uncommitted_changes": ["src/components/New.tsx"],
  "failed_task": "01-02-task-1",
  "failure_reason": "API endpoint changed"
}
```




