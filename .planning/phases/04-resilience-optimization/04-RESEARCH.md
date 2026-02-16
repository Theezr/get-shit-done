# Phase 4: Resilience & Optimization - Research

**Researched:** 2026-02-16
**Domain:** MCP graceful degradation, token optimization via research caching and conditional prompt loading, gsd-tools.js phase-specific extraction
**Confidence:** HIGH

## Summary

Phase 4 is a hardening phase -- no new skills or agents are created. The four existing skills (plan-phase, execute-phase, verify-work, review) and their agent definitions already work. This phase makes them more resilient (MCP unavailability handling) and more efficient (fewer wasted tokens).

The work decomposes into four distinct workstreams aligned with the four requirements (MCP-08, TOKEN-03, TOKEN-04, TOKEN-05). Each workstream modifies existing files rather than creating new ones. The verify-work skill already demonstrates the correct pre-flight check pattern for Chrome DevTools MCP (`list_pages` call, fallback to "inconclusive" status). This pattern needs to be generalized across all MCP-using agents. The executor already reads RESEARCH.md before Context7 queries (TOKEN-03 partially implemented). The gsd-tools.js already has `roadmap get-phase` but lacks `requirements get-phase` and `--format` options (TOKEN-04). The executor agent definition is ~17KB with ~34% in conditional sections (TDD, checkpoints, auth gates) that are loaded even when the plan does not require them (TOKEN-05).

**Primary recommendation:** Modify 7 existing files across the four workstreams. No new files needed. The MCP degradation workstream is the most impactful (touches all 4 skill agent defs). The conditional prompt loading workstream (TOKEN-05) is the most architecturally significant -- it requires splitting agent definitions into core + conditional reference files and modifying SKILL.md orchestrators to conditionally include them based on plan frontmatter metadata.

## Standard Stack

### Core

| Component | Location | Purpose | Why Standard |
|-----------|----------|---------|--------------|
| gsd-tools.js | `~/.claude/get-shit-done/bin/gsd-tools.js` | CLI utility -- add `requirements get-phase`, extend `roadmap get-phase` | Existing tool, all agents already depend on it |
| SKILL.md orchestrators | `~/.claude/skills/gsd-*/SKILL.md` | Modify Task prompts to conditionally include reference files | Established orchestrator pattern from Phase 1 |
| Agent definitions | `~/.claude/skills/gsd-*/references/agents/*.md` | Add MCP degradation sections, split into core + conditional files | Established agent definition pattern from Phase 1-2 |
| Plan frontmatter | `type`, `autonomous`, `tdd` fields | Drive conditional loading decisions in orchestrator | Already parsed by `phase-plan-index` command |

### Supporting

| Component | Location | Purpose | When to Use |
|-----------|----------|---------|-------------|
| WebSearch/WebFetch | Built-in tools | Fallback when Context7/microsoft-docs MCPs unavailable | MCP degradation fallback path |
| RESEARCH.md | `{phase_dir}/*-RESEARCH.md` | Cache research findings for downstream agents | Already exists, needs consistency enforcement |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Splitting agent .md into multiple files | Loading full agent .md always | Current approach wastes ~1,450 tokens (executor) on unused sections; splitting saves ~34% per invocation for non-checkpoint non-TDD plans |
| Conditional `@`-references in Task prompts | Runtime config in gsd-tools.js | `@`-references are the existing pattern; config would add a new mechanism |
| New `requirements` top-level command | Extending `roadmap` command | Requirements and roadmap are separate concepts; own command is cleaner |
| Pre-flight MCP check in each agent | Pre-flight check in SKILL.md orchestrator only | Agent-level is more resilient -- orchestrator might not know which MCPs the agent will use |

## Architecture Patterns

### Pattern 1: Pre-flight MCP Availability Check

**What:** Before using any MCP tool, attempt a lightweight "ping" operation. If it fails, set a flag and use fallback path for the rest of the session.

**When to use:** Every agent that uses MCP tools (researcher, executor, browser-tester, code-reviewer).

