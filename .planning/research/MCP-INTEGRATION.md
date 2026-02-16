# MCP Integration Research

**Date:** 2026-02-15
**Scope:** Context7, microsoft-docs, Chrome DevTools, shadcn, Figma, IDE diagnostics
**Goal:** Mandatory MCP-based documentation lookups in planning/execution; Chrome DevTools for runtime verification

---

## Key Findings

- **Nick's skills already demonstrate the gold standard.** `nick:plan`, `nick:execute`, and `nick:verify` show the exact patterns needed: explicit `allowed-tools` lists with individual MCP tool names, mandatory lookup rules in prose, and clear fallback guidance. The GSD framework's existing agents only partially adopt this (3 of 11 agents have `mcp__context7__*`; none have microsoft-docs, shadcn, Chrome DevTools, or Figma).

- **Two tool declaration styles exist: wildcard vs. explicit.** GSD agents use `mcp__context7__*` (wildcard), while Nick's skills list every MCP tool individually (e.g., `mcp__context7__resolve-library-id`, `mcp__context7__query-docs`). The explicit style is superior for skills because it documents exactly what the skill needs, prevents accidental use of unneeded tools, and matches Anthropic's skill guide recommendations.

- **Context7's two-step pattern is non-negotiable.** Every Context7 usage requires `resolve-library-id` then `query-docs`. You cannot skip the resolve step. The system prompt for the tool itself states: "You MUST call this function before 'query-docs' to obtain a valid Context7-compatible library ID." There is also a 3-call limit per question to prevent runaway lookups.

- **Chrome DevTools MCP is the most tool-rich integration.** It provides 25+ tools spanning navigation, interaction, inspection, and performance. The verify skill correctly scopes these into a pipeline: snapshot -> interact -> verify. The test-flow skill demonstrates the delegation pattern: orchestrator -> test-planner agent (Sonnet) -> browser-tester agent (Haiku).

- **MCP unavailability must be handled gracefully.** MCPs can be disconnected, misconfigured, or rate-limited. The GSD constraint from PROJECT.md states: "Skills must gracefully handle MCPs being unavailable (fallback to training data)." The phase-researcher agent already models this with its confidence hierarchy (Context7 HIGH > Official docs MEDIUM > WebSearch needs verification).

- **Token cost of MCP lookups is modest but adds up.** A Context7 resolve + query pair costs approximately 200-500 tokens of input (tool call overhead) plus 500-2000 tokens of output (documentation returned). For a planning session touching 5 libraries, that is 3,500-12,500 tokens. Compare to the 15,000-45,000 tokens wasted by GSD's content-passing pattern -- MCP lookups are far more efficient than redundant context passing.

- **Multi-MCP coordination follows the skill guide's Pattern 2.** The official Claude Code Skill Guide (Chapter 5) documents "Multi-MCP coordination" with clear phase separation, data passing between MCPs, validation before moving to next phase, and centralized error handling.

---

## Context7 Integration Patterns

### The Mandatory Two-Step

Every Context7 lookup follows this exact flow:

```
1. mcp__context7__resolve-library-id
   Input: { libraryName: "next.js", query: "server components data fetching" }
   Output: libraryId (e.g., "/vercel/next.js")

2. mcp__context7__query-docs
   Input: { libraryId: "/vercel/next.js", query: "server components data fetching patterns" }
   Output: Documentation snippets with code examples
```

### When to Trigger Lookups

**Planning phase (mandatory):**
- For EVERY library or framework the implementation will touch (nick:plan, lines 59-65)
- For EVERY Azure service mentioned (microsoft-docs MCP)
- For EVERY shadcn component being planned (shadcn MCP)

**Execution phase (mandatory, verification purpose):**
- Before implementing any API call -- verify the plan's references are still current (nick:execute, lines 45-48)
- If the plan references an outdated API, adapt and note the deviation

**Research phase (mandatory, investigation purpose):**
- Context7 is the FIRST source, before WebSearch or WebFetch (gsd-phase-researcher, tool priority table)
- Cross-verify WebSearch findings with Context7 when possible

