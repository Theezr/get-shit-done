# MCP Integration Patterns & Pitfalls

## MCP Server Inventory

### Context7
**Role:** Up-to-date library/framework documentation lookup for accurate API references

**Tools:**
- `mcp__context7__resolve-library-id` — Resolves package/library name to Context7-compatible ID
- `mcp__context7__query-docs` — Queries documentation using the resolved library ID

**Which skills use it:**
- **nick:plan** — MANDATORY before referencing any library API in plan
- **nick:execute** — MANDATORY to re-verify plan's API references are current before implementation

**Required vs Optional:** REQUIRED for any skill involving library/framework code (React, Next.js, NestJS, Prisma, tRPC, Tailwind, shadcn/ui, Zod, React Hook Form, TanStack Query, etc.)

**Call limit:** Do not call more than 3 times per question (per tool documentation)

---

### microsoft-docs
**Role:** Official Microsoft/Azure documentation and code samples

**Tools:**
- `mcp__microsoft-docs__microsoft_docs_search` — Search Microsoft Learn documentation (returns up to 10 content chunks, max 500 tokens each)
- `mcp__microsoft-docs__microsoft_code_sample_search` — Search for code snippets with optional language filter (csharp, javascript, typescript, python, powershell, azurecli, etc.)
- `mcp__microsoft-docs__microsoft_docs_fetch` — Fetch complete webpage content as markdown

**Which skills use it:**
- **nick:plan** — MANDATORY for any Azure-related features (search → code samples → fetch)
- **nick:execute** — MANDATORY for Azure implementations (verify service configs, get current code examples)

**Required vs Optional:** REQUIRED for Azure features, OPTIONAL otherwise

**Follow-up Pattern:** Always use `microsoft_docs_fetch` when high-value pages are identified by search to get complete detail

---

### Chrome DevTools
**Role:** Runtime verification, browser automation, performance testing

**Tools:**
- Navigation: `list_pages`, `select_page`, `navigate_page`, `new_page`, `close_page`
- Inspection: `take_snapshot`, `take_screenshot`, `list_console_messages`, `get_console_message`
- Network: `list_network_requests`, `get_network_request`
- Performance: `performance_start_trace`, `performance_stop_trace`, `performance_analyze_insight`
- Interaction: `click`, `fill`, `fill_form`, `hover`, `press_key`, `drag`, `upload_file`
- Utilities: `evaluate_script`, `wait_for`, `emulate`, `resize_page`, `handle_dialog`

**Which skills use it:**
- **nick:verify** — Core tool for runtime verification (workflow tests + manual inspection)

**Required vs Optional:** REQUIRED for verifier role, NOT AVAILABLE to planner/executor/reviewer

**Verification Flow:** Always `take_snapshot` BEFORE every interaction (snapshot → interact → verify pattern)

---

### shadcn
**Role:** UI component library integration

**Tools:**
- `mcp__shadcn__getComponents` — List available shadcn components
- `mcp__shadcn__getComponent` — Get exact usage for specific component

**Which skills use it:**
- **nick:plan** — When planning UI with shadcn components
- **nick:execute** — When implementing shadcn components

**Required vs Optional:** REQUIRED when using shadcn/ui components, OPTIONAL otherwise

---

### IDE
**Role:** Language diagnostics (TypeScript errors, linting, etc.)

**Tools:**
- `mcp__ide__getDiagnostics` — Get diagnostics for all files or specific URI
- `mcp__ide__executeCode` — Execute code in Jupyter kernel (for notebooks only)

**Which skills use it:**
- **nick:review** — MANDATORY to check TypeScript errors across modified files

**Required vs Optional:** REQUIRED for reviewer role, OPTIONAL for others

---

### Figma
**Role:** Design-to-code handoff, generating UI from Figma designs