**Existing implementation (browser-tester agent, verify-work SKILL.md):**
```markdown
## Step 3: Pre-flight Check

Attempt `mcp__chrome-devtools__list_pages` to verify Chrome DevTools MCP is connected.

- **If unavailable:** Report "Chrome DevTools MCP not connected -- runtime verification skipped." Set status to "inconclusive".
- **If available:** Proceed to spawn browser tester.
```

**Generalized pattern for all MCP-using agents:**
```markdown
## MCP Pre-flight

### Context7
Attempt: `mcp__context7__resolve-library-id` with a known library (e.g., "react")
- Available: Use Context7 for lookups. Confidence = HIGH.
- Unavailable: Fall back to WebSearch + WebFetch. Confidence = MEDIUM max.

### microsoft-docs
Attempt: `mcp__microsoft-docs__microsoft_docs_search` with a known query (e.g., "Azure App Service")
- Available: Use microsoft-docs for Azure lookups.
- Unavailable: Fall back to WebSearch + WebFetch.

### IDE Diagnostics
Attempt: `mcp__ide__getDiagnostics`
- Available: Use for type checking. Cross-verify with `npm run typecheck`.
- Unavailable: Rely solely on `npm run typecheck`.
```

**Key insight:** The pre-flight check should happen once at the start of the agent's session, not before every individual MCP call. Store the result as a boolean flag and reference it throughout execution.

### Pattern 2: Confidence Tagging Based on Source

**What:** Tag all research findings and implementation decisions with a confidence level based on the source.

**When to use:** All agents that produce or consume external information.

**Existing implementation (Phase 2 researcher `<mcp_protocol>`):**
```markdown
| Source | Confidence | Tag in RESEARCH.md |
|--------|-----------|---------------------|
| Context7 verified | HIGH | "Verified via Context7" |
| Official docs via WebFetch | MEDIUM | "Verified via official docs" |
| WebSearch cross-verified | MEDIUM | "WebSearch verified with [source]" |
| Training data only | LOW | "Training data -- needs validation" |
```

**Extension for degradation scenarios:**
```markdown
| Scenario | Confidence | Tag |
|----------|-----------|-----|
| MCP available, result found | HIGH | "MCP-verified" |
| MCP unavailable, WebSearch fallback with official source | MEDIUM | "WebSearch fallback (MCP unavailable)" |
| MCP unavailable, WebSearch only | LOW | "WebSearch only (MCP unavailable, needs validation)" |
| MCP unavailable, training data only | LOW | "Training data only (MCP unavailable)" |
```

### Pattern 3: Conditional Reference Loading in Orchestrator

**What:** The SKILL.md orchestrator reads plan frontmatter and only includes conditional reference files in the Task prompt's `<execution_context>` based on what the plan actually needs.

**When to use:** Execute-phase orchestrator spawning executor agents.

**Current state (always loads everything):**
```markdown
<execution_context>
Read these workflow files for execution instructions:
@~/.claude/get-shit-done/workflows/execute-plan.md
@~/.claude/get-shit-done/templates/summary.md
@~/.claude/get-shit-done/references/checkpoints.md
@~/.claude/get-shit-done/references/tdd.md
</execution_context>
```

**Target state (conditional loading):**
```markdown
<execution_context>
Read these workflow files for execution instructions:
@~/.claude/get-shit-done/workflows/execute-plan.md
@~/.claude/get-shit-done/templates/summary.md
{IF plan.autonomous === false:}
@~/.claude/get-shit-done/references/checkpoints.md
{IF plan.type === "tdd":}
@~/.claude/get-shit-done/references/tdd.md
</execution_context>
```

**Implementation approach:** The execute-phase SKILL.md orchestrator already reads plan frontmatter via `phase-plan-index`. It can check `autonomous` and `type` fields from the plan index to build the Task prompt dynamically. This is NOT a runtime config change -- it is the orchestrator constructing the correct prompt for each specific plan.

### Pattern 4: Agent Definition Split (Core + Conditional)

**What:** Split the executor agent definition into a core file (always loaded) and conditional files (loaded only when needed).

**When to use:** Agent definitions with large conditional sections (executor is the primary candidate at ~34% conditional content).

