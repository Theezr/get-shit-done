# Phase 2: MCP-Powered Planning & Execution - Research

**Researched:** 2026-02-15
**Domain:** MCP integration into GSD planning/execution skill agents
**Confidence:** HIGH

## Summary

Phase 2 integrates MCP servers (Context7, microsoft-docs) and best-practice skill loading into the existing GSD plan-phase and execute-phase skills created in Phase 1. The work is purely about modifying agent definition files in the repo at `skills/` -- the researcher, planner, executor, and the two SKILL.md orchestrators. No new skills are created; existing agent definitions get enhanced sections and the orchestrator SKILL.md files get updated `allowed-tools` lists.

The user's existing `nick:plan` and `nick:execute` skills (at `docs/nick/plan.md` and `docs/nick/execute.md`) serve as the reference implementation. They demonstrate the exact pattern: mandatory Context7 2-step lookup for every library, microsoft-docs 3-step lookup for Azure features, shadcn MCP for UI components, and skill loading before writing code. Phase 2 adapts these patterns into the GSD agent architecture, which differs from the nick:* skills in that GSD uses a multi-agent pipeline (orchestrator spawns researcher, planner, executor, verifier) rather than a single-skill approach.

A critical discovery: the `mcp__shadcn__*` tools referenced in MCP-03 do not exist in the current system's MCP configuration. The official shadcn MCP server exists and can be configured (`npx shadcn@latest mcp`), but with tool names that differ from what the requirements assume (`getComponents`/`getComponent`). Since this MCP is not installed, the implementation should prepare for it gracefully: add the tool names to allowed-tools, include instructions for when shadcn MCP is available, and provide a Context7/WebFetch fallback for shadcn/ui documentation when the MCP is unavailable.

**Primary recommendation:** Modify 5 agent definition files and 2 SKILL.md files in the repo's `skills/` directory. Changes are additive sections within existing agent definitions (new `<mcp_protocol>` sections, enhanced `<tool_strategy>` sections) plus `allowed-tools` additions in SKILL.md frontmatter.

## Standard Stack

### Core

| Component | Location | Purpose | Why Standard |
|-----------|----------|---------|--------------|
| Context7 MCP | `mcp__context7__resolve-library-id`, `mcp__context7__query-docs` | Library documentation lookup | Already in plan-phase allowed-tools; proven 2-step pattern (resolve then query) |
| microsoft-docs MCP | `mcp__microsoft-docs__microsoft_docs_search`, `mcp__microsoft-docs__microsoft_code_sample_search`, `mcp__microsoft-docs__microsoft_docs_fetch` | Azure documentation lookup | Confirmed tool names; 3-step pattern (search, code samples, fetch) |
| Best-practice skills | `~/.claude/skills/{name}/SKILL.md` | Code quality standards | 5 skills already installed (nestjs, vercel-react, owasp, web-design-guidelines, security-review) |
| Agent definition files | `skills/gsd-*/references/agents/*.md` | Agent role instructions with MCP protocols | Phase 1 created these; Phase 2 modifies them |

### Supporting

| Component | Location | Purpose | When to Use |
|-----------|----------|---------|-------------|
| shadcn MCP (future) | `mcp__shadcn__*` (not yet installed) | UI component lookup | When shadcn MCP is configured; fallback to Context7/WebFetch otherwise |
| WebFetch | Built-in tool | Fetch official docs as fallback | When MCP is unavailable for a specific library |
| WebSearch | Built-in tool | Ecosystem discovery | When neither Context7 nor official docs cover a topic |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Context7 for shadcn/ui docs | shadcn MCP server | shadcn MCP not installed; Context7 can resolve shadcn/ui as a library; use Context7 as primary fallback |
| Embedding MCP instructions in agent body | New `<mcp_protocol>` XML section | Dedicated section is cleaner, easier to find, avoids bloating existing sections |
| Loading all best-practice skills always | Heuristic-based skill detection | Loading all wastes context; detect relevant skills from plan files/tech stack |
| Modifying skills at `~/.claude/skills/` | Modifying repo source at `skills/` | Repo is source of truth; install script copies to `~/.claude/skills/` |

## Architecture Patterns

### Pattern 1: Mandatory Context7 2-Step Lookup (Researcher)

