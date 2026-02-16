# Phase 7: Prototype-Driven Design in Plan Phase and Mandatory Frontend Verification After Execution - Research

**Researched:** 2026-02-16
**Domain:** Workflow skill modification (plan-phase planner + execute-phase orchestrator + verify-work integration)
**Confidence:** HIGH

## Summary

Phase 7 adds two capabilities to the existing GSD workflow pipeline: (1) the planner agent produces visual prototype artifacts during planning that guide executor implementation, and (2) the execute-phase orchestrator automatically triggers runtime verification for phases that contain frontend work rather than leaving it as an optional manual step.

This phase modifies existing skill files rather than creating new ones. The planner agent (`nick-planner.md`) gains a new prototype generation step that produces HTML mockup files alongside PLAN.md files. The execute-phase orchestrator (`nick-execute-phase/SKILL.md`) gains frontend detection logic and an automatic verify-work trigger after wave execution completes. The verify-work skill (`nick-verify-work/SKILL.md`) needs no changes -- it already handles both UI and non-UI phases correctly.

**Primary recommendation:** Keep prototypes as static HTML/CSS files in the phase directory (not React components or design tools). Detect frontend work via plan frontmatter metadata rather than file-extension heuristics. Auto-trigger verify-work from execute-phase orchestrator only when `has_frontend: true` is set in plan frontmatter.

## Standard Stack

### Core

This phase modifies existing markdown skill files. No new libraries or tools are introduced.

| Component | Location | Purpose | Why It Stays |
|-----------|----------|---------|--------------|
| nick-planner.md | `~/.claude/skills/nick-plan-phase/references/agents/nick-planner.md` | Planner agent -- gains prototype step | Core planning logic lives here |
| nick-execute-phase SKILL.md | `~/.claude/skills/nick-execute-phase/SKILL.md` | Execute orchestrator -- gains auto-verify trigger | Controls post-execution flow |
| nick-plan-checker.md | `~/.claude/skills/nick-plan-phase/references/agents/nick-plan-checker.md` | Plan checker -- gains prototype validation dimension | Validates plan quality |
| nick-executor.md | `~/.claude/skills/nick-execute-phase/references/agents/nick-executor.md` | Executor agent -- gains prototype reference loading | Implements from prototypes |
| PLAN.md format | Planner output | Gains `has_frontend` and `prototype` frontmatter fields | Carries metadata to executor and orchestrator |

### Supporting

| Component | Location | Purpose | When Needed |
|-----------|----------|---------|-------------|
| nick-verify-work SKILL.md | `~/.claude/skills/nick-verify-work/SKILL.md` | Already complete -- no changes needed | Referenced but not modified |
| nick-browser-tester.md | `~/.claude/skills/nick-verify-work/references/agents/nick-browser-tester.md` | Already handles snapshot-interact-verify | Referenced but not modified |
| nick-phase-researcher.md | `~/.claude/skills/nick-plan-phase/references/agents/nick-phase-researcher.md` | Researcher agent -- no changes needed | Research runs before planner |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Static HTML prototypes | React/JSX prototype components | React requires build tooling, adds complexity; HTML is universally renderable and cheaper for the planner to produce |
| Static HTML prototypes | Figma/design tool integration | No MCP for Figma exists in the current stack; HTML stays within Claude's capabilities |
| Static HTML prototypes | ASCII wireframes in markdown | ASCII wireframes lack visual fidelity; cannot be opened in a browser for comparison during verify-work |
| `has_frontend` frontmatter flag | File extension scanning (.tsx/.jsx/.html detection) | Extension scanning is fragile and happens too late (at execution time); frontmatter flag is explicit and available during planning |
| Auto-trigger verify-work | Manual verify-work invocation | Current behavior -- user must remember to run verify-work; automation ensures frontend work is always tested |
| Phase-level auto-verify | Per-plan auto-verify | Per-plan is too granular -- verification makes more sense after all plans in a phase execute since features may span plans |

## Architecture Patterns

### Pattern 1: Prototype-Driven Planning

