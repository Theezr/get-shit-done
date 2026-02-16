---
phase: quick-3
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - get-shit-done/bin/gsd-tools.cjs
  - skills/nick-plan-phase/SKILL.md
  - skills/nick-plan-phase/references/agents/nick-phase-researcher.md
  - skills/nick-plan-phase/references/agents/nick-planner.md
  - agents/nick-planner.md
autonomous: true
has_frontend: false

must_haves:
  truths:
    - "When research_strategy is 'parallel' and phase goal spans multiple domains, SKILL.md spawns multiple researcher agents concurrently with scoped focuses"
    - "Each parallel researcher writes a domain-scoped file like {padded}-RESEARCH-{domain}.md"
    - "Planner reads all RESEARCH*.md files (single or multiple) and synthesizes them"
    - "When research_strategy is 'single' (default), behavior is unchanged from current flow"
    - "gsd-tools init plan-phase returns research_strategy in its JSON output"
  artifacts:
    - path: "get-shit-done/bin/gsd-tools.cjs"
      provides: "research_strategy config field, updated has_research detection"
      contains: "research_strategy"
    - path: "skills/nick-plan-phase/SKILL.md"
      provides: "Parallel research orchestration logic in Section 3"
      contains: "research_strategy"
    - path: "skills/nick-plan-phase/references/agents/nick-phase-researcher.md"
      provides: "Domain-scoped research output"
      contains: "RESEARCH-"
    - path: "skills/nick-plan-phase/references/agents/nick-planner.md"
      provides: "Multi-RESEARCH file consumption"
      contains: "RESEARCH-"
  key_links:
    - from: "skills/nick-plan-phase/SKILL.md"
      to: "gsd-tools.cjs init plan-phase"
      via: "reads research_strategy from init JSON"
      pattern: "research_strategy"
    - from: "skills/nick-plan-phase/SKILL.md"
      to: "nick-phase-researcher.md"
      via: "Task() spawn with domain scope"
      pattern: "domain_focus"
    - from: "skills/nick-plan-phase/references/agents/nick-planner.md"
      to: "phase_dir/*-RESEARCH*.md"
      via: "glob pattern for multiple research files"
      pattern: "RESEARCH"
---

<objective>
Add parallel research capability to nick-plan-phase so that when a phase spans multiple domains (e.g., backend + frontend + config), the orchestrator spawns multiple researcher agents concurrently with scoped focuses instead of one monolithic researcher.

Purpose: Phases that produce 3+ plans across different domains get better research quality -- each researcher dives deep into its domain within a focused context window instead of one researcher trying to cover everything shallowly.

Output: Modified SKILL.md with parallel research orchestration, updated researcher/planner agents, config.json schema update in gsd-tools.cjs.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/execute-plan.md
@~/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@skills/nick-plan-phase/SKILL.md
@skills/nick-plan-phase/references/agents/nick-phase-researcher.md
@skills/nick-plan-phase/references/agents/nick-planner.md
@agents/nick-planner.md
@get-shit-done/bin/gsd-tools.cjs (loadConfig function ~line 158, cmdConfigEnsureSection ~line 572, cmdInitPlanPhase ~line 3909, findPhaseInternal has_research detection ~line 2584)
</context>

<tasks>

<task type="auto">
  <name>Task 1: Add research_strategy to config schema and update has_research detection in gsd-tools.cjs</name>
  <files>get-shit-done/bin/gsd-tools.cjs</files>
  <action>
Three changes in gsd-tools.cjs:

1. **loadConfig (~line 158):** Add `research_strategy` to defaults as `'single'`. In the return object, add:
   ```js
   research_strategy: get('research_strategy', { section: 'workflow', field: 'research_strategy' }) ?? defaults.research_strategy,
   ```

2. **cmdConfigEnsureSection (~line 572):** Add `research_strategy: 'single'` to the `hardcoded.workflow` object so new projects get the field in their config.json.

3. **cmdInitPlanPhase (~line 3909):** Add `research_strategy: config.research_strategy` to the result object, right after `research_enabled`.

4. **has_research detection (~lines 2584, 3743, 4422):** The existing pattern `f.endsWith('-RESEARCH.md')` already matches both `01-RESEARCH.md` AND `01-RESEARCH-backend.md` since both end with `-RESEARCH.md`... Wait, no -- `01-RESEARCH-backend.md` ends with `-backend.md`, not `-RESEARCH.md`. Update all three `hasResearch` checks to:
   ```js
   hasResearch = phaseFiles.some(f => /(-RESEARCH\.md$|-RESEARCH-.+\.md$|^RESEARCH\.md$)/.test(f));
   ```
   This matches: `01-RESEARCH.md` (single mode), `01-RESEARCH-backend.md` (parallel mode), and bare `RESEARCH.md`.