**What:** The researcher agent MUST perform Context7 lookups for every library the phase will touch, not optionally.

**When to use:** Every research invocation that involves external libraries.

**Current state (optional):**
```markdown
<!-- From gsd-phase-researcher.md tool_strategy -->
| 1st | Context7 | Library APIs, features, configuration, versions | HIGH |
```
The current researcher mentions Context7 as "1st priority" but usage is optional -- there is no enforcement mechanism. The researcher can skip Context7 and still produce valid RESEARCH.md.

**Target state (mandatory):**
```markdown
<mcp_protocol>

## Mandatory Context7 Lookups

**For EVERY library the phase implementation will touch:**

1. Identify all libraries from phase requirements, roadmap goal, and codebase analysis
2. For each library:
   a. `mcp__context7__resolve-library-id` with library name
   b. `mcp__context7__query-docs` with resolved ID + specific query about APIs that will be used
3. Record findings in RESEARCH.md under `## Standard Stack` with version and API details

**Do NOT skip this. Do NOT assume you know the current API. Libraries change.**

**If Context7 cannot resolve a library:** Fall back to WebFetch of official docs. Mark confidence as MEDIUM instead of HIGH.

**If neither Context7 nor WebFetch works:** Use training data. Mark confidence as LOW and flag for validation.

### Confidence Tagging from Source

| Source | Confidence | Tag in RESEARCH.md |
|--------|-----------|---------------------|
| Context7 verified | HIGH | "Verified via Context7" |
| Official docs via WebFetch | MEDIUM | "Verified via official docs" |
| WebSearch cross-verified | MEDIUM | "WebSearch verified with [source]" |
| Training data only | LOW | "Training data -- needs validation" |

</mcp_protocol>
```

### Pattern 2: microsoft-docs 3-Step Lookup (Researcher)

**What:** When the phase involves Azure services, the researcher performs a 3-step microsoft-docs MCP lookup.

**When to use:** Any phase that touches Azure services (App Service, Functions, AKS, CosmosDB, Blob Storage, Key Vault, Entra ID, etc.)

**Target section (add to researcher `<mcp_protocol>`):**
```markdown
## Azure Documentation Lookup

**If the phase involves ANY Azure service:**

1. `mcp__microsoft-docs__microsoft_docs_search` -- find relevant documentation
2. `mcp__microsoft-docs__microsoft_code_sample_search` -- find code examples (specify language: TypeScript, JavaScript, etc.)
3. `mcp__microsoft-docs__microsoft_docs_fetch` -- fetch full page content for key articles

**Azure service indicators:** Azure SDK imports (`@azure/*`), ARM templates, Bicep files, Azure CLI commands, `process.env.AZURE_*` or `APPLICATIONINSIGHTS_*` patterns, App Service/Functions configuration.

Record Azure findings in RESEARCH.md under a `## Azure Services` section.
```

### Pattern 3: shadcn/UI Component Lookup with Graceful Fallback

**What:** When the phase involves UI component planning, look up shadcn component details -- but handle the case where the shadcn MCP is not installed.

**When to use:** Any phase that involves UI component work with shadcn/ui.

**Target section (add to researcher `<mcp_protocol>`):**
```markdown
## UI Component Documentation (shadcn/ui)

**If the phase involves UI components and the project uses shadcn/ui:**

**Primary (if shadcn MCP available):**
1. Use shadcn MCP tools to browse/search available components
2. Get detailed component docs for each component the plan will use

**Fallback (if shadcn MCP unavailable):**
1. Use Context7: resolve "shadcn/ui" then query for specific component docs
2. If Context7 lacks coverage: WebFetch from `https://ui.shadcn.com/docs/components/{component-name}`

**Detection:** Check `components.json` or `package.json` for `shadcn` dependency. Check for `@/components/ui/` import patterns.

Record component findings in RESEARCH.md under `## UI Components` section with usage patterns and props.
```

### Pattern 4: Executor Pre-Implementation API Verification

**What:** Before implementing any task, the executor verifies the plan's API references against current Context7 docs. This catches stale APIs from the plan.

**When to use:** Every executor task that uses external libraries.

**Current state (reactive):**
```markdown
<!-- From gsd-executor.md tool_strategy -->
When to use:
- Unsure about a library's API or method signature
- Need current configuration patterns or setup code
```
The current executor only uses Context7 when "unsure" -- this is reactive, not proactive.

**Target state (proactive pre-verification):**
```markdown
<mcp_protocol>