### Token Cost Management

**Problem:** Naive lookups for every library in a large project could burn 10,000+ tokens on documentation that may not be needed.

**Strategies observed in existing skills:**

1. **Scoped queries:** Nick's skills say "look up the specific APIs you'll reference in the plan" -- not "look up everything about React." Query specificity reduces returned tokens dramatically.

2. **Conditional lookups:** The execute skill only re-verifies APIs that the plan actually references. It does not re-research the entire stack.

3. **Progressive escalation:** The discovery-phase workflow uses three depth levels:
   - Level 1 (Quick Verify): Single Context7 call to confirm syntax/version -- 2-5 minutes
   - Level 2 (Standard): Context7 for each option + WebSearch -- 15-30 minutes
   - Level 3 (Deep Dive): Exhaustive Context7 + official docs + WebSearch -- 1+ hour

4. **3-call limit:** Context7's own documentation warns: "Do not call this tool more than 3 times per question." This is a built-in guard against token waste.

**Recommended pattern for GSD integration:**

```
IF planning:
  - resolve + query for each library mentioned in phase goals (max 5-6 pairs)
  - Store findings in RESEARCH.md or plan's "Research Findings" section
  - Downstream agents read the file, not re-query

IF executing:
  - resolve + query ONLY for APIs being implemented in the current step
  - Skip if plan already includes verified API signatures with versions

IF researching:
  - Full Context7 exploration, multiple queries per library
  - Document everything in RESEARCH.md with source attribution
```

### Handling Context7 Unavailability

```
IF mcp__context7__resolve-library-id fails:
  1. Log: "Context7 unavailable, falling back to training data"
  2. Mark confidence as LOW for any library references
  3. Use WebSearch + WebFetch for official docs as fallback
  4. Flag in output: "API references NOT verified via Context7"

IF resolve succeeds but query-docs fails:
  1. Retry once with a simpler query
  2. If still fails, use the libraryId to construct docs URL and WebFetch it
  3. Mark confidence as MEDIUM
```

---

## Chrome DevTools Verification Pipeline

### Architecture

The verify skill (`nick:verify`) demonstrates a three-tier testing architecture:

```
Tier 1: Automated Workflow Tests
  Orchestrator (nick:verify)
    -> test-planner agent (Sonnet) -- reads workflow, produces test plan
    -> browser-tester agent (Haiku) -- executes tests via Chrome DevTools MCP

Tier 2: Chrome DevTools Inspection
  Direct MCP tool usage by the verify skill itself:
    list_pages -> navigate_page -> take_snapshot ->
    list_console_messages -> list_network_requests ->
    performance_start_trace -> performance_analyze_insight

Tier 3: Visual Verification
  User manually inspects the running app
```

### The Snapshot-Interact-Verify Pattern

Every Chrome DevTools test step follows this protocol:

```
1. SNAPSHOT: take_snapshot (or take_screenshot for visual comparison)
   - Get the current page state (a11y tree with UIDs)
   - This is the "before" state

2. INTERACT: click / fill / hover / press_key / navigate_page
   - Use UIDs from the snapshot to target elements
   - Always snapshot BEFORE interacting (never interact blind)

3. VERIFY: take_snapshot + assertions
   - Take a new snapshot after interaction
   - Assert expected elements exist, text matches, states changed
   - Check console for errors: list_console_messages(types: ["error", "warn"])
   - Check network: list_network_requests for expected API calls
```

### Full Verification Pipeline