**Structure:**
```
references/agents/
  gsd-executor.md              # Core: role, mcp_protocol, best_practice_skills,
                                # execution_flow, task_commit_protocol, summary_creation,
                                # self_check, state_updates, final_commit, completion_format
  gsd-executor-checkpoints.md  # Conditional: checkpoint_protocol, checkpoint_return_format,
                                # continuation_handling, authentication_gates
  gsd-executor-tdd.md          # Conditional: tdd_execution
```

**Token savings estimate:**
- Current: ~4,300 tokens loaded every time
- With split: ~2,850 tokens (core) + ~1,000 tokens (checkpoints, when needed) + ~175 tokens (TDD, when needed)
- Average savings per non-checkpoint non-TDD plan: ~1,450 tokens (~34%)

### Anti-Patterns to Avoid

- **Over-engineering the split:** Only the executor agent has enough conditional content to justify splitting. The planner's conditional sections (TDD, checkpoints, gap closure) are needed at planning time because the planner decides what to create, not what to execute. Do NOT split the planner.
- **Splitting too fine-grained:** Two conditional files (checkpoints + TDD) is enough. Do not create one file per section.
- **Pre-flight in wrong place:** Do NOT put MCP pre-flight checks in gsd-tools.js. The tools are called by agents, not by the Node.js CLI.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Phase-specific requirement extraction | Custom grep/sed scripts in agents | `gsd-tools.js requirements get-phase N` | Agents already depend on gsd-tools; consistent pattern |
| MCP availability detection | Complex try/catch wrappers | Simple pre-flight call + boolean flag | MCP tools either work or throw; no partial states |
| Conditional prompt construction | String template engine | Orchestrator if/else on plan frontmatter | Orchestrator already reads frontmatter; keep it simple |

**Key insight:** This phase modifies existing files and patterns. The temptation is to build abstractions (MCP wrapper layer, config-driven prompt builder). Resist this. The existing patterns (pre-flight check, @-references, gsd-tools commands) are the right level of abstraction.

## Common Pitfalls

### Pitfall 1: Breaking the Agent Contract

**What goes wrong:** Splitting agent definitions or changing MCP handling breaks the implicit contract between orchestrator and agent.
**Why it happens:** Agent definitions are prompts. Removing sections (even conditional ones) changes agent behavior.
**How to avoid:** Always test the full pipeline after changes. Verify that the executor still handles checkpoints when the checkpoint file IS loaded, and that it does not crash when it is NOT loaded.
**Warning signs:** Agent produces incomplete SUMMARY.md, misses checkpoint handling, fails TDD cycle.

### Pitfall 2: Inconsistent RESEARCH.md Caching

**What goes wrong:** Some agents check RESEARCH.md before MCP queries, others do not. Result: redundant MCP calls consuming tokens.
**Why it happens:** TOKEN-03 was partially implemented in Phase 2 (executor reads RESEARCH.md first) but the pattern may not be consistent across all agents.
**How to avoid:** Audit every agent that calls Context7 or microsoft-docs. Ensure all have the "check RESEARCH.md first, skip MCP if covered" pattern.
**Warning signs:** Researcher documents a library in RESEARCH.md, then executor queries Context7 for the same library.

### Pitfall 3: gsd-tools.js Parsing Fragility

**What goes wrong:** New `requirements get-phase` command parses REQUIREMENTS.md incorrectly for edge cases (decimal phases, multi-line requirement descriptions).
**Why it happens:** Markdown parsing with regex is fragile. The existing `roadmap get-phase` works because ROADMAP.md has a consistent format, but REQUIREMENTS.md has a different structure.
**How to avoid:** Use the existing Traceability table in REQUIREMENTS.md as the source of truth for phase-to-requirement mapping. Parse the table, not the prose.
**Warning signs:** Command returns wrong requirements for a phase, or misses requirements that span multiple lines.

### Pitfall 4: Pre-flight Check False Negatives

**What goes wrong:** MCP pre-flight check succeeds but later calls fail (transient network issue, rate limit).
**Why it happens:** Pre-flight tests connectivity at one point in time, not continuously.
**How to avoid:** Even with pre-flight success, wrap individual MCP calls in try/catch. If an MCP call fails mid-session, downgrade confidence and fall back gracefully.
**Warning signs:** Agent assumes MCP is available based on pre-flight, then gets error mid-execution and crashes.