## Pre-Implementation API Verification

**Before implementing each task:**

1. Scan the task's `<action>` for library references (imports, API calls, method names)
2. For each library referenced:
   a. `mcp__context7__resolve-library-id` with library name
   b. `mcp__context7__query-docs` with resolved ID + the specific API/method from the plan
3. Compare plan's API usage with Context7 results:
   - **Match:** Proceed with plan's instructions
   - **Mismatch:** Adapt to current API. Document deviation: "Plan referenced X, current API is Y"
   - **Not found in Context7:** Proceed with plan's instructions (plan was already researched)

**When NOT to verify:** Pure codebase-internal work (no external library APIs), configuration-only tasks, file copy/move tasks.

**Efficiency:** Batch resolve-library-id calls. Only query-docs for APIs you are about to use, not entire library docs.

</mcp_protocol>
```

### Pattern 5: Best-Practice Skill Loading (Executor)

**What:** Before writing code, the executor loads relevant best-practice skills based on what the task involves.

**When to use:** Every executor task that writes or modifies code.

**Reference implementation (from nick:execute):**
```markdown
**React/Next.js code:** Load the `vercel-react-best-practices` skill BEFORE
writing any React component, hook, or Next.js page.

**NestJS code:** Load the `nestjs-best-practices` skill BEFORE writing any
NestJS controller, service, module, guard, pipe, or interceptor.

**UI components:** Load the `web-design-guidelines` skill BEFORE writing any
visual component, page, or layout.

**Security-sensitive code:** Load the `owasp-security` skill when implementing
auth, input handling, API endpoints, or anything touching user data.
```

**Target section (add to executor):**
```markdown
<best_practice_skills>

## Load Best-Practice Skills Before Writing Code

**MANDATORY:** Before implementing code in a task, read the relevant best-practice skill SKILL.md files to follow their patterns.

### Detection Heuristics

| Task Indicators | Skill to Load | Skill Path |
|-----------------|---------------|------------|
| React components, hooks, Next.js pages/routes, JSX/TSX files | vercel-react-best-practices | `~/.claude/skills/vercel-react-best-practices/SKILL.md` |
| NestJS controllers, services, modules, decorators (@Controller, @Injectable) | nestjs-best-practices | `~/.claude/skills/nestjs-best-practices/SKILL.md` |
| Visual components, pages, layouts, UI styling, accessibility | web-design-guidelines | `~/.claude/skills/web-design-guidelines/SKILL.md` |
| Auth, input validation, API endpoints, user data handling, secrets | owasp-security | `~/.claude/skills/owasp-security/SKILL.md` |

### How to Load

1. Read the skill's SKILL.md to get the rule categories and priorities
2. For detailed rules, read the referenced files in `references/` as needed
3. Apply CRITICAL-priority rules as hard requirements
4. Apply HIGH-priority rules as strong recommendations

### Loading Strategy