5. **research file include in cmdInitPlanPhase (~line 3968):** Update the `includes.has('research')` block to find ALL research files (not just the first one). Change from `files.find(...)` to `files.filter(...)` and return an array or concatenated content. Specifically:
   ```js
   if (includes.has('research') && phaseInfo?.directory) {
     const phaseDirFull = path.join(cwd, phaseInfo.directory);
     try {
       const files = fs.readdirSync(phaseDirFull);
       const researchFiles = files.filter(f => /(-RESEARCH\.md$|-RESEARCH-.+\.md$|^RESEARCH\.md$)/.test(f)).sort();
       if (researchFiles.length === 1) {
         result.research_content = safeReadFile(path.join(phaseDirFull, researchFiles[0]));
       } else if (researchFiles.length > 1) {
         result.research_files = researchFiles.map(f => ({
           name: f,
           content: safeReadFile(path.join(phaseDirFull, f))
         }));
       }
     } catch {}
   }
   ```

Do NOT change the `loadConfig` defaults structure shape -- keep it flat with nested fallback via `get()`, matching existing pattern.
  </action>
  <verify>
Run `node --test get-shit-done/bin/gsd-tools.test.js` to ensure existing tests pass. Then verify the new field is accessible:
```bash
node -e "
const m = require('./get-shit-done/bin/gsd-tools.cjs');
// Verify loadConfig defaults include research_strategy
"
```
Grep for `research_strategy` in gsd-tools.cjs and confirm it appears in loadConfig defaults, cmdConfigEnsureSection hardcoded.workflow, cmdInitPlanPhase result, and the regex update in has_research checks.
  </verify>
  <done>
- `loadConfig` returns `research_strategy: 'single'` by default, reads from config.json `research_strategy` or `workflow.research_strategy`
- `cmdConfigEnsureSection` includes `research_strategy: 'single'` in workflow defaults
- `cmdInitPlanPhase` includes `research_strategy` in output JSON
- `hasResearch` detection matches both `XX-RESEARCH.md` and `XX-RESEARCH-{domain}.md` patterns
- Research include returns array when multiple research files exist
- All existing tests pass
  </done>
</task>

<task type="auto">
  <name>Task 2: Update SKILL.md orchestration for parallel research and update researcher/planner agents</name>
  <files>
skills/nick-plan-phase/SKILL.md
skills/nick-plan-phase/references/agents/nick-phase-researcher.md
skills/nick-plan-phase/references/agents/nick-planner.md
agents/nick-planner.md
  </files>
  <action>
**A. SKILL.md Section 3 -- Replace the current research section with parallel-aware logic:**

After the existing skip conditions, add research strategy detection:

```
## 3. Handle Research (conditional)

**Skip if:** `--gaps` flag, `--skip-research` flag, `research_enabled=false` (without `--research` override), or RESEARCH.md exists (without `--research` flag).

**Determine strategy:** Read `research_strategy` from init JSON. If `parallel`, identify domain scopes from the phase goal and roadmap description. Typical domain splits:
- backend/api, frontend/ui, config/infra
- data-model, business-logic, integration
- Or any natural 2-4 domain groupings based on the phase goal

**If strategy is `single` (default):** Spawn one researcher as before (unchanged).

**If strategy is `parallel`:** Analyze the phase goal to identify 2-4 distinct domains. For each domain, spawn a researcher in parallel:

\```
# Spawn all researchers concurrently using Task()
# Each gets a scoped focus and writes to a domain-specific file

Task(
  prompt="First, read ~/.claude/skills/nick-plan-phase/references/agents/nick-phase-researcher.md for your role and instructions.

  <objective>Research Phase {phase_number}: {phase_name}
  Answer: 'What do I need to know to PLAN this phase well?'

  DOMAIN FOCUS: {domain_name} -- Focus your research ONLY on {domain_description}. Other domains are being researched by parallel agents. Do NOT research outside your domain scope.</objective>

  <domain_scope>{domain_name}: {domain_description}</domain_scope>

  <files_to_read>
  - .planning/STATE.md
  - .planning/ROADMAP.md (Phase {N} section)
  - .planning/REQUIREMENTS.md (Phase {N} requirements)
  - {phase_dir}/*-CONTEXT.md (if exists)
  </files_to_read>

  <output>Write to: {phase_dir}/{padded_phase}-RESEARCH-{domain_slug}.md</output>",
  subagent_type="general-purpose",
  model="{researcher_model}"
)
\```

Wait for ALL researchers to complete. If any returns `## RESEARCH BLOCKED`, display blocker and offer options. If all return `## RESEARCH COMPLETE`, continue.
```

Keep the existing single-researcher Task() call intact for the `single` strategy path. The parallel path adds new Task() calls alongside it.

Also update the `has_research` check in SKILL.md line 58 to note that existing RESEARCH-*.md files also count:
Change "or RESEARCH.md exists" to "or any RESEARCH*.md files exist".

**B. nick-phase-researcher.md -- Add domain scope awareness:**

In the `<role>` section, add after "Core responsibilities":
```
- When given a `<domain_scope>`, restrict research to that domain only. Write output to the domain-scoped filename provided in `<output>`.
- When NO `<domain_scope>` is provided, research the full phase (default single-researcher behavior).
```

In the `<output_format>` section, update the RESEARCH.md header to include domain when scoped:
```markdown
# Phase [X]: [Name] - Research {- [Domain] (if domain-scoped)}