**Tools:**
- `mcp__figma__get_design_context` — Generate UI code from Figma node
- `mcp__figma__get_variable_defs` — Get variable definitions (colors, fonts, etc.)
- `mcp__figma__get_screenshot` — Generate screenshot of Figma node
- `mcp__figma__get_metadata` — Get metadata in XML format (structure overview)
- `mcp__figma__get_figjam` — Generate UI code from FigJam node
- `mcp__figma__create_design_system_rules` — Generate design system rules

**Which skills use it:**
- NOT CURRENTLY USED in Nick's skill prototypes (but available for design-to-code workflows)

**Required vs Optional:** OPTIONAL (specialized use case)

---

### Mermaid
**Role:** Diagram generation and validation

**Tools:**
- `mcp__mermaid__validate_and_render_mermaid_diagram` — Renders diagrams (validates during render)
- `mcp__mermaid__get_diagram_title` — Generate title from diagram content
- `mcp__mermaid__get_diagram_summary` — Generate summary from diagram content

**Which skills use it:**
- NOT CURRENTLY USED in Nick's skill prototypes (but available for documentation/visualization)

**Required vs Optional:** OPTIONAL (specialized use case)

---

## Integration Patterns

### Context7 Two-Step Pattern
**Mandatory Sequence:**
1. `mcp__context7__resolve-library-id` — Convert library name to Context7-compatible ID
2. `mcp__context7__query-docs` — Query using the resolved ID

**When to use:**
- BEFORE referencing ANY library API in plans (nick:plan)
- BEFORE implementing library code (nick:execute — re-verification step)

**Why it's mandatory:**
Libraries change. APIs get deprecated. Never assume you know the current API. Always verify.

**Common libraries to check:**
React, Next.js, NestJS, Prisma, tRPC, Tailwind, shadcn/ui, Zod, React Hook Form, TanStack Query

**Example from plan.md:**
```markdown
## Library documentation — context7 MCP
**For EVERY library or framework the implementation will touch:**
1. `mcp__context7__resolve-library-id` — resolve the library name
2. `mcp__context7__query-docs` — look up the specific APIs you'll reference in the plan

Do NOT skip this. Do NOT assume you know the current API. Libraries change.
```

**Pitfall Prevention:**
- Never query-docs without resolve-library-id first
- Respect the 3-call limit per question
- If resolve fails 3 times, use best available result

---

### Chrome DevTools Verification Flow
**Pattern:** snapshot → interact → verify

**Step-by-step:**
1. **Setup:**
   - `list_pages` — Find the running app
   - `select_page` — Select the page with the app
   - `navigate_page` — Navigate to the feature

2. **Before EVERY interaction:**
   - `take_snapshot` — Get current page structure (provides UIDs for elements)

3. **Interact:**
   - `click`, `fill`, `press_key`, etc. using UIDs from snapshot

4. **Verify:**
   - `list_console_messages` with `types: ["error", "warn"]` — Check for errors
   - `list_network_requests` — Verify API calls
   - `get_network_request` — Inspect specific request/response
   - `performance_start_trace` + `performance_stop_trace` + `performance_analyze_insight` — Check performance

**Multi-agent workflow (from verify.md):**
1. **test-planner agent (Sonnet):** Reads workflow file, produces structured test plan
2. **browser-tester agent (Haiku):** Executes test plan step-by-step using snapshot → interact → verify pattern

**Key Rules:**
- ALWAYS `take_snapshot` before interaction (snapshot provides UIDs)
- Environment issues (app not running, MCP disconnected) = "inconclusive", NOT failure
- Check for: zero console errors, correct API responses, no long tasks >50ms

**Example from verify.md:**
```markdown
1. `list_pages` — find the running app
2. `navigate_page` — navigate to the feature
3. `take_snapshot` — get the page structure
4. `list_console_messages` — check for errors/warnings (filter by `types: ["error", "warn"]`)
5. `list_network_requests` — verify API calls work correctly
6. `performance_start_trace` with `reload: true, autoStop: true` — check for performance issues
7. `performance_analyze_insight` — investigate any highlighted issues
```

---

### IDE Diagnostics Pattern
**When to check:**
- **During review phase** (nick:review) — MANDATORY to check TypeScript errors