- **Read SKILL.md first** (contains rule summary and references to detailed docs)
- **Read specific references/** only for the categories relevant to your task
- **Multiple skills can apply** to a single task (e.g., React + OWASP for a login form)
- **Do NOT load all skills for every task** -- only load what the task indicators match

### Documenting in SUMMARY.md

Record which skills were loaded in the SUMMARY frontmatter:
```yaml
skills-loaded:
  - vercel-react-best-practices
  - owasp-security
```

</best_practice_skills>
```

### Pattern 6: Planner Context7 for Discovery (Not Research)

**What:** The planner uses Context7 for quick Level 1 verification (confirming syntax/version for known libraries), not for deep research (that is the researcher's job).

**Current state:** The planner already has `mcp__context7__*` in its tools list and mentions Context7 in discovery_levels. No changes needed to the planner agent itself for MCP-01 -- the researcher handles the mandatory lookups and writes findings to RESEARCH.md, which the planner reads.

**Key insight:** MCP-01 is primarily a researcher responsibility. The planner benefits indirectly by reading the researcher's RESEARCH.md which now contains Context7-verified API details. The planner only uses Context7 directly for Level 1 quick verification during planning.

### Anti-Patterns to Avoid

- **Double-querying:** Researcher already looked up library docs via Context7. Executor should check RESEARCH.md first before re-querying Context7. Only re-verify if the specific API being used is not covered in RESEARCH.md.
- **Loading all skills always:** Reading all 4 best-practice skills for every task wastes context. Only load skills that match the task's technology indicators.
- **Blocking on unavailable MCP:** If Context7 is down or a library is not indexed, do not stop. Fall back to WebFetch/WebSearch and tag confidence accordingly.
- **Treating MCP results as infallible:** Context7 docs can be outdated or incomplete. If MCP results conflict with the codebase's existing patterns, follow the codebase and document the discrepancy.
- **Modifying installed skills at `~/.claude/skills/`:** Phase 2 modifies repo source files at `skills/gsd-*/`. The install script copies these to `~/.claude/skills/`. Never modify installed copies directly.

## Files to Modify

This is the core deliverable mapping. Each file gets specific additions:

### 1. `skills/gsd-plan-phase/SKILL.md` (Orchestrator)

**Changes:**
- Add to `allowed-tools`: `mcp__microsoft-docs__microsoft_docs_search`, `mcp__microsoft-docs__microsoft_code_sample_search`, `mcp__microsoft-docs__microsoft_docs_fetch`
- Optionally add shadcn MCP tools to allowed-tools (for future readiness)
- Update researcher spawn prompt to mention MCP requirements in `<objective>`

### 2. `skills/gsd-plan-phase/references/agents/gsd-phase-researcher.md` (Researcher Agent)

**Changes:**
- Add new `<mcp_protocol>` section with mandatory Context7 lookup protocol
- Add microsoft-docs 3-step lookup protocol to `<mcp_protocol>`
- Add shadcn/UI component lookup with graceful fallback to `<mcp_protocol>`
- Update `<tool_strategy>` to reference the new `<mcp_protocol>` section
- Add `mcp__microsoft-docs__*` to the `tools:` frontmatter line
- Update RESEARCH.md output format to include `## Azure Services` and `## UI Components` sections when applicable

### 3. `skills/gsd-plan-phase/references/agents/gsd-planner.md` (Planner Agent)

**Changes:**
- No significant changes needed for MCP-01 (researcher handles mandatory lookups)
- Minor: Add awareness note that RESEARCH.md now contains Context7-verified API details with confidence tags
- Ensure discovery_levels Level 1 still mentions Context7 for quick verification

### 4. `skills/gsd-execute-phase/SKILL.md` (Orchestrator)

**Changes:**
- Add to `allowed-tools`: `mcp__context7__resolve-library-id`, `mcp__context7__query-docs`
- The orchestrator itself does not use MCP tools, but listing them allows the skill to be invoked in contexts where these tools are needed
- Update executor spawn prompt to mention pre-implementation verification and skill loading in `<objective>`

### 5. `skills/gsd-execute-phase/references/agents/gsd-executor.md` (Executor Agent)

**Changes:**
- Replace reactive `<tool_strategy>` with proactive `<mcp_protocol>` for pre-implementation API verification
- Add new `<best_practice_skills>` section with detection heuristics and loading protocol
- Ensure `mcp__context7__*` remains in `tools:` frontmatter
- Add skill loading documentation to `<summary_creation>` (skills-loaded field)

### 6. `skills/gsd-plan-phase/references/agents/gsd-plan-checker.md` (Plan Checker)

**Changes:**
- No MCP tools needed (plan checker verifies plan structure, not API accuracy)
- Minor: Could add a check that plans reference RESEARCH.md findings rather than unverified claims
- This file has minimal changes for Phase 2

### 7. `skills/gsd-execute-phase/references/agents/gsd-verifier.md` (Verifier Agent)