**Researched:** [date]
**Domain:** [primary technology/problem domain] {OR: scoped domain from <domain_scope>}
**Scope:** {full | domain-name}
**Confidence:** [HIGH/MEDIUM/LOW]
```

In Step 5 (Write RESEARCH.md), add: "If `<domain_scope>` was provided, write to the output path given (e.g., `{padded}-RESEARCH-{domain}.md`). Otherwise write to the standard `{padded}-RESEARCH.md`."

**C. nick-planner.md (both copies: skills/nick-plan-phase/references/agents/ AND agents/):**

In the `gather_phase_context` step, update the research file reading pattern:
```bash
cat "$phase_dir"/*-RESEARCH*.md 2>/dev/null   # Reads single or multiple research files
```

Add a note: "If multiple RESEARCH-{domain}.md files exist (from parallel research), read ALL of them. Synthesize findings across domains -- each file covers a specific domain scope. Treat them as complementary, not competing."

In the `<files_to_read>` references in SKILL.md Section 4 (planner spawn), update:
```
- {phase_dir}/*-RESEARCH*.md (if exists -- may be single or multiple domain files)
```
  </action>
  <verify>
1. Grep all four modified files for `research_strategy` and `RESEARCH-` to confirm the new patterns are present:
   ```bash
   grep -n 'research_strategy\|RESEARCH-\|domain_scope\|domain_focus' skills/nick-plan-phase/SKILL.md skills/nick-plan-phase/references/agents/nick-phase-researcher.md skills/nick-plan-phase/references/agents/nick-planner.md agents/nick-planner.md
   ```

2. Verify SKILL.md still has all 7 sections (1-7) intact and the single-researcher path is preserved.

3. Verify the researcher agent still has all required sections: role, upstream_input, downstream_consumer, philosophy, tool_strategy, mcp_protocol, output_format, execution_flow, structured_returns, success_criteria.

4. Verify the planner agent's gather_phase_context step references `*-RESEARCH*.md` pattern.
  </verify>
  <done>
- SKILL.md Section 3 branches on `research_strategy`: single (unchanged behavior) vs parallel (spawns 2-4 domain-scoped researchers concurrently)
- nick-phase-researcher.md accepts optional `<domain_scope>` and writes to domain-scoped filename when present
- nick-planner.md (both copies) reads `*-RESEARCH*.md` glob pattern and synthesizes multiple research files
- SKILL.md Section 4 planner spawn references `*-RESEARCH*.md` pattern
- Default behavior (`research_strategy: 'single'`) is 100% unchanged from current
  </done>
</task>

</tasks>

<verification>
1. Existing tests pass: `node --test get-shit-done/bin/gsd-tools.test.js`
2. `research_strategy` appears in gsd-tools.cjs loadConfig, cmdConfigEnsureSection, and cmdInitPlanPhase
3. SKILL.md has both single and parallel research paths
4. Researcher agent handles domain_scope parameter
5. Planner agent reads multiple RESEARCH files
6. Default config produces `research_strategy: 'single'` (backward compatible)
7. has_research detection finds both `XX-RESEARCH.md` and `XX-RESEARCH-{domain}.md`
</verification>

<success_criteria>
- A user with `research_strategy: 'single'` (or no setting) gets identical behavior to today
- A user with `research_strategy: 'parallel'` gets N concurrent researcher agents when the orchestrator identifies N domains in the phase goal
- Each parallel researcher writes to `{padded}-RESEARCH-{domain}.md`
- The planner reads and synthesizes all research files regardless of strategy
- gsd-tools.cjs `init plan-phase` returns the strategy so the orchestrator can branch
</success_criteria>

<output>
After completion, create `.planning/quick/3-parallel-research-for-multi-plan-phases-/3-SUMMARY.md`
</output>