**What to look for:**
- TypeScript errors across modified files
- Type strictness violations (no `any`, proper types)

**How to use:**
```typescript
mcp__ide__getDiagnostics({ uri: "optional-specific-file-path" })
// If no URI provided, gets diagnostics for ALL files
```

**Integration with review.md:**
```markdown
### 5. IDE Diagnostics

Use `mcp__ide__getDiagnostics` to check for TypeScript errors across the
modified files.
```

---

### Microsoft Docs Three-Step Pattern
**For Azure features:**
1. `microsoft_docs_search` — Find relevant documentation
2. `microsoft_code_sample_search` — Get code examples (specify language: csharp, javascript, typescript, python, powershell, azurecli, etc.)
3. `microsoft_docs_fetch` — Get full page content for key articles

**When to use:**
- ANY Azure service implementation
- Service configurations
- Current code examples

**Follow-up requirement:**
When search identifies high-value pages, use `microsoft_docs_fetch` to get complete detail (search returns truncated chunks, fetch returns full content)

---

## Allowed-Tools by Skill Role

### Planner (nick:plan)
**Core exploration:**
- Read, Glob, Grep, Bash

**Output:**
- Write, Task

**Mode control:**
- EnterPlanMode, ExitPlanMode

**Documentation lookup:**
- WebSearch, WebFetch
- mcp__context7__resolve-library-id
- mcp__context7__query-docs
- mcp__microsoft-docs__microsoft_docs_search
- mcp__microsoft-docs__microsoft_code_sample_search
- mcp__microsoft-docs__microsoft_docs_fetch
- mcp__shadcn__getComponents
- mcp__shadcn__getComponent

**NOT ALLOWED:**
- Edit (read-only during planning)
- Chrome DevTools (no runtime access)
- IDE diagnostics (happens at review)
- Git operations (no commits)

---

### Executor (nick:execute)
**Core implementation:**
- Read, Write, Edit, Glob, Grep, Bash

**Coordination:**
- Task, Skill

**Documentation re-verification:**
- WebSearch, WebFetch
- mcp__context7__resolve-library-id
- mcp__context7__query-docs
- mcp__microsoft-docs__microsoft_docs_search
- mcp__microsoft-docs__microsoft_code_sample_search
- mcp__microsoft-docs__microsoft_docs_fetch
- mcp__shadcn__getComponents
- mcp__shadcn__getComponent

**NOT ALLOWED:**
- Chrome DevTools (no runtime verification — that's verifier's job)
- IDE diagnostics (happens at review)
- Git commits (verifier commits AFTER review passes)

**Key rule:** Do NOT commit — the Verifier commits after review passes

---

### Verifier (nick:verify)
**Core verification:**
- Read, Glob, Grep, Bash, Write

**Coordination:**
- Task (for spawning test-planner + browser-tester agents)

**Chrome DevTools (FULL SUITE):**
- Navigation: list_pages, select_page, navigate_page, new_page, close_page
- Inspection: take_snapshot, take_screenshot, list_console_messages, get_console_message
- Network: list_network_requests, get_network_request
- Performance: performance_start_trace, performance_stop_trace, performance_analyze_insight
- Interaction: click, fill, fill_form, hover, press_key, drag, upload_file, wait_for
- Utilities: evaluate_script, emulate, resize_page, handle_dialog

**NOT ALLOWED:**
- Edit (read-only, report findings only)
- Context7/microsoft-docs (not needed for runtime verification)
- Git operations (reviewer commits, not verifier)

**Key rules:**
- NEVER modify source files — report findings only
- NEVER write `.pipeline/reviews/` file — that's reviewer's job
- NEVER commit — that's reviewer's job

---

### Reviewer (nick:review)
**Core review:**
- Read, Glob, Grep, Bash, Write

**Coordination:**
- Task, Skill (for loading best practices skills)

**Diagnostics:**
- mcp__ide__getDiagnostics

**NOT ALLOWED:**
- Edit (read-only, report findings only)
- Chrome DevTools (use /nick:verify for runtime testing)
- Context7/microsoft-docs (executor already verified)