**Changes:**
- No MCP tools needed (verifier checks codebase artifacts, not library docs)
- No changes for Phase 2

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Library doc lookup | Custom WebFetch-based doc scraper | Context7 MCP (resolve-library-id + query-docs) | Context7 indexes 15,000+ libraries; handles versioning, relevance ranking |
| Azure doc search | Manual Microsoft Learn browsing | microsoft-docs MCP (search + code-samples + fetch) | Semantic search against full Microsoft Learn corpus; language-specific code samples |
| Best-practice enforcement | Inline code quality rules in executor | Existing SKILL.md files at `~/.claude/skills/` | Already curated, categorized by priority, with references/ for detailed rules |
| MCP availability detection | Try/catch around every MCP call | Pre-flight pattern: attempt one small call, fallback gracefully | Phase 4 (MCP-08) will formalize this; for Phase 2, simple try-then-fallback is sufficient |
| Skill detection heuristics | Complex AST analysis of plan files | Simple keyword/pattern matching on task files/action text | Detection does not need to be perfect -- false positive (loading an unneeded skill) costs some context but does not cause errors |

**Key insight:** The MCP servers and best-practice skills already exist and are proven. Phase 2's job is to wire them into the GSD agent pipeline with the right protocols and detection heuristics, not to build new capabilities.

## Common Pitfalls

### Pitfall 1: Context7 Library Name Mismatch

**What goes wrong:** `resolve-library-id` cannot find the library because the name does not match Context7's index (e.g., "next" vs "next.js" vs "vercel/next.js").
**Why it happens:** Library naming is inconsistent across ecosystems. Context7 uses specific identifiers.
**How to avoid:** Try the most common name first. If resolve fails, try variations (with/without scope, with/without .js suffix). Document the resolved ID for reuse.
**Warning signs:** "No library found" responses from resolve-library-id; agent proceeding without any library docs.

### Pitfall 2: Over-Querying Context7

**What goes wrong:** Agent queries Context7 for every import in the codebase, consuming excessive context window and time.
**Why it happens:** "Mandatory for every library" interpreted too literally.
**How to avoid:** Query libraries that the phase INTRODUCES or whose APIs the plan REFERENCES. Do not query standard Node.js built-ins, project-internal modules, or trivial utilities (e.g., `lodash.debounce`). Focus on framework-level libraries (React, Next.js, NestJS, Prisma) and new dependencies.
**Warning signs:** More than 8-10 Context7 queries per research session; querying `fs`, `path`, or `util`.

### Pitfall 3: Executor Re-Researching Everything

**What goes wrong:** The executor re-queries Context7 for libraries that the researcher already documented in RESEARCH.md.
**Why it happens:** Executor's pre-implementation verification duplicates researcher's work.
**How to avoid:** Executor should read RESEARCH.md first. Only re-verify APIs that (a) are not covered in RESEARCH.md, or (b) are used in a way the research did not anticipate.
**Warning signs:** Executor making 10+ Context7 calls per task; identical queries to what researcher already performed.

### Pitfall 4: Skill Loading Bloats Context

**What goes wrong:** Executor reads all 4 best-practice SKILL.md files (combined ~3,000+ lines) for every task, consuming 15-20% of context before writing any code.
**Why it happens:** "Load relevant skills" interpreted as "load all skills."
**How to avoid:** Use detection heuristics. Only load skills matching the task's technology. Read SKILL.md first (rule summary), then read specific `references/` files only for the relevant rule categories.
**Warning signs:** Context usage exceeding 40% before first file write; reading entire `references/` directories of skills.

### Pitfall 5: shadcn MCP Tools Assumed Available

**What goes wrong:** Agent instructions reference `mcp__shadcn__getComponents` / `mcp__shadcn__getComponent` but these tools do not exist in the system, causing errors.
**Why it happens:** MCP-03 requirement assumes shadcn MCP is installed; it is not.
**How to avoid:** Always provide a fallback path. Check MCP availability first. Use Context7 or WebFetch for shadcn docs when MCP is unavailable. Include shadcn tools in allowed-tools for forward compatibility but never depend on them exclusively.
**Warning signs:** MCP tool call errors; agents failing silently when shadcn MCP is unavailable.

### Pitfall 6: Modifying Installed Copies Instead of Repo Source

**What goes wrong:** Developer modifies `~/.claude/skills/gsd-plan-phase/...` directly. Changes are overwritten on next install.
**Why it happens:** Installed copies are what the system reads; instinct is to edit them.
**How to avoid:** Always edit repo source files at `skills/gsd-*/`. Use the install script to copy to `~/.claude/skills/`.
**Warning signs:** Changes at `~/.claude/skills/` that do not match repo source; drift between installed and repo versions.