```
Step 1: Find the app
  list_pages -> find running app
  select_page -> focus the correct tab

Step 2: Navigate to feature
  navigate_page(url: "http://localhost:3000/feature-path")
  wait_for(text: "Expected heading")

Step 3: Structural verification
  take_snapshot -> verify page structure
  - Expected elements present
  - Correct labels, placeholders, default values
  - Accessibility attributes (from a11y tree)

Step 4: Interaction testing
  For each user flow:
    take_snapshot -> get UIDs
    click/fill/hover -> interact with elements
    take_snapshot -> verify state change
    list_console_messages(types: ["error"]) -> zero errors

Step 5: Network verification
  list_network_requests -> verify API calls
  get_network_request(reqid) -> check response bodies
  - Correct endpoints called
  - Expected status codes (200, 201, etc.)
  - No failed requests (4xx, 5xx)

Step 6: Performance verification
  performance_start_trace(reload: true, autoStop: true)
  performance_analyze_insight -> check for:
  - No long tasks > 50ms
  - Acceptable LCP, FID, CLS scores
  - No layout thrashing

Step 7: Report
  Aggregate all results into structured report
```

### Tool Selection for Chrome DevTools

The verify skill lists 20 Chrome DevTools tools explicitly. These fall into categories:

| Category | Tools | Purpose |
|----------|-------|---------|
| Navigation | `navigate_page`, `list_pages`, `select_page` | Find and navigate to features |
| Inspection | `take_snapshot`, `take_screenshot` | Capture page state |
| Interaction | `click`, `fill`, `hover`, `press_key` | Simulate user actions |
| Console | `list_console_messages`, `get_console_message` | Check for errors |
| Network | `list_network_requests`, `get_network_request` | Verify API calls |
| Performance | `performance_start_trace`, `performance_stop_trace`, `performance_analyze_insight` | Performance profiling |
| Scripting | `evaluate_script` | Custom assertions |
| Environment | `emulate`, `wait_for` | Device emulation, timing |

### Agent Delegation for Testing

The test-flow skill demonstrates an effective delegation pattern:

- **Orchestrator** (main skill): Reads workflow, dispatches agents, aggregates results
- **test-planner** (Sonnet model): Reads workflow guide content, produces numbered test plan with snapshot-interact-verify steps. Higher model used because test plan quality matters.
- **browser-tester** (Haiku model): Executes test plan mechanically, reports pass/fail with evidence. Lower model used because this is rote execution with clear instructions.

This is cost-efficient: expensive reasoning for plan creation, cheap execution for repetitive browser actions.

### Chrome DevTools Unavailability

```
IF Chrome DevTools MCP not connected:
  1. Report: "Chrome DevTools MCP not available"
  2. Skip automated browser testing
  3. Offer alternatives:
     a. Visual verification (user checks manually)
     b. Run test suite only (npm run test)
  4. Mark runtime verification as "inconclusive" (not "failed")
  Environment issues = inconclusive, not failure (nick:verify rules)

IF app not running:
  1. Report: "App not running at expected URL"
  2. Suggest: "Start the app with `npm run dev` and re-run verification"
  3. Mark as inconclusive
```

---

## Microsoft-Docs Integration

### When to Use

The microsoft-docs MCP is triggered by Azure service involvement:

- **nick:plan** (line 68): "If the feature involves ANY Azure service"
- **nick:execute** (line 53): "If the plan involves ANY Azure service"

### Three-Tool Pattern

```
1. microsoft_docs_search(query: "Azure Blob Storage authentication")
   -> Returns up to 10 content chunks (500 tokens each)
   -> Provides article titles, URLs, excerpts

2. microsoft_code_sample_search(query: "Azure Blob Storage upload", language: "typescript")
   -> Returns code snippets from official docs
   -> Filter by language for relevant examples

3. microsoft_docs_fetch(url: "https://learn.microsoft.com/en-us/azure/...")
   -> Full page content in markdown
   -> Use AFTER search identifies high-value pages
   -> Required step per tool documentation for comprehensive results
```

### Integration Pattern

The microsoft-docs MCP follows a search-then-fetch pattern similar to Context7's resolve-then-query. The key difference: microsoft-docs has three tools instead of two, and the `fetch` step is for deep-reading specific pages identified by search.

**Token cost:** Search returns chunked excerpts (efficient). Fetch returns full pages (expensive). Only fetch pages that are highly relevant.

### When microsoft-docs vs. Context7