**What:** The planner agent creates a static HTML/CSS prototype file for each plan that involves frontend work. The prototype is a single-file HTML document with inline CSS that shows the expected layout, structure, and key interactive elements. It lives alongside the PLAN.md file.

**When to use:** Any plan where `has_frontend: true` in frontmatter (plans that create/modify .tsx, .jsx, .html files, or involve UI routes).

**File structure:**
```
.planning/phases/XX-name/
  XX-01-PLAN.md           # Plan file (existing)
  XX-01-PROTOTYPE.html    # NEW: Visual prototype for this plan
  XX-02-PLAN.md           # Plan file (no frontend work)
  XX-03-PLAN.md           # Plan file (frontend work)
  XX-03-PROTOTYPE.html    # NEW: Visual prototype for this plan
```

**Prototype file naming:** `{phase}-{plan}-PROTOTYPE.html`

**Prototype scope:** Each prototype shows ONE plan's visual output. If a plan creates a dashboard page, the prototype shows the dashboard layout with placeholder data. If a plan creates a form, the prototype shows the form fields, labels, and button states.

**What prototypes include:**
- Layout structure (grid, flex, spacing)
- Component hierarchy (what contains what)
- Placeholder data (realistic fake content, not "Lorem ipsum")
- Interactive element states (buttons, inputs, empty states, loading states, error states)
- Responsive breakpoints if relevant (desktop + mobile widths)
- Color scheme and typography from the project's design system (if one exists -- check for tailwind.config, theme files)

**What prototypes do NOT include:**
- Actual JavaScript behavior (no event handlers, no fetch calls)
- Framework-specific code (no React, no Vue -- pure HTML/CSS)
- Backend integration (no real API calls)
- Animation or transition details (describe in plan action text instead)

**Why static HTML:**
- The planner agent can produce it directly (Write tool)
- The executor agent can open it in a browser via Chrome DevTools for reference
- The browser-tester agent can compare the prototype layout to the implemented layout
- No build step, no dependencies, no framework coupling
- If using shadcn/Tailwind, the planner references the actual component docs during prototype creation but renders the visual intent in plain CSS

### Pattern 2: Frontend Detection via Plan Frontmatter

**What:** Plans declare `has_frontend: true` or `has_frontend: false` in their YAML frontmatter. The planner sets this flag based on whether the plan's tasks involve UI files.

**Detection heuristic for the planner:**
```
has_frontend = true if ANY of:
  - files_modified contains .tsx, .jsx, .html, .css, .scss
  - files_modified contains paths matching: src/app/**/page.tsx, src/components/**, pages/**
  - tasks involve UI components, layouts, pages, or visual styling
  - objective mentions "UI", "page", "component", "form", "dashboard", "layout"

has_frontend = false if:
  - all files are .ts, .js, .json, .md, .yml, .yaml
  - work is pure backend, CLI, config, documentation, or infrastructure
```

**Frontmatter addition to PLAN.md:**
```yaml
---
phase: XX-name
plan: NN
type: execute
wave: N
depends_on: []
files_modified: []
autonomous: true
has_frontend: true          # NEW field
prototype: XX-NN-PROTOTYPE.html  # NEW field (only if has_frontend: true)
must_haves:
  truths: []
  artifacts: []
  key_links: []
---
```

**Why frontmatter over runtime detection:**
- Available at plan time, not just execution time
- Explicit -- no ambiguity about whether a plan is frontend work
- The plan checker can validate it (prototype exists when has_frontend: true)
- The orchestrator reads it without parsing file lists at runtime

### Pattern 3: Auto-Triggered Verification After Frontend Execution

**What:** The execute-phase orchestrator automatically spawns the verify-work skill after all waves complete if ANY plan in the phase has `has_frontend: true`.

**Integration point:** Between Step 5 (Verify Phase Goal) and Step 6 (Update Roadmap) in the execute-phase orchestrator.