### Pitfall 5: Conditional Loading Breaks Continuation

**What goes wrong:** Continuation agent (spawned after checkpoint) does not get the checkpoint reference file because the orchestrator re-evaluates conditions.
**Why it happens:** Checkpoint handling occurs AFTER the agent has been spawned. If the continuation agent is spawned without checkpoint references, it cannot handle the checkpoint protocol.
**How to avoid:** The continuation agent MUST always get the checkpoint reference file. Conditional loading applies to the initial spawn only based on plan frontmatter. Continuation spawns always include checkpoint references.
**Warning signs:** Continuation agent fails to return structured checkpoint message, or does not understand the completed tasks table format.

## Code Examples

### Example 1: Pre-flight MCP Check for Researcher Agent

```markdown
<mcp_degradation>

## Pre-flight MCP Availability

At the start of your research session, test each MCP you plan to use:

### Context7
```
mcp__context7__resolve-library-id with libraryName="react" query="react library"
```
- **Success:** Set `context7_available = true`. Use Context7 for all library lookups.
- **Failure:** Set `context7_available = false`. Fall back to WebSearch + WebFetch for all library lookups. Tag findings as MEDIUM confidence max.

### microsoft-docs (if phase involves Azure)
```
mcp__microsoft-docs__microsoft_docs_search with query="Azure App Service"
```
- **Success:** Set `microsoft_docs_available = true`. Use for Azure documentation.
- **Failure:** Set `microsoft_docs_available = false`. Fall back to WebSearch.

### Confidence Tagging with Degradation

| context7_available | Source Used | Confidence |
|--------------------|------------|------------|
| true | Context7 result found | HIGH |
| true | Context7 no result, WebFetch official docs | MEDIUM |
| false | WebSearch + official source | MEDIUM |
| false | WebSearch only | LOW |
| false | Training data only | LOW |

</mcp_degradation>
```

### Example 2: requirements get-phase Command

```javascript
// In gsd-tools.js -- new subcommand under 'requirements'
function cmdRequirementsGetPhase(cwd, phaseNum, raw) {
  const reqPath = path.join(cwd, '.planning', 'REQUIREMENTS.md');
  if (!fs.existsSync(reqPath)) {
    output({ found: false, error: 'REQUIREMENTS.md not found' }, raw, '');
    return;
  }

  const content = fs.readFileSync(reqPath, 'utf-8');

  // Parse the Traceability table for phase mapping
  const tablePattern = /\|\s*([\w-]+)\s*\|\s*Phase\s+(\d+(?:\.\d+)?)\s*\|\s*(\w+)\s*\|/g;
  const requirements = [];
  let match;
  while ((match = tablePattern.exec(content)) !== null) {
    if (match[2] === String(phaseNum)) {
      requirements.push({
        id: match[1].trim(),
        phase: match[2],
        status: match[3].trim(),
      });
    }
  }

  // For each requirement ID found, extract its description from the v1 Requirements section
  const reqDetails = requirements.map(req => {
    const descPattern = new RegExp(
      `\\*\\*${req.id}\\*\\*:?\\s*(.+)`,
      'i'
    );
    const descMatch = content.match(descPattern);
    return {
      ...req,
      description: descMatch ? descMatch[1].trim() : null,
    };
  });

  output({
    found: reqDetails.length > 0,
    phase: phaseNum,
    count: reqDetails.length,
    requirements: reqDetails,
  }, raw);
}
```

### Example 3: Conditional Reference Loading in Execute-Phase Orchestrator

```markdown
## Step 4b: Spawn executor agents

For each plan, read its frontmatter to determine what reference files to include:

**Build execution_context dynamically:**

```
// Core references (always included)
execution_context_refs = [
  "@~/.claude/get-shit-done/workflows/execute-plan.md",
  "@~/.claude/get-shit-done/templates/summary.md"
]

// Conditional: checkpoint references
if plan.autonomous === false:
  execution_context_refs.append("@~/.claude/get-shit-done/references/checkpoints.md")