| Scenario | Use |
|----------|-----|
| Azure SDK API reference | microsoft-docs (authoritative for Azure) |
| Azure service configuration | microsoft-docs |
| React/Next.js/NestJS APIs | Context7 |
| Library used WITH Azure (e.g., @azure/storage-blob) | Both -- microsoft-docs for Azure side, Context7 for SDK patterns |
| General TypeScript/JavaScript | Context7 |

---

## Multi-MCP Coordination

### Pattern from Skill Guide (Chapter 5, Pattern 2)

The Claude Code Skill Guide documents multi-MCP coordination as:

```markdown
# Phase 1: Design Export (Figma MCP)
1. Export design assets
2. Generate specifications

# Phase 2: Asset Storage (Drive MCP)
1. Create project folder
2. Upload assets

# Phase 3: Task Creation (Linear MCP)
1. Create tasks with asset links

# Phase 4: Notification (Slack MCP)
1. Post summary
```

Key techniques: clear phase separation, data passing between MCPs, validation before next phase, centralized error handling.

### Observed Multi-MCP Patterns in Nick's Skills

**nick:plan** coordinates 3 MCP servers:
1. Context7 (library docs) -- for every library
2. microsoft-docs (Azure) -- conditional on Azure involvement
3. shadcn (UI components) -- conditional on UI work

**nick:execute** coordinates the same 3, with a different purpose (verification vs. research).

**nick:verify** uses only Chrome DevTools but internally coordinates it with workflow files and sub-agents.

**nick:review** uses only IDE diagnostics (`mcp__ide__getDiagnostics`).

### Recommended Multi-MCP Workflow for GSD

```
Planning Phase:
  1. Context7: Resolve + query for each library (MANDATORY)
  2. microsoft-docs: Search + sample + fetch for Azure services (IF Azure)
  3. shadcn: getComponents + getComponent for UI components (IF UI)
  4. Figma: get_design_context for design specs (IF design provided)

  Output: Research findings embedded in plan

Execution Phase:
  1. Context7: Verify plan's API references are current (MANDATORY)
  2. microsoft-docs: Get implementation details for Azure (IF Azure)
  3. shadcn: Get exact component usage patterns (IF UI)

  Output: Correct code, deviations noted

Verification Phase:
  1. Chrome DevTools: Automated workflow tests (IF workflow exists)
  2. Chrome DevTools: Runtime inspection (console, network, performance)
  3. IDE diagnostics: TypeScript errors across modified files

  Output: Verification report

Review Phase:
  1. IDE diagnostics: getDiagnostics for all modified files
  2. (No other MCPs needed -- review is about code quality, not docs)

  Output: Review with pass/fail
```

### Coordination Rules

1. **Never re-query what was already looked up.** If planning produced research findings with API details, execution reads those findings first. Only re-query Context7 if the plan's references seem outdated or incomplete.

2. **Pass paths, not content.** When an agent spawns sub-agents, pass the path to the research file, not its content. Sub-agents have fresh 200k context and can read files efficiently.

3. **Validate between phases.** Before moving from planning MCP calls to execution MCP calls, ensure the plan is complete and approved. Do not mix phases.

4. **Fail fast, report clearly.** If any MCP is unavailable, report it immediately with the specific tool that failed. Do not silently fall back.

---

## Graceful Degradation

### Degradation Hierarchy

```
Level 0: All MCPs available
  -> Full pipeline: Context7 + microsoft-docs + shadcn + Chrome DevTools + IDE

Level 1: Context7 unavailable
  -> Fallback: WebSearch + WebFetch for official docs
  -> Impact: API references may be stale; mark confidence as LOW
  -> Mitigation: Include year in searches, verify with official docs URLs

Level 2: Chrome DevTools unavailable
  -> Fallback: npm run test + manual visual verification
  -> Impact: No automated runtime verification
  -> Mitigation: Mark verification as "inconclusive", ask user for visual check

Level 3: microsoft-docs unavailable
  -> Fallback: WebSearch + WebFetch for learn.microsoft.com
  -> Impact: May miss Azure-specific patterns
  -> Mitigation: Still usable -- Azure docs are publicly accessible via web

Level 4: IDE diagnostics unavailable
  -> Fallback: npx tsc --noEmit for TypeScript errors
  -> Impact: Minimal -- CLI typecheck catches most issues

Level 5: All MCPs unavailable
  -> Fallback: Training data + WebSearch + CLI tools
  -> Impact: Significant quality reduction
  -> Mitigation: Flag prominently: "No MCPs available. All API references from training data (may be stale)."
```