## Code Examples

### Example 1: Mandatory Context7 Lookup in Researcher

```markdown
<!-- Add to gsd-phase-researcher.md as new <mcp_protocol> section -->

<mcp_protocol>

## Mandatory MCP Lookups

### Context7: Library Documentation

**For EVERY external library the phase implementation will touch:**

1. Identify libraries from: phase requirements, roadmap goal, existing package.json
2. For each library:
   a. `mcp__context7__resolve-library-id` -- resolve library name to Context7 ID
   b. `mcp__context7__query-docs` -- query specific APIs the phase will use

**Example:**
```
Phase goal: "Add JWT authentication using jose library"
Libraries to look up:
1. jose -- resolve, then query "sign JWT", "verify JWT", "JWK"
2. next.js -- resolve, then query "middleware", "cookies"
```

**Record in RESEARCH.md `## Standard Stack` table:**
```markdown
| Library | Version | Purpose | Source |
|---------|---------|---------|--------|
| jose | 5.x | JWT sign/verify | Context7 verified |
| next.js | 15.x | Middleware for auth | Context7 verified |
```

### microsoft-docs: Azure Documentation

**If the phase involves ANY Azure service:**

1. `mcp__microsoft-docs__microsoft_docs_search` -- find relevant documentation
2. `mcp__microsoft-docs__microsoft_code_sample_search` -- find code examples
3. `mcp__microsoft-docs__microsoft_docs_fetch` -- fetch full article content

### shadcn/ui: Component Documentation

**If the phase involves UI components with shadcn/ui:**

1. Try Context7: resolve "shadcn/ui", query for specific components
2. Fallback: WebFetch `https://ui.shadcn.com/docs/components/{name}`

### Confidence Tagging

| Source | Confidence |
|--------|-----------|
| Context7 verified | HIGH |
| Official docs via WebFetch | MEDIUM |
| WebSearch cross-verified | MEDIUM |
| Training data only | LOW |

</mcp_protocol>
```

### Example 2: Executor Pre-Implementation Verification

```markdown
<!-- Replace existing <tool_strategy> in gsd-executor.md -->

<tool_strategy>

## Pre-Implementation: Verify APIs and Load Skills

### Step 1: Read RESEARCH.md (if exists)

Before any Context7 calls, check if the researcher already documented the libraries:
```bash
cat "$PHASE_DIR"/*-RESEARCH.md 2>/dev/null
```
If RESEARCH.md covers the library's API you need, skip Context7 for that library.

### Step 2: Context7 API Verification

For each library referenced in the task's <action> that is NOT covered in RESEARCH.md:
1. `mcp__context7__resolve-library-id` with library name
2. `mcp__context7__query-docs` with resolved ID + specific API from the plan
3. If plan's API differs from current docs: adapt and document deviation

### Step 3: Load Best-Practice Skills

Based on what the task builds:

| Task involves | Read this skill |
|---------------|-----------------|
| React/Next.js components, hooks, pages | `~/.claude/skills/vercel-react-best-practices/SKILL.md` |
| NestJS services, controllers, modules | `~/.claude/skills/nestjs-best-practices/SKILL.md` |
| UI layouts, visual components, accessibility | `~/.claude/skills/web-design-guidelines/SKILL.md` |
| Auth, validation, API security, user data | `~/.claude/skills/owasp-security/SKILL.md` |

Read SKILL.md first for rule summary. Read specific references/ only for relevant categories.

### Step 4: Implement

Follow the plan. Apply best-practice rules as constraints on your implementation.

</tool_strategy>
```

### Example 3: Updated allowed-tools in plan-phase SKILL.md

```yaml
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - WebFetch
  - WebSearch
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
  - mcp__microsoft-docs__microsoft_docs_search
  - mcp__microsoft-docs__microsoft_code_sample_search
  - mcp__microsoft-docs__microsoft_docs_fetch