**Flow:**
```
Step 4: Execute Waves (existing)
  |
Step 4.5: Auto-Verify Frontend (NEW)
  |-- Check: any plan has_frontend: true?
  |   |-- YES: Spawn verify-work with phase number
  |   |   |-- Read RUNTIME-VERIFICATION.md result
  |   |   |-- If "passed": continue
  |   |   |-- If "issues_found": present to user, offer fix or continue
  |   |   |-- If "inconclusive": warn user (Chrome DevTools not available), continue
  |   |-- NO: Skip (log "No frontend work detected, skipping runtime verification")
  |
Step 5: Verify Phase Goal (existing)
```

**Why between steps 4 and 5:**
- Runtime verification tests the running app BEFORE the static goal-backward verification
- Issues found in runtime can inform gap closure before the phase is marked "complete"
- If runtime verification finds issues, the user can address them before the phase goal is assessed

**Orchestrator code change (pseudocode for the SKILL.md):**
```
## Step 4.5: Auto-Verify Frontend (conditional)

Check if any plan in the phase has `has_frontend: true`:
```bash
HAS_FRONTEND=$(grep -l "has_frontend: true" "$PHASE_DIR"/*-PLAN.md 2>/dev/null | wc -l)
```

If HAS_FRONTEND > 0:

```
Task(
  prompt="First, read ~/.claude/skills/nick-verify-work/SKILL.md for the verification workflow.

  <objective>
  Verify frontend work for phase {phase_number}-{phase_name}.
  Phase directory: {phase_dir}

  This is an auto-triggered verification. The executor completed frontend-affecting plans.
  Run the standard verify-work flow.
  </objective>",
  subagent_type="general-purpose"
)
```

Read RUNTIME-VERIFICATION.md status:
- **passed:** Log "Runtime verification passed." Continue to Step 5.
- **issues_found:** Present issue summary. Offer: "Fix issues before phase goal verification?" (yes/no). If yes, halt. If no, continue with note.
- **inconclusive:** Log "Runtime verification inconclusive (Chrome DevTools unavailable). Manual verification recommended." Continue to Step 5.

If HAS_FRONTEND = 0:
Log "No frontend plans detected. Skipping runtime verification."
```

### Pattern 4: Prototype-Guided Execution

**What:** The executor agent reads the prototype HTML file before implementing frontend tasks. The prototype serves as a visual specification alongside the PLAN.md textual specification.

**Executor behavior change:**
1. Before implementing a task with frontend files, check if `prototype` field exists in plan frontmatter
2. If yes, read the prototype file to understand expected layout, component structure, and visual design
3. Use the prototype as the "design spec" -- the implemented component should match the prototype's structure
4. In SUMMARY.md, note: "Implemented from prototype: {prototype-file}"

**Executor does NOT:**
- Copy the prototype HTML directly into the implementation (it is plain HTML, the implementation will be React/framework-specific)
- Modify the prototype file
- Use the prototype for non-visual aspects (API contracts, state management, data flow)

### Anti-Patterns to Avoid

- **Prototype scope creep:** Prototypes should be quick mockups (15-30 min), not pixel-perfect designs. The planner produces them as part of planning, not as a separate design phase.
- **Prototype as implementation:** The prototype is a guide, not a template. Executors should build proper React components, not wrap prototype HTML in JSX.
- **Over-detection:** Only detect frontend work through explicit frontmatter, not by scanning for CSS classes or utility imports that might appear in config files.
- **Verification blocking:** Auto-verification should NOT block phase completion if Chrome DevTools is unavailable. Inconclusive is acceptable; failed is actionable.
- **Double verification:** The auto-verify runs the SAME verify-work skill. Do not create a second verification path. If the user also manually runs `/gsd:verify-work`, the existing re-verification detection (Step 0 in nick-verifier.md) handles it.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Frontend detection | Custom file-type scanner | `has_frontend` frontmatter flag | Explicit metadata is reliable; file scanning is fragile |
| Prototype rendering | Custom preview server | Open HTML file directly in browser (`file://` or `python -m http.server`) | No infrastructure needed for static HTML |
| Visual comparison | Custom screenshot diffing tool | Browser-tester's existing snapshot-interact-verify cycle | Already built in Phase 3, handles DOM inspection and visual verification |
| Design tokens | Custom theme extraction | Read existing tailwind.config.ts or CSS variables | Projects already have design systems; don't duplicate |

**Key insight:** This phase leverages existing infrastructure (verify-work skill, browser-tester agent, Chrome DevTools MCP) rather than building new tools. The changes are primarily prompt/workflow modifications to existing agent definitions.

## Common Pitfalls

### Pitfall 1: Prototypes That Are Too Detailed

**What goes wrong:** The planner spends excessive context creating pixel-perfect prototypes with complex CSS animations, responsive edge cases, and micro-interactions. This consumes context budget meant for actual plan creation.
**Why it happens:** No explicit scope constraint on prototype complexity.
**How to avoid:** Define prototype scope: layout, component structure, key states. Cap prototype size at ~100-150 lines of HTML. The planner should spend 10-15% of context on prototypes, not 30-40%.
**Warning signs:** Prototype HTML exceeds 200 lines. Planner context usage approaches 70% before finishing plans.

### Pitfall 2: Frontend Detection False Positives

**What goes wrong:** A plan modifies a `.css` file for backend tooling configuration, or creates a `.tsx` file that is a server component with no visual output. Auto-verification runs unnecessarily and reports "inconclusive" or tests non-visual elements.
**Why it happens:** Filename-based detection is inherently imprecise.
**How to avoid:** Use explicit `has_frontend: true/false` in frontmatter rather than file extension detection. The planner understands intent; the orchestrator should trust the planner's classification.
**Warning signs:** RUNTIME-VERIFICATION.md consistently reports "inconclusive" or "N/A -- no runtime component" for phases the orchestrator thought had frontend work.

### Pitfall 3: Verify-Work Timing Issues

**What goes wrong:** Auto-verification runs immediately after execution, but the dev server is not running or has not rebuilt. Chrome DevTools reports "inconclusive" because the app is not available.
**Why it happens:** The execute-phase orchestrator spawns verify-work without checking if the app is running.
**How to avoid:** The verify-work skill already has a pre-flight check (Step 3: `list_pages`). If Chrome DevTools is not connected, it reports "inconclusive" gracefully. The orchestrator should handle "inconclusive" as a non-blocking warning, not a failure. Optionally, the orchestrator could prompt the user: "Frontend verification ready. Start your dev server and press Enter, or type 'skip' to continue."
**Warning signs:** Every auto-verification returns "inconclusive" because no dev server is running.

### Pitfall 4: Breaking Existing Non-Frontend Workflow

**What goes wrong:** Adding `has_frontend` as a required frontmatter field causes plan validation to fail for all existing plans and all non-frontend plans.
**Why it happens:** The frontmatter validator (`gsd-tools.cjs frontmatter validate --schema plan`) enforces required fields.
**How to avoid:** Make `has_frontend` OPTIONAL with default `false`. Do NOT add it to the required fields list. When absent, treat as `has_frontend: false`. Only the planner needs to set it explicitly when creating plans for frontend work.
**Warning signs:** Existing plan validation fails after the change. Plan checker reports false issues on non-frontend plans.

### Pitfall 5: Prototype Files Left Orphaned

**What goes wrong:** Prototypes are created but never referenced by executors or testers because the linkage is not explicit.
**Why it happens:** The prototype filename convention exists but agents do not know to look for it.
**How to avoid:** The `prototype` field in plan frontmatter provides the explicit link. The executor reads it. The browser-tester could optionally compare prototype layout to actual layout (future enhancement, not required for this phase).
**Warning signs:** SUMMARY.md files never mention prototypes. Executor does not reference prototype in implementation.

## Code Examples

### Example 1: Prototype HTML File

```html
<!-- .planning/phases/08-dashboard/08-01-PROTOTYPE.html -->
<!-- Prototype for Plan 08-01: Dashboard Layout -->
<!-- Purpose: Show card grid, metric widgets, and navigation structure -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Dashboard - Prototype</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: system-ui, sans-serif; background: #f8f9fa; color: #1a1a2e; }
    .container { max-width: 1200px; margin: 0 auto; padding: 24px; }
    .header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 24px; }
    .header h1 { font-size: 24px; font-weight: 600; }
    .metrics { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; margin-bottom: 32px; }
    .metric-card { background: white; border-radius: 8px; padding: 20px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
    .metric-card .label { font-size: 14px; color: #6b7280; margin-bottom: 4px; }
    .metric-card .value { font-size: 28px; font-weight: 700; }
    .metric-card .change { font-size: 12px; color: #10b981; margin-top: 4px; }
    .content-grid { display: grid; grid-template-columns: 2fr 1fr; gap: 24px; }
    .card { background: white; border-radius: 8px; padding: 20px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
    .card h2 { font-size: 16px; font-weight: 600; margin-bottom: 16px; }
    .list-item { padding: 12px 0; border-bottom: 1px solid #f3f4f6; display: flex; justify-content: space-between; }
    .list-item:last-child { border-bottom: none; }
    .badge { font-size: 12px; padding: 2px 8px; border-radius: 12px; background: #dbeafe; color: #1d4ed8; }
    /* Mobile: single column */
    @media (max-width: 768px) {
      .metrics { grid-template-columns: 1fr; }
      .content-grid { grid-template-columns: 1fr; }
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>Dashboard</h1>
      <button style="padding:8px 16px; border-radius:6px; background:#2563eb; color:white; border:none; cursor:pointer;">New Project</button>
    </div>
    <div class="metrics">
      <div class="metric-card">
        <div class="label">Total Projects</div>
        <div class="value">24</div>
        <div class="change">+3 this week</div>
      </div>
      <div class="metric-card">
        <div class="label">Active Tasks</div>
        <div class="value">142</div>
        <div class="change">+12 this week</div>
      </div>
      <div class="metric-card">
        <div class="label">Completion Rate</div>
        <div class="value">87%</div>
        <div class="change">+2% this month</div>
      </div>
    </div>
    <div class="content-grid">
      <div class="card">
        <h2>Recent Activity</h2>
        <div class="list-item"><span>Updated project settings</span><span class="badge">2h ago</span></div>
        <div class="list-item"><span>Completed auth integration</span><span class="badge">5h ago</span></div>
        <div class="list-item"><span>Added new team member</span><span class="badge">1d ago</span></div>
        <div class="list-item"><span>Deployed to production</span><span class="badge">2d ago</span></div>
      </div>
      <div class="card">
        <h2>Quick Actions</h2>
        <div class="list-item"><span>Create project</span></div>
        <div class="list-item"><span>Invite team member</span></div>
        <div class="list-item"><span>View reports</span></div>
      </div>
    </div>
  </div>
</body>
</html>
```

### Example 2: Plan Frontmatter with Frontend Fields

```yaml
---
phase: 08-dashboard
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - src/app/dashboard/page.tsx
  - src/components/MetricCard.tsx
  - src/components/ActivityFeed.tsx
autonomous: true
has_frontend: true
prototype: 08-01-PROTOTYPE.html

must_haves:
  truths:
    - "User sees metric cards with project count, active tasks, and completion rate"
    - "User sees recent activity list with timestamps"
    - "Dashboard is responsive (single column on mobile)"
  artifacts:
    - path: "src/app/dashboard/page.tsx"
      provides: "Dashboard page layout"
      min_lines: 40
    - path: "src/components/MetricCard.tsx"
      provides: "Reusable metric display card"
      min_lines: 20
  key_links:
    - from: "src/app/dashboard/page.tsx"
      to: "/api/metrics"
      via: "fetch in useEffect or server component"
      pattern: "fetch.*api/metrics"
---
```

### Example 3: Non-Frontend Plan (No Prototype)

```yaml
---
phase: 08-dashboard
plan: 02
type: execute
wave: 1
depends_on: []
files_modified:
  - src/app/api/metrics/route.ts
  - src/lib/metrics.ts
autonomous: true
has_frontend: false

must_haves:
  truths:
    - "GET /api/metrics returns JSON with projectCount, activeTasks, completionRate"
  artifacts:
    - path: "src/app/api/metrics/route.ts"
      provides: "Metrics API endpoint"
      min_lines: 20
  key_links:
    - from: "src/app/api/metrics/route.ts"
      to: "prisma.project"
      via: "database query"
      pattern: "prisma\\.project\\.(find|count)"
---
```

### Example 4: Execute-Phase Orchestrator Auto-Verify Step

```markdown
## Step 4.5: Auto-Verify Frontend (conditional)

After all waves complete, check if any executed plan has frontend work:

\`\`\`bash
HAS_FRONTEND=$(grep -l "has_frontend: true" "$PHASE_DIR"/*-PLAN.md 2>/dev/null | wc -l)
\`\`\`

**If HAS_FRONTEND > 0:**

Inform user: "Frontend work detected. Running runtime verification..."

Spawn verify-work:

\`\`\`
Task(
  prompt="First, read ~/.claude/skills/nick-verify-work/SKILL.md for the verification workflow.

  <objective>
  Auto-verify frontend work for phase {phase_number}-{phase_name}.
  Phase directory: {phase_dir}
  </objective>

  <files_to_read>
  - {phase_dir}/*-SUMMARY.md
  - {phase_dir}/*-PLAN.md
  </files_to_read>

  <output>Create {phase_dir}/{padded_phase}-RUNTIME-VERIFICATION.md</output>",
  subagent_type="general-purpose"
)
\`\`\`

Handle result from RUNTIME-VERIFICATION.md frontmatter `status`:
- **passed:** "Runtime verification passed." Continue to Step 5.
- **issues_found:** Display issues. "Frontend issues found. Address before proceeding? (yes to halt / no to continue)"
- **inconclusive:** "Runtime verification skipped (Chrome DevTools unavailable or app not running). Consider running `/gsd:verify-work {phase}` manually." Continue to Step 5.

**If HAS_FRONTEND = 0:**
Skip. "No frontend plans in this phase. Skipping runtime verification."
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Plans are text-only (no visual spec) | Plans with prototype HTML files | Phase 7 (this phase) | Executors get visual guidance, reducing interpretation errors |
| Verify-work is always manual | Auto-triggered for frontend phases | Phase 7 (this phase) | Frontend work always gets runtime testing |
| Non-UI phases get "inconclusive" | Non-UI phases skip verify entirely | Phase 7 (this phase) | Cleaner flow -- only relevant phases trigger verification |

**Deprecated/outdated:**
- Nothing deprecated. This phase extends existing patterns rather than replacing them.

## Open Questions

1. **Should the browser-tester compare prototype to implementation?**
   - What we know: The browser-tester takes DOM snapshots. A prototype HTML file also has a DOM structure. In theory, a structural comparison could be done.
   - What's unclear: How to compare meaningfully. The prototype uses plain CSS; the implementation uses Tailwind/shadcn classes. DOM structure will differ (prototype uses `<div class="card">`, implementation uses `<Card>` component).
   - Recommendation: Defer for now. The prototype serves as a human-readable spec and executor guide. Visual regression testing against prototypes is a future enhancement. For Phase 7, the prototype is an input to the executor, not a verification target.

2. **Should the planner use shadcn MCP to build prototypes?**
   - What we know: The plan-phase already has shadcn MCP access (Phase 2). Component specs could inform prototype structure.
   - What's unclear: Whether shadcn component HTML translates well to static prototypes. Shadcn components use React + Tailwind; prototypes are plain HTML/CSS.
   - Recommendation: The planner should query shadcn MCP for component NAMES and PROPS during research/planning (already done), then use that knowledge to structure the prototype. But the prototype itself should remain plain HTML -- do not attempt to render actual shadcn components in the prototype.

3. **Should `has_frontend` be auto-derived or manually set?**
   - What we know: The planner creates plans and knows whether they are frontend work. The `files_modified` field already lists file paths.
   - What's unclear: Whether to require the planner to explicitly set `has_frontend` or to derive it from `files_modified` patterns.
   - Recommendation: Have the planner set it explicitly. Add a self-check step: "If files_modified contains .tsx/.jsx/.html files, has_frontend SHOULD be true." This gives the planner override capability (e.g., a .tsx file that is a server-only component) while providing a safety net.

4. **What happens when auto-verify returns "inconclusive" repeatedly?**
   - What we know: If the user never runs a dev server, every auto-verify will be inconclusive.
   - What's unclear: Whether to suppress repeated inconclusive warnings or always show them.
   - Recommendation: Always show the warning but make it brief: "Runtime verification skipped (no dev server detected). Run `/gsd:verify-work {phase}` when ready." Do not block or nag.

5. **Should prototype creation consume plan context budget?**
   - What we know: The planner targets ~50% context. Adding prototype creation adds work.
   - What's unclear: How much context prototypes consume.
   - Recommendation: Set prototype budget at 10-15% of planner context. Cap prototype size at 150 lines. If a plan has many visual elements, create a simpler prototype showing layout structure rather than every detail. The prototype is a guide, not a spec.

## Modification Surface

### Files That Need Changes

| File | Change Type | What Changes |
|------|-------------|-------------|
| `nick-planner.md` (repo + installed) | Major addition | New `<prototype_creation>` section with rules, format, and integration into planning flow |
| `nick-plan-checker.md` (repo + installed) | Minor addition | New verification dimension: "Prototype Consistency" -- has_frontend=true plans must have prototype file |
| `nick-executor.md` (repo + installed) | Minor addition | Step to read prototype file before frontend tasks |
| `nick-execute-phase/SKILL.md` (repo + installed) | Medium addition | New Step 4.5 for auto-verify frontend |
| PLAN.md format (in nick-planner.md) | Minor addition | `has_frontend` and `prototype` frontmatter fields |
| `gsd-tools.cjs` (frontmatter validate) | Minor addition | Accept `has_frontend` and `prototype` as valid optional fields in plan schema |

### Files That Do NOT Change

| File | Why No Change |
|------|---------------|
| `nick-verify-work/SKILL.md` | Already handles UI and non-UI phases correctly |
| `nick-browser-tester.md` | Already has snapshot-interact-verify cycle |
| `nick-phase-researcher.md` | Research runs before planning; no prototype involvement |
| `nick-review/SKILL.md` | Code review is separate from runtime verification |
| `nick-verifier.md` | Goal-backward verification is separate from runtime verification |
| `nick-executor-checkpoints.md` | Checkpoint protocol unchanged |
| `nick-executor-tdd.md` | TDD protocol unchanged |

### Dual Location Pattern

All modified skill files exist in TWO locations (established in Phase 5):
1. **Repo source:** `skills/` directory in the project (version control)
2. **Installed location:** `~/.claude/skills/` (where Claude Code loads them)

Both must be updated identically. Use the same copy-from-repo-to-installed pattern from Phase 5.

## Sources

### Primary (HIGH confidence)

- Direct file reads of all skill files in `~/.claude/skills/nick-*` -- current production state of all agents
- Direct file read of `.planning/ROADMAP.md` -- phase description and dependencies
- Direct file read of `.planning/STATE.md` -- accumulated decisions from phases 1-6
- Direct file read of `.planning/config.json` -- workflow configuration

### Secondary (MEDIUM confidence)

- Architectural patterns derived from analysis of the existing 6-phase evolution -- patterns are well-established and consistent
- HTML prototype format is a standard practice in web development (no external source needed -- this is basic HTML/CSS)

### Tertiary (LOW confidence)

- Prototype context budget estimate (10-15%) -- based on reasoning about planner behavior, not measured data. Needs validation during execution.
- Prototype line count cap (150 lines) -- reasonable heuristic, may need adjustment for complex UIs.

## Metadata

**Confidence breakdown:**
- Modification surface: HIGH -- all files read directly, change points identified precisely
- Architecture patterns: HIGH -- extending established patterns from phases 1-6, no new abstractions
- Pitfalls: MEDIUM -- based on reasoning about edge cases, not observed failures
- Prototype format: MEDIUM -- reasonable design decision but untested in this workflow; may need iteration

**Research date:** 2026-02-16
**Valid until:** 2026-03-16 (30 days -- stable domain, all changes are to local workflow files)