### Implementation Pattern

Each skill should include a preamble check:

```markdown
## Pre-flight Check

Before starting work, verify MCP availability:

1. Test Context7: attempt `resolve-library-id` for a known library (e.g., "react")
2. If it fails: log "Context7 unavailable", set CONTEXT7_AVAILABLE=false
3. Test microsoft-docs: attempt `microsoft_docs_search` for "Azure" (if Azure feature)
4. If it fails: log "microsoft-docs unavailable", set MS_DOCS_AVAILABLE=false

Adjust workflow based on availability:
- If Context7 unavailable: use WebSearch + WebFetch, mark all API refs as LOW confidence
- If microsoft-docs unavailable: use WebSearch for Azure docs
- If Chrome DevTools unavailable: skip automated browser tests, offer alternatives
- If IDE diagnostics unavailable: use CLI typecheck instead
```

### The Confidence Tag Pattern

From gsd-phase-researcher's source hierarchy, confidence tagging is the key mechanism for graceful degradation:

```
HIGH:  Verified via Context7 or official docs MCP
MEDIUM: Verified via WebFetch of official docs URL
LOW:   WebSearch only, or training data only

Rule: Never present LOW confidence findings as authoritative.
```

When MCPs are unavailable, all findings automatically get lower confidence tags, which propagate downstream. The planner sees LOW confidence tags and adds validation steps. The executor sees LOW confidence tags and re-verifies before implementing.

---

## Allowed-Tools Declarations

### Skills Format (nick:plan, nick:execute, etc.)

Skills use a YAML array in frontmatter with explicit tool names:

```yaml
---
name: nick:plan
description: Plan a feature...
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write
  - Task
  - WebSearch
  - WebFetch
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
  - mcp__microsoft-docs__microsoft_docs_search
  - mcp__microsoft-docs__microsoft_code_sample_search
  - mcp__microsoft-docs__microsoft_docs_fetch
  - mcp__shadcn__getComponents
  - mcp__shadcn__getComponent
---
```

**Advantages:**
- Exact tool set documented in frontmatter
- No accidental access to tools not needed
- Self-documenting: reading the frontmatter tells you what MCPs the skill uses
- Matches Anthropic's skill guide format

### Agent Format (gsd-planner, gsd-phase-researcher, etc.)

Agents use comma-separated `tools:` with wildcards:

```yaml
---
name: gsd-phase-researcher
tools: Read, Write, Bash, Grep, Glob, WebSearch, WebFetch, mcp__context7__*
---
```

**Advantages:**
- Shorter declaration
- Automatically includes new tools added to the MCP server
- Works for agents that need broad access

**Disadvantages:**
- `mcp__context7__*` grants access to ALL Context7 tools, including future ones
- Less self-documenting
- Does not match skill format

### Recommended Pattern for GSD Skills

Use explicit tool lists (skill format) for all converted commands:

```yaml
# Planning skills
allowed-tools:
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
  - mcp__microsoft-docs__microsoft_docs_search
  - mcp__microsoft-docs__microsoft_code_sample_search
  - mcp__microsoft-docs__microsoft_docs_fetch
  - mcp__shadcn__getComponents
  - mcp__shadcn__getComponent

# Execution skills (same as planning, for verification)
allowed-tools:
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
  - mcp__microsoft-docs__microsoft_docs_search
  - mcp__microsoft-docs__microsoft_code_sample_search
  - mcp__microsoft-docs__microsoft_docs_fetch
  - mcp__shadcn__getComponents
  - mcp__shadcn__getComponent

# Verification skills (Chrome DevTools focus)
allowed-tools:
  - mcp__chrome-devtools__take_snapshot
  - mcp__chrome-devtools__take_screenshot
  - mcp__chrome-devtools__navigate_page
  - mcp__chrome-devtools__list_pages
  - mcp__chrome-devtools__select_page
  - mcp__chrome-devtools__list_console_messages
  - mcp__chrome-devtools__get_console_message
  - mcp__chrome-devtools__list_network_requests
  - mcp__chrome-devtools__get_network_request
  - mcp__chrome-devtools__evaluate_script
  - mcp__chrome-devtools__performance_start_trace
  - mcp__chrome-devtools__performance_stop_trace
  - mcp__chrome-devtools__performance_analyze_insight
  - mcp__chrome-devtools__click
  - mcp__chrome-devtools__fill
  - mcp__chrome-devtools__hover
  - mcp__chrome-devtools__press_key
  - mcp__chrome-devtools__emulate
  - mcp__chrome-devtools__wait_for

# Review skills (IDE diagnostics only)
allowed-tools:
  - mcp__ide__getDiagnostics

# Design-to-code skills (Figma)
allowed-tools:
  - mcp__figma__get_design_context
  - mcp__figma__get_screenshot
  - mcp__figma__get_metadata
  - mcp__figma__get_variable_defs
```

### Cross-Runtime Compatibility

From the install.js analysis:

- **Claude Code:** MCP tools keep their `mcp__*` format as-is
- **OpenCode:** MCP tools keep their `mcp__*` format (pass-through)
- **Gemini CLI:** MCP tools are EXCLUDED from frontmatter (auto-discovered at runtime from `mcpServers` config)

This means the explicit `allowed-tools` approach works for Claude Code and OpenCode. For Gemini CLI, the install script already strips MCP tools, so no special handling is needed.

---

## Recommendations

### Priority 1: Convert GSD agents to use MCP-first patterns

**Effort:** Medium | **Impact:** High

The 3 agents that already have `mcp__context7__*` (planner, phase-researcher, project-researcher) should be upgraded to:
1. Use explicit tool lists instead of wildcards
2. Add microsoft-docs and shadcn MCP tools where relevant
3. Add mandatory lookup instructions matching nick:plan/nick:execute patterns

The remaining 8 agents should be evaluated for MCP needs:
- `gsd-executor`: NEEDS Context7 + microsoft-docs + shadcn (same as nick:execute)
- `gsd-verifier`: NEEDS Chrome DevTools (same as nick:verify)
- `gsd-debugger`: COULD BENEFIT from Context7 (checking API docs during debugging)
- `gsd-codebase-mapper`, `gsd-integration-checker`, `gsd-plan-checker`, `gsd-research-synthesizer`, `gsd-roadmapper`: Do not need MCPs (they work with local files only)

### Priority 2: Add Chrome DevTools verification to execution pipeline

**Effort:** Medium | **Impact:** High

The current GSD `verify-work` command uses `gsd-verifier` which does goal-backward verification by reading files. It does NOT use Chrome DevTools. Converting this to use nick:verify's pattern (automated workflow tests + Chrome DevTools inspection) would dramatically improve verification quality.

Implementation path:
1. Add Chrome DevTools tools to `gsd-verifier` agent's tool list
2. Add the snapshot-interact-verify protocol to the verification workflow
3. Integrate the test-flow pattern (test-planner -> browser-tester delegation)

### Priority 3: Implement the confidence-tag system framework-wide

**Effort:** Low | **Impact:** Medium

The phase-researcher already uses HIGH/MEDIUM/LOW confidence tags with source attribution. This pattern should be standardized across all research and documentation:
- All Context7-verified findings: tag as HIGH
- All WebSearch-only findings: tag as LOW
- Downstream agents respect tags (LOW = add validation step)
- MCP unavailability automatically downgrades tags