**Key rules:**
- NEVER modify source files — report findings only
- Only commit AFTER review is written and result is PASS
- ASK user before running OWASP security review (expensive)
- ALWAYS load vercel-react-best-practices skill for every review

---

## Pitfalls & Prevention

### MCP Unavailability
**Symptom:** Skill loads but MCP calls fail

**Causes:**
1. MCP server not connected
2. Authentication issues (API keys expired, permissions missing)
3. OAuth tokens need refresh
4. Tool names incorrect (case-sensitive)

**Graceful Degradation Strategies:**

**For Planner:**
- If Context7 unavailable: Fall back to WebSearch for library docs (warn user about potential API staleness)
- If microsoft-docs unavailable: Use WebSearch for Azure docs (warn about potentially unofficial sources)
- If shadcn unavailable: Read local shadcn components from codebase if available

**For Executor:**
- Same as Planner (re-verification is MANDATORY, but can fall back to WebSearch)
- Document the fallback in build summary

**For Verifier:**
- If Chrome DevTools unavailable: Mark verification as "inconclusive" NOT "failure"
- Offer alternative: Visual verification (user verifies manually)
- Environment issues (app not running, MCP disconnected) = "inconclusive"

**For Reviewer:**
- If IDE diagnostics unavailable: Fall back to `npm run typecheck` via Bash
- Document the limitation in review

**Prevention:**
1. **Verify MCP connection before starting:**
   - Check Settings > Extensions > [Service]
   - Should show "Connected" status

2. **Test MCP independently:**
   - Ask Claude to call MCP directly (without skill)
   - If this fails, issue is MCP not skill

3. **Check tool names:**
   - Tool names are case-sensitive
   - Verify against MCP server documentation

**From claude-code-skill-guide.md:**
```markdown
#### MCP connection issues

**Symptom:** Skill loads but MCP calls fail

**Checklist:**

1. **Verify MCP server is connected**
   - Claude.ai: Settings > Extensions > [Your Service]
   - Should show "Connected" status

2. **Check authentication**
   - API keys valid and not expired
   - Proper permissions/scopes granted
   - OAuth tokens refreshed

3. **Test MCP independently**
   - Ask Claude to call MCP directly (without skill)
   - "Use [Service] MCP to fetch my projects"
   - If this fails, issue is MCP not skill

4. **Verify tool names**
   - Skill references correct MCP tool names
   - Check MCP server documentation
   - Tool names are case-sensitive
```

---

### Stale Documentation
**Problem:** Context7/microsoft-docs might have cached or outdated content

**When it matters:**
- Breaking API changes in recent library versions
- Deprecated patterns still showing in search results
- New features not yet indexed

**How to handle:**

**For Context7:**
- Always use resolve-library-id first (gets latest version mapping)
- Cross-reference with WebSearch if results seem outdated
- Check library's official changelog if API behavior differs from docs

**For microsoft-docs:**
- Use `microsoft_docs_fetch` for complete, up-to-date content (search returns chunks)
- Cross-reference official Microsoft Learn URLs
- Check Azure portal for latest UI/configurations

**Cache behavior:**
- Context7: Self-cleaning cache (not specified in docs)
- microsoft-docs: Likely crawls official sources regularly
- WebFetch: 15-minute cache (from claude-code-skill-guide.md)

**Prevention:**
1. Always include version context in queries ("Next.js 14 App Router" not just "Next.js")
2. Use follow-up pattern: search → fetch complete page
3. Document which library versions were verified in plan/build summary

---

### Token Overhead from MCP Calls
**Problem:** MCP calls consume tokens, especially with large responses

**When to lookup vs when to trust:**

**ALWAYS lookup (MANDATORY):**
- Library APIs (Context7) — libraries change, never trust memory
- Azure services (microsoft-docs) — configurations change
- shadcn component usage — exact patterns matter