// Conditional: TDD references
if plan.type === "tdd":
  execution_context_refs.append("@~/.claude/get-shit-done/references/tdd.md")
```

Include in Task prompt:
```
<execution_context>
Read these workflow files for execution instructions:
{execution_context_refs joined with newlines}
</execution_context>
```
```

### Example 4: Executor Agent Definition Split

**Core file (gsd-executor.md) -- always loaded:**
```markdown
---
name: gsd-executor
---

<role>...</role>
<mcp_protocol>...</mcp_protocol>
<mcp_degradation>...</mcp_degradation>  <!-- NEW: Phase 4 addition -->
<best_practice_skills>...</best_practice_skills>
<execution_flow>...</execution_flow>
<deviation_rules>...</deviation_rules>
<task_commit_protocol>...</task_commit_protocol>
<summary_creation>...</summary_creation>
<self_check>...</self_check>
<state_updates>...</state_updates>
<final_commit>...</final_commit>
<completion_format>...</completion_format>
<success_criteria>...</success_criteria>
```

**Checkpoint file (gsd-executor-checkpoints.md) -- loaded when autonomous=false:**
```markdown
<checkpoint_protocol>...</checkpoint_protocol>
<checkpoint_return_format>...</checkpoint_return_format>
<continuation_handling>...</continuation_handling>
<authentication_gates>...</authentication_gates>
```