### Priority 4: Add graceful degradation to all MCP-using skills

**Effort:** Low | **Impact:** Medium

Every skill that uses MCPs should include a pre-flight check section and fallback instructions. Pattern from this research:
```
1. Test MCP availability at start
2. Log unavailability clearly
3. Fall back to WebSearch + WebFetch
4. Downgrade confidence tags
5. Flag in output that references are unverified
```

### Priority 5: Token-optimize MCP lookups with scoped queries

**Effort:** Low | **Impact:** Medium

Enforce the pattern: "look up the specific APIs you'll reference" (not "look up everything about library X"). Context7's `query` parameter should be as specific as possible:
- Bad: `query: "React"` (returns generic overview)
- Good: `query: "useEffect cleanup function for async operations"` (returns targeted docs)

### Priority 6: Add Figma MCP to planning pipeline

**Effort:** Low | **Impact:** Low-Medium

When a design file URL is provided with a task, the planning skill should:
1. `mcp__figma__get_design_context` for UI code generation hints
2. `mcp__figma__get_screenshot` for visual reference
3. Feed design context into the component hierarchy section of the plan

This is lower priority because it only applies when Figma designs are provided.

### Priority 7: Research Context7 cache and rate limits

**Effort:** Low | **Impact:** Low

Context7 documentation warns about a 3-call-per-question limit. For GSD's agent-based architecture where multiple agents may query Context7 independently, this limit applies per agent (each gets a fresh context). However, if the same library is queried by the researcher agent and then again by the planner agent, that is redundant. The recommendation: have the researcher produce RESEARCH.md with all needed API details, and have downstream agents read that file instead of re-querying.

---

## Appendix: MCP Tool Inventory

### Context7 (2 tools)
- `mcp__context7__resolve-library-id` -- Resolve library name to ID
- `mcp__context7__query-docs` -- Query documentation by library ID

### microsoft-docs (3 tools)
- `mcp__microsoft-docs__microsoft_docs_search` -- Search Microsoft Learn
- `mcp__microsoft-docs__microsoft_code_sample_search` -- Search code samples
- `mcp__microsoft-docs__microsoft_docs_fetch` -- Fetch full doc page

### Chrome DevTools (25 tools)
- Navigation: `navigate_page`, `list_pages`, `select_page`, `new_page`, `close_page`
- Inspection: `take_snapshot`, `take_screenshot`
- Interaction: `click`, `fill`, `fill_form`, `hover`, `press_key`, `drag`, `upload_file`
- Console: `list_console_messages`, `get_console_message`
- Network: `list_network_requests`, `get_network_request`
- Performance: `performance_start_trace`, `performance_stop_trace`, `performance_analyze_insight`
- Environment: `emulate`, `resize_page`, `wait_for`
- Dialog: `handle_dialog`
- Script: `evaluate_script`

### shadcn (2 tools)
- `mcp__shadcn__getComponents` -- List available components
- `mcp__shadcn__getComponent` -- Get component details

### Figma (6 tools)
- `mcp__figma__get_design_context` -- Generate UI code for a node
- `mcp__figma__get_screenshot` -- Screenshot a node
- `mcp__figma__get_metadata` -- Get node metadata (XML)
- `mcp__figma__get_variable_defs` -- Get variable definitions
- `mcp__figma__get_figjam` -- FigJam file support
- `mcp__figma__create_design_system_rules` -- Generate design system rules

### IDE (2 tools)
- `mcp__ide__getDiagnostics` -- Get language diagnostics
- `mcp__ide__executeCode` -- Execute code in Jupyter kernel

---

*Research completed: 2026-02-15*
*Sources: Nick's existing skills (plan.md, execute.md, verify.md, review.md), GSD agent files (gsd-phase-researcher.md, gsd-project-researcher.md, gsd-planner.md), Claude Code Skill Guide (Chapter 5 patterns), GSD architecture research (ARCHITECTURE.md, FEATURES.md), discovery-phase.md workflow, install.js MCP handling code*