**CAN skip lookup:**
- Standard JavaScript/TypeScript syntax (not library-specific)
- Well-established patterns (React basics that haven't changed)
- Project-specific code patterns (already in codebase)

**Token-saving strategies:**

**For Context7:**
- Respect 3-call limit per question
- Batch questions: "How to use useEffect, useState, and useCallback" instead of 3 separate calls
- Query specific APIs, not general concepts

**For microsoft-docs:**
- Use `head_limit` parameter to limit results
- Use language filter in code_sample_search to reduce noise
- Fetch only high-value pages identified by search

**For Chrome DevTools:**
- Use `includeSnapshot: false` when snapshot not needed
- Use `pageSize` parameter to limit list results
- Use `types` filter for console messages (error/warn only)

**From tool documentation:**
- Context7: "Do not call this tool more than 3 times per question"
- microsoft-docs search: "returns up to 10 high-quality content chunks (each max 500 tokens)"
- microsoft-docs fetch: "Maximum 20 pages per request" for PDFs

---

### Connection Timeouts
**Problem:** MCP calls can timeout or fail intermittently

**Handling strategies:**

**For Planner:**
1. If resolve-library-id times out: Retry once, then fall back to WebSearch
2. If query-docs times out: Try more specific query (narrower scope = faster)
3. Document fallback in plan: "Note: Context7 unavailable, verified via WebSearch"

**For Executor:**
1. Same as Planner (re-verification can use fallbacks)
2. Document in build summary: "Deviations from Plan: Used WebSearch instead of Context7 due to timeout"

**For Verifier:**
1. If Chrome DevTools times out: Mark test as "inconclusive" or "skipped"
2. Do NOT mark as "failure" — environment issues != code issues
3. Offer retry or alternative verification method

**For Reviewer:**
1. If IDE diagnostics times out: Fall back to `npm run typecheck`
2. Note the limitation in review

**From verify.md:**
```markdown
- Environment issues (app not running, MCP disconnected) = "inconclusive", not failure
```

**Prevention:**
1. Test MCP connection before running expensive workflows
2. Use timeout parameters where available (Chrome DevTools has timeout options)
3. Implement retry logic for transient failures

---

### Chrome DevTools Snapshot Staleness
**Problem:** Page state changes between snapshot and interaction

**Handling:**
1. ALWAYS take fresh snapshot before EVERY interaction
2. Never reuse UIDs from old snapshots
3. Use `wait_for` to wait for dynamic content before snapshot

**From verify.md:**
```markdown
3. Spawn a `browser-tester` agent (Haiku) with the complete test plan and instructions to execute every test step-by-step, use `take_snapshot` before every interaction, and compile a summary report.
```

**Pattern:**
```
1. take_snapshot → get UIDs
2. click/fill using UID from snapshot
3. take_snapshot → verify result, get new UIDs
4. Next interaction using new UIDs
```

**NOT:**
```
1. take_snapshot once
2. click/fill/click/fill using same UIDs (STALE!)
```

---

### Skill Over/Under-Triggering
**Not strictly an MCP issue, but affects MCP-enhanced skills**

**Under-triggering:**
- Skill doesn't load when it should
- Solution: Add more trigger phrases to description field

**Over-triggering:**
- Skill loads for irrelevant queries
- Solution: Add negative triggers, be more specific

**From claude-code-skill-guide.md:**
```yaml
# Good - includes trigger phrases
description: Analyzes Figma design files and generates developer handoff documentation. Use when user uploads .fig files, asks for "design specs", "component documentation", or "design-to-code handoff".

# Add negative triggers
description: Advanced data analysis for CSV files. Use for statistical modeling, regression, clustering. Do NOT use for simple data exploration (use data-viz skill instead).
```

---

## Recommendations

### For the Migration

**1. Tool Allowlist Philosophy:**
- **Planner:** Documentation MCPs (Context7, microsoft-docs, shadcn) + exploration (Read, Grep, Glob) + WebSearch fallback
- **Executor:** Same as Planner (re-verification) + implementation (Write, Edit) + Skill loading
- **Verifier:** Chrome DevTools (full suite) + Task coordination + NO Edit/Context7
- **Reviewer:** IDE diagnostics + Skill loading (best practices) + NO Edit/Chrome

**2. MCP Call Sequencing:**
- Context7: ALWAYS resolve → query (never skip resolve)
- microsoft-docs: ALWAYS search → code-sample → fetch (progressive detail)
- Chrome DevTools: ALWAYS snapshot → interact → verify (never skip snapshot)

**3. Graceful Degradation:**
- MCP unavailable → Fall back to WebSearch/Bash equivalents
- Document fallback in summary (plan/build/review)
- Environment issues → "inconclusive" NOT "failure"

**4. Token Management:**
- Respect call limits (Context7: 3 calls per question)
- Use filters (microsoft-docs language filter, Chrome DevTools types filter)
- Batch queries when possible
- Use head_limit/pageSize to reduce response size

**5. Error Handling:**
- Test MCP connection before expensive workflows
- Implement retry for transient failures
- Mark environment issues separately from code issues
- Provide clear error messages with remediation steps

**6. Documentation Requirements:**
- Plans MUST document which library versions were verified
- Build summaries MUST note any API deviations or fallbacks
- Reviews MUST note if diagnostics unavailable (and what was used instead)
- Verification reports MUST distinguish "failed" vs "inconclusive"

**7. Mandatory MCP Usage:**
- Context7: MANDATORY for any library/framework code (never code from memory)
- microsoft-docs: MANDATORY for Azure features
- IDE diagnostics: MANDATORY for reviews (fallback to npm run typecheck if unavailable)
- Chrome DevTools: MANDATORY for verification (fallback to visual verification if unavailable)

**8. Optional MCP Usage:**
- shadcn: Use when implementing shadcn components, skip otherwise
- Figma: Specialized use case (design-to-code), not in current workflow
- Mermaid: Specialized use case (diagrams), not in current workflow

**9. Multi-Agent Patterns:**
- Verifier spawns test-planner (Sonnet) + browser-tester (Haiku)
- Test-planner reads workflow, produces structured plan
- Browser-tester executes plan with snapshot → interact → verify pattern
- Both agents collect and return results to verifier

**10. Best Practices Skills:**
- Executor: Load vercel-react-best-practices, nestjs-best-practices, frontend-design, owasp-security BEFORE writing code
- Reviewer: ALWAYS load vercel-react-best-practices (primary check), conditionally load others

---

## Critical Rules Summary

**NEVER:**
- Skip Context7 resolve-library-id before query-docs
- Skip snapshot before Chrome DevTools interaction
- Modify source files from verifier role
- Commit from executor role (reviewer commits)
- Assume current API without verification (libraries change)

**ALWAYS:**
- Verify library APIs via Context7 before coding
- Use microsoft-docs for Azure implementations
- Take fresh snapshot before EVERY interaction
- Check IDE diagnostics in reviews
- Document fallbacks/deviations in summaries
- Distinguish "failed" vs "inconclusive" in verification

**ASK USER:**
- Before generating new workflow (expensive: 5 explorers + composer)
- Before running OWASP security review (expensive)
- When MCP unavailable: offer fallback or alternative path

---

## References

**Source files analyzed:**
- /Users/nick/Documents/git/get-shit-done/docs/nick/plan.md
- /Users/nick/Documents/git/get-shit-done/docs/nick/execute.md
- /Users/nick/Documents/git/get-shit-done/docs/nick/verify.md
- /Users/nick/Documents/git/get-shit-done/docs/nick/review.md
- /Users/nick/Documents/git/get-shit-done/docs/claude-code-skill-guide.md

**Key patterns extracted:**
1. Context7 two-step: resolve → query (from plan.md, execute.md)
2. Chrome DevTools flow: snapshot → interact → verify (from verify.md)
3. microsoft-docs three-step: search → code-sample → fetch (from plan.md)
4. Multi-agent verification: test-planner + browser-tester (from verify.md)
5. Graceful degradation: MCP fail → fallback → document (from verify.md, claude-code-skill-guide.md)