```

### Example 4: Updated allowed-tools in execute-phase SKILL.md

```yaml
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Task
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
```

### Example 5: Executor Spawn Prompt Update (in execute-phase SKILL.md)

```markdown
Task(
  prompt="First, read ~/.claude/skills/gsd-execute-phase/references/agents/gsd-executor.md for your role.

  <objective>
  Execute plan {plan_id} of phase {phase_number}-{phase_name}.

  BEFORE implementing each task:
  1. Verify the plan's API references via Context7 (check RESEARCH.md first)
  2. Load relevant best-practice skills for the technology being used

  Commit each task atomically. Create SUMMARY.md. Update STATE.md.
  </objective>

  <files_to_read>
  - Plan: {phase_dir}/{plan_file}
  - Research: {phase_dir}/*-RESEARCH.md (if exists -- check for pre-verified APIs)
  - State: .planning/STATE.md
  - Config: .planning/config.json (if exists)
  </files_to_read>

  <success_criteria>
  - [ ] API references verified via Context7 before implementation
  - [ ] Relevant best-practice skills loaded before writing code
  - [ ] All tasks executed
  - [ ] Each task committed individually
  - [ ] SUMMARY.md created with skills-loaded field
  - [ ] STATE.md updated with position and decisions
  </success_criteria>",
  subagent_type="general-purpose"
)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Context7 optional in researcher | Context7 MANDATORY for every library | Phase 2 (this work) | Every RESEARCH.md has verified API docs, not training-data guesses |
| No microsoft-docs integration | 3-step microsoft-docs lookup for Azure | Phase 2 (this work) | Azure features use current docs, not stale training data |
| Executor uses Context7 reactively ("when unsure") | Executor verifies APIs proactively before implementing | Phase 2 (this work) | Catches stale API references in plans before code is written |
| No best-practice skill loading | Executor loads relevant skills before coding | Phase 2 (this work) | Code follows established patterns (React perf, NestJS conventions, OWASP security) |
| shadcn MCP assumed available | Graceful fallback (Context7 or WebFetch) | Phase 2 (this work) | UI component docs available even without shadcn MCP installed |

**Not available (handle gracefully):**
- `mcp__shadcn__getComponents` / `mcp__shadcn__getComponent`: These specific tool names do not match any installed MCP. The official shadcn MCP exists (`npx shadcn@latest mcp`) but is not configured. Tool names in the official server differ from what MCP-03 assumes. Add to allowed-tools for forward compatibility; use Context7/WebFetch fallback.
- `frontend-design` skill: Referenced in PROJECT.md and requirements but does not exist at `~/.claude/skills/`. Use `web-design-guidelines` skill instead (which IS installed).

## Requirement-to-Change Mapping

| Requirement | What Changes | Files Modified |
|-------------|-------------|----------------|
| MCP-01: Mandatory Context7 in plan skill | Add `<mcp_protocol>` to researcher; mandatory 2-step lookup for every library | `gsd-phase-researcher.md`, `gsd-plan-phase/SKILL.md` (allowed-tools) |
| MCP-02: microsoft-docs for Azure | Add Azure lookup protocol to researcher `<mcp_protocol>`; add tools to allowed-tools | `gsd-phase-researcher.md`, `gsd-plan-phase/SKILL.md` (allowed-tools) |
| MCP-03: shadcn for UI components | Add shadcn lookup with Context7/WebFetch fallback to researcher; future-proof allowed-tools | `gsd-phase-researcher.md`, `gsd-plan-phase/SKILL.md` (allowed-tools, optional) |
| MCP-04: Context7 verification in executor | Replace reactive `<tool_strategy>` with proactive `<mcp_protocol>` | `gsd-executor.md`, `gsd-execute-phase/SKILL.md` (allowed-tools) |
| MCP-05: Best-practice skill loading | Add `<best_practice_skills>` section with detection heuristics | `gsd-executor.md` |

## Open Questions

1. **shadcn MCP tool name mismatch**
   - What we know: MCP-03 references `getComponents`/`getComponent`. The official shadcn MCP uses different tool names. The MCP is not installed in the current system.
   - What's unclear: What tool names the official shadcn MCP exposes when installed. Whether to use the official `npx shadcn@latest mcp` or a third-party server.
   - Recommendation: Add generic shadcn MCP references to allowed-tools. Provide Context7/WebFetch fallback. When shadcn MCP is eventually configured, update tool names to match actual server output. This satisfies MCP-03's intent without blocking on unavailable infrastructure.

2. **`frontend-design` skill does not exist**
   - What we know: Requirements and PROJECT.md reference a `frontend-design` skill, but `~/.claude/skills/` has `web-design-guidelines` instead. No `frontend-design` directory exists.
   - What's unclear: Whether `frontend-design` was renamed to `web-design-guidelines` or was never created.
   - Recommendation: Use `web-design-guidelines` in place of `frontend-design` in the executor's detection heuristics. Document this substitution.

3. **RESEARCH.md caching vs re-verification**
   - What we know: The executor should read RESEARCH.md before re-querying Context7 (TOKEN-03 from Phase 4). But Phase 2 is about making verification mandatory.
   - What's unclear: Exact boundary between "trust RESEARCH.md" and "always re-verify."
   - Recommendation: Executor reads RESEARCH.md first. Only re-verifies via Context7 if (a) the specific API is not documented in RESEARCH.md, or (b) the RESEARCH.md is from a previous phase (stale).

4. **Allowed-tools for sub-agents vs orchestrator**
   - What we know: `allowed-tools` in SKILL.md restricts the orchestrator, not sub-agents. Sub-agents get tools from their Task() spawn.
   - What's unclear: Whether adding `mcp__context7__*` to execute-phase SKILL.md's allowed-tools is needed (the orchestrator does not call Context7 directly; the executor agent does).
   - Recommendation: Add Context7 tools to execute-phase's allowed-tools for completeness and forward compatibility. The orchestrator may eventually use them for spot-checks.

## Sources

### Primary (HIGH confidence)
- `skills/gsd-plan-phase/SKILL.md` -- Current plan-phase orchestrator (read directly)
- `skills/gsd-plan-phase/references/agents/gsd-phase-researcher.md` -- Current researcher agent (read directly)
- `skills/gsd-plan-phase/references/agents/gsd-planner.md` -- Current planner agent (read directly)
- `skills/gsd-execute-phase/SKILL.md` -- Current execute-phase orchestrator (read directly)
- `skills/gsd-execute-phase/references/agents/gsd-executor.md` -- Current executor agent (read directly)
- `docs/nick/plan.md` -- User's reference planner skill with mandatory MCP integration (read directly)
- `docs/nick/execute.md` -- User's reference executor skill with skill loading pattern (read directly)
- `~/.claude/skills/nestjs-best-practices/SKILL.md` -- Installed best-practice skill (read directly)
- `~/.claude/skills/vercel-react-best-practices/SKILL.md` -- Installed best-practice skill (read directly)
- `~/.claude/skills/owasp-security/SKILL.md` -- Installed best-practice skill (read directly)
- `~/.claude/skills/web-design-guidelines/SKILL.md` -- Installed best-practice skill (read directly)

### Secondary (MEDIUM confidence)
- [Official shadcn MCP docs](https://ui.shadcn.com/docs/mcp) -- Confirmed official MCP exists with `npx shadcn@latest mcp` setup
- [Microsoft Learn MCP reference](https://learn.microsoft.com/en-us/training/support/mcp-developer-reference) -- Confirmed tool names: `microsoft_docs_search`, `microsoft_code_sample_search`, `microsoft_docs_fetch`
- [Context7 on Smithery](https://smithery.ai/server/@upstash/context7-mcp) -- Confirmed tools: `resolve-library-id`, `get-library-docs`

### Tertiary (LOW confidence)
- shadcn MCP exact tool names when installed -- not verified locally, may differ from `getComponents`/`getComponent` assumed in requirements
- `frontend-design` skill existence -- referenced in docs but not found at `~/.claude/skills/`; likely `web-design-guidelines` is the actual name

## Metadata

**Confidence breakdown:**
- MCP tool integration (Context7, microsoft-docs): HIGH -- tools confirmed available and working, patterns proven in nick:* skills
- Best-practice skill loading: HIGH -- skills exist, paths verified, detection heuristics straightforward
- shadcn MCP handling: MEDIUM -- MCP exists officially but not installed; fallback via Context7/WebFetch is reliable
- Architecture (additive sections to agents): HIGH -- modification pattern is clear; no structural changes needed to agent definitions

**Research date:** 2026-02-15
**Valid until:** 2026-03-15 (MCP tools are stable; best-practice skills update independently)