**TDD file (gsd-executor-tdd.md) -- loaded when type=tdd:**
```markdown
<tdd_execution>...</tdd_execution>
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Agents load full definition always | Conditional section loading based on plan metadata | Phase 4 (now) | ~34% token savings per executor invocation for standard plans |
| No MCP fallback handling | Graceful degradation with confidence tagging | Phase 4 (now) | Skills work without MCP servers; confidence tracks data quality |
| Agents re-query MCPs for same info | RESEARCH.md caching, agents read file first | Phase 2 (partial) -> Phase 4 (complete) | Eliminates redundant MCP calls across agent pipeline |
| Full file reads (ROADMAP.md, REQUIREMENTS.md) | Phase-specific extraction via gsd-tools.js | Phase 4 (now) | Agents get only what they need, not entire files |

## Files to Modify

### MCP-08: Graceful MCP Degradation

| File | Change | Lines Added (est.) |
|------|--------|-------------------|
| `skills/gsd-plan-phase/references/agents/gsd-phase-researcher.md` | Add `<mcp_degradation>` section with pre-flight checks and fallback paths | ~40 |
| `skills/gsd-execute-phase/references/agents/gsd-executor.md` | Add `<mcp_degradation>` section for Context7 fallback | ~30 |
| `skills/gsd-verify-work/references/agents/gsd-browser-tester.md` | Already has pre-flight; add confidence tagging to report | ~10 |
| `skills/gsd-review/references/agents/gsd-code-reviewer.md` | Add IDE diagnostics fallback (already uses `npm run typecheck` as backup) | ~15 |
| `~/.claude/agents/gsd-executor.md` | Mirror changes from references copy | ~30 |
| `~/.claude/agents/gsd-phase-researcher.md` | Mirror changes from references copy | ~40 |
| `~/.claude/agents/gsd-verifier.md` | Mirror changes from references copy | ~10 |

### TOKEN-03: Research Caching Consistency

| File | Change | Lines Added (est.) |
|------|--------|-------------------|
| `skills/gsd-plan-phase/references/agents/gsd-planner.md` | Add "Read RESEARCH.md before Context7" protocol (matches executor pattern) | ~15 |
| `skills/gsd-plan-phase/references/agents/gsd-plan-checker.md` | Add "Read RESEARCH.md before Context7" if checker uses MCP | ~10 |
| `~/.claude/agents/gsd-planner.md` | Mirror changes | ~15 |

### TOKEN-04: Phase-Specific Extraction

| File | Change | Lines Added (est.) |
|------|--------|-------------------|
| `~/.claude/get-shit-done/bin/gsd-tools.js` | Add `requirements get-phase N`, extend `roadmap get-phase` with `--format section` | ~80 |

### TOKEN-05: Conditional Agent Prompt Sections

| File | Change | Lines Added (est.) |
|------|--------|-------------------|
| `skills/gsd-execute-phase/SKILL.md` | Modify Task prompt to conditionally include checkpoint/TDD refs based on plan frontmatter | ~15 |
| `skills/gsd-execute-phase/references/agents/gsd-executor.md` | Remove checkpoint_protocol, checkpoint_return_format, continuation_handling, authentication_gates, tdd_execution sections | -120 (moved) |
| `skills/gsd-execute-phase/references/agents/gsd-executor-checkpoints.md` | NEW: checkpoint + auth gate sections extracted from executor | ~80 |
| `skills/gsd-execute-phase/references/agents/gsd-executor-tdd.md` | NEW: TDD execution section extracted from executor | ~20 |

## Open Questions

1. **Should the planner agent also be split?**
   - What we know: The planner has ~25% conditional content (TDD integration, checkpoints, gap closure, revision mode). However, the planner decides what goes into plans -- it needs to know about ALL plan types to make those decisions.
   - What's unclear: Whether gap_closure_mode and revision_mode could be separate files loaded only when `--gaps` or revision mode is active.
   - Recommendation: Do NOT split the planner in Phase 4. The planner needs all information to make planning decisions. Token savings (~2,300 tokens) would come at the cost of the planner not knowing about plan types it should consider. Revisit if planner grows significantly.

2. **Should the global agent definitions at `~/.claude/agents/` be kept in sync with references/ copies?**
   - What we know: Two copies exist (e.g., `~/.claude/agents/gsd-executor.md` and `skills/gsd-execute-phase/references/agents/gsd-executor.md`). The references/ copies are "latest" (have MCP protocol from Phase 2). The global copies are older.
   - What's unclear: Whether both are actually used. Skills use the references/ copies. What uses the global copies?
   - Recommendation: Update both copies to stay in sync. The global copies at `~/.claude/agents/` may be used by standalone commands or other workflows. Do not remove them.

3. **How should `phase-plan-index` expose plan type for conditional loading?**
   - What we know: `phase-plan-index` already extracts `autonomous` from frontmatter but not `type` (execute vs tdd).
   - What's unclear: Whether the orchestrator should use `phase-plan-index` or read frontmatter directly.
   - Recommendation: Add `type` to the `phase-plan-index` output. The orchestrator already uses this command; adding `type` keeps the pattern consistent. Also add `has_tdd` (boolean) to the `tdd` attribute in frontmatter.

## Sources

### Primary (HIGH confidence)

- **Codebase inspection** -- All findings verified against actual files in the repository:
  - `~/.claude/skills/gsd-*/SKILL.md` (4 skills)
  - `~/.claude/skills/gsd-*/references/agents/*.md` (5 agent definitions)
  - `~/.claude/agents/gsd-*.md` (11 global agent definitions)
  - `~/.claude/get-shit-done/bin/gsd-tools.js` (~4,400 lines)
  - `.planning/REQUIREMENTS.md`, `.planning/ROADMAP.md`
  - `.planning/phases/02-*/02-RESEARCH.md`, `.planning/phases/03-*/03-RESEARCH.md`

### Secondary (MEDIUM confidence)

- **Phase 2 and Phase 3 research documents** -- Prior research validated the MCP integration patterns and confidence tagging approach now being hardened.

### Tertiary (LOW confidence)

- None. All findings are based on direct codebase inspection.

## Metadata

**Confidence breakdown:**
- MCP degradation pattern: HIGH -- verified against existing browser-tester pre-flight check and Phase 2 researcher fallback patterns
- Research caching (TOKEN-03): HIGH -- verified executor already has RESEARCH.md-first pattern; planner does not
- Phase-specific extraction (TOKEN-04): HIGH -- verified gsd-tools.js has no `requirements` command; `roadmap get-phase` exists but lacks `--format`
- Conditional loading (TOKEN-05): HIGH -- verified executor has ~34% conditional content; plan frontmatter already has `type` and `autonomous` fields; orchestrator already reads frontmatter

**Research date:** 2026-02-16
**Valid until:** 2026-03-16 (stable -- this is internal tooling, not external dependencies)
