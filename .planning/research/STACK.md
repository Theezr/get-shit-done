# Skills Architecture & Patterns Research

## Executive Summary

This research analyzes the Claude Code skills architecture based on the comprehensive skill guide and 7 production-ready skill prototypes. The findings reveal a sophisticated pattern-based system optimized for progressive disclosure, clear role separation, composability, and tool restrictions aligned with skill responsibilities.

**Key Finding:** The skill system succeeds by treating skills as specialized agents with bounded contexts, not general-purpose instructions. The strongest skills have narrow, well-defined roles with explicit tool allowlists that prevent scope creep.

## Skill Structure

### File Organization Pattern

All skills follow this canonical structure:

```
skill-name/
├── SKILL.md              # Required - frontmatter + instructions
├── scripts/              # Optional - executable code
├── references/           # Optional - documentation loaded as needed
└── assets/               # Optional - templates, fonts, icons
```

### SKILL.md Anatomy

Every SKILL.md has two distinct parts:

1. **YAML Frontmatter** (always loaded in system prompt)
2. **Markdown Body** (loaded when skill is invoked)

### Critical Naming Rules

From the guide:
- SKILL.md must be EXACTLY `SKILL.md` (case-sensitive, no variations)
- Folder names MUST use kebab-case: `notion-project-setup` ✅
- NO spaces, underscores, or capitals in folder names
- NO README.md inside skill folders (documentation goes in SKILL.md or references/)

### Progressive Disclosure in Practice

The guide defines three levels:

**Level 1 (Frontmatter):** Always in context. Must be minimal but sufficient for Claude to know when to load the skill.

**Level 2 (SKILL.md body):** Loaded when skill is triggered. Contains full instructions, workflows, examples.

**Level 3 (Linked files):** references/, scripts/, assets/ - loaded only when explicitly needed.

**Observed Pattern from Prototypes:** Skills keep SKILL.md under 200 lines by moving detailed documentation to references/ and linking to it contextually.

## YAML Frontmatter Patterns

### Required Fields

```yaml
---
name: skill-name-in-kebab-case
description: What it does and when to use it. Include specific trigger phrases.
---
```

### Extended Frontmatter Pattern (from prototypes)

```yaml
---
name: nick:plan
description: Plan a feature — explore codebase, look up docs, write implementation plan
argument-hint: "<task-description or path-to-task-file>"
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
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
  - mcp__microsoft-docs__microsoft_docs_search
  - mcp__microsoft-docs__microsoft_code_sample_search
  - mcp__microsoft-docs__microsoft_docs_fetch
  - mcp__shadcn__getComponents
  - mcp__shadcn__getComponent
---
```

### Description Field Best Practices

**Effective descriptions** follow this pattern:
```
[Action verb] + [specific task] + [trigger conditions/phrases]
```

**Good examples from prototypes:**
- `Plan a feature — explore codebase, look up docs, write implementation plan`
- `Execute a plan — implement code from a planner-created plan file`
- `Verify a build — plan workflow tests, execute them with Chrome DevTools`
- `Review code quality, security, and best practices for a build`

**What makes these work:**
1. Clear action verb (Plan, Execute, Verify, Review)
2. Specific task scope (a feature, a plan, a build)
3. Method hints (explore codebase, look up docs)
4. Implicit trigger (user asks to plan/execute/verify/review)

### argument-hint Pattern

All prototypes use `argument-hint` to guide users:
```yaml
argument-hint: "<feature-slug>"
argument-hint: "<task-description or path-to-task-file>"
argument-hint: "path/to/workflow-guide.md"
argument-hint: "[feature description or path hint]"
```

This is NOT in the official guide but appears to be a custom extension for workflow skills.

### allowed-tools Pattern

**Critical finding:** Tool restrictions are THE primary mechanism for enforcing skill boundaries.

From prototypes:
- **Planner (research-only):** Read, Glob, Grep, Bash, Write, Task, WebSearch, WebFetch, MCP docs, EnterPlanMode, ExitPlanMode — NO Edit, NO Skill
- **Executor (implementation):** Read, Write, Edit, Glob, Grep, Bash, Task, WebSearch, WebFetch, Skill, MCP docs — YES Edit, YES Skill
- **Verifier (runtime testing):** Read, Glob, Grep, Bash, Write, Task, chrome-devtools tools — NO Edit, NO source modification
- **Reviewer (code quality):** Read, Glob, Grep, Bash, Write, Task, Skill, mcp__ide__getDiagnostics — NO Edit, reads only

**Key insight:** The allowed-tools list is a contract. It defines what the skill CAN do, which directly shapes what it SHOULD do.

## Progressive Disclosure

### What Goes Where

**Frontmatter (always loaded):**
- name, description, argument-hint
- allowed-tools (tool restriction contract)
- metadata for discovery (author, version, mcp-server)

**SKILL.md body (loaded on invocation):**
- Role definition ("You are the Planner")
- Phase-based workflows with numbered steps
- Mandatory vs optional actions
- Research requirements
- Output format templates
- Rules and constraints

**references/ (loaded on demand):**
- API documentation
- Extended examples
- Troubleshooting guides
- Domain-specific knowledge

### Pattern from Prototypes: Phase-Based Structure

All workflow skills use this pattern:

```markdown
## Phase 1: [Preparation]
### Step 1 — [Atomic action]
### Step 2 — [Atomic action]

## Phase 2: [Main Work]
### Step X — [Action] (MANDATORY)
...

## Phase 3: [Finalization]
...
```

**Why this works:**
- Phases group related steps
- Steps are atomic and numbered
- MANDATORY tags highlight non-optional actions
- Clear progression from start to finish

### Pattern: Template Injection

Multiple prototypes include complete document templates within the skill:

```markdown
### Step 6 — Write the plan in the plan mode file

Compose the full plan using this template in the plan mode file:

\`\`\`markdown
# Plan: {Feature Name}

## Summary
One paragraph — what this does and why.

## Requirements
- What must be true when this is done
...
\`\`\`
```

This ensures consistent output structure across invocations.

## Composability

### Skill-to-Skill Invocation Pattern

**Executor loads domain skills based on code type:**

```markdown
### Skills — MANDATORY for relevant code

**React/Next.js code:** Load the `vercel-react-best-practices` skill BEFORE
writing any React component, hook, or Next.js page. Follow every pattern.

**NestJS code:** Load the `nestjs-best-practices` skill BEFORE writing any
NestJS controller, service, module, guard, pipe, or interceptor.

**UI components:** Load the `frontend-design` skill BEFORE writing any visual
component, page, or layout. Every UI element must follow the design system.

**Security-sensitive code:** Load the `owasp-security` skill when implementing
auth, input handling, API endpoints, or anything touching user data.
```

**Key insight:** The Executor skill orchestrates other skills via the `Skill` tool. It doesn't need to know HOW to write good React code — it knows WHEN to delegate to the React skill.

### Agent Composition Pattern

**Verifier and Generate-Workflow use Task tool for agent delegation:**

```markdown
1. Spawn a `test-planner` agent (Sonnet) with the full workflow content...
2. Spawn a `browser-tester` agent (Haiku) with the complete test plan...
```

**Generate-Workflow spawns 5 parallel explorers:**

```markdown
Spawn ALL 5 explorer sub-agents IN PARALLEL using the Task tool:
- Explorer 1: Routes & Pages
- Explorer 2: Forms & Validation Schemas
- Explorer 3: Dropdowns & Data Sources
- Explorer 4: API Endpoints & Server Actions
- Explorer 5: Data Flow & UI Behavior
```

**Pattern:** Skills can spawn sub-agents with narrower scopes via Task tool. The skill provides the coordination logic; agents do specialized research.

### Cross-Skill Data Flow

**Plan → Execute → Verify → Review pipeline:**

1. Planner writes `.pipeline/plans/{slug}.md`
2. Executor reads plan, implements, writes `.pipeline/builds/{slug}.md`, moves plan to `done/`
3. Verifier reads build summary + original plan, runs tests, reports
4. Reviewer reads build summary + original plan, writes `.pipeline/reviews/{slug}.md`, commits if PASS

**File-based handoff contract:**
- Each skill reads predecessors' outputs
- Each skill writes artifacts in predictable locations
- No direct skill-to-skill messaging — communication via structured files

### MCP Composability

**All four workflow skills use the same MCP pattern:**

```yaml
allowed-tools:
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
  - mcp__microsoft-docs__microsoft_docs_search
  - mcp__microsoft-docs__microsoft_code_sample_search
  - mcp__microsoft-docs__microsoft_docs_fetch
  - mcp__shadcn__getComponents
  - mcp__shadcn__getComponent
```

**Instruction pattern (Planner and Executor):**

```markdown
#### Library documentation — context7 MCP
**For EVERY library or framework the implementation will touch:**
1. `mcp__context7__resolve-library-id` — resolve the library name
2. `mcp__context7__query-docs` — look up the specific APIs you'll reference

Do NOT skip this. Do NOT assume you know the current API. Libraries change.
```

**Why this matters:** Skills enforce MCP usage through explicit workflow steps, not just tool availability.

## Tool Restrictions

### Role-Based Tool Allowlists

#### Research Role (Planner)

```yaml
allowed-tools:
  - Read          # Read existing code
  - Glob          # Find files
  - Grep          # Search code
  - Bash          # Run read-only commands
  - Write         # Write plan files only
  - Task          # Spawn sub-agents
  - WebSearch     # Research
  - WebFetch      # Fetch docs
  - EnterPlanMode # Plan mode workflow
  - ExitPlanMode  # Plan mode workflow
  - mcp__context7__*        # Library docs
  - mcp__microsoft-docs__*  # Azure docs
  - mcp__shadcn__*          # UI components
```

**NO Edit, NO Skill** — Planner cannot modify code or delegate to other skills.

#### Implementation Role (Executor)

```yaml
allowed-tools:
  - Read
  - Write
  - Edit          # CAN modify source code
  - Glob
  - Grep
  - Bash
  - Task
  - WebSearch
  - WebFetch
  - Skill         # CAN load domain skills
  - mcp__context7__*
  - mcp__microsoft-docs__*
  - mcp__shadcn__*
```

**YES Edit, YES Skill** — Only Executor can modify code and load best-practices skills.

#### Verification Role (Verifier)

```yaml
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write         # Write workflow files and reports
  - Task          # Spawn test agents
  - mcp__chrome-devtools__*  # Full DevTools MCP
```

**NO Edit, NO Skill** — Verifier observes and tests, never modifies source.

#### Review Role (Reviewer)

```yaml
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Write         # Write review files
  - Task
  - Skill         # Load best-practices skills for review
  - mcp__ide__getDiagnostics
```

**NO Edit** — Reviewer reads code, loads skills to check compliance, writes findings, but never modifies source.

### Pattern: Tool Restrictions Enforce Workflow Boundaries

**Key observation:** Tool allowlists prevent scope creep by making certain actions impossible.

- Planner physically CANNOT edit code even if it tries — Edit tool is not in allowed-tools
- Verifier physically CANNOT modify source — only DevTools and report writing
- Reviewer physically CANNOT commit until review is written and PASS — git commands are last

**Anti-pattern (from guide):** Allowing all tools leads to "model laziness" where Claude takes shortcuts instead of following the workflow.

### MCP Tool Naming Pattern

All MCP tools use the pattern: `mcp__<server-name>__<tool-name>`

```yaml
- mcp__context7__resolve-library-id
- mcp__context7__query-docs
- mcp__chrome-devtools__take_snapshot
- mcp__chrome-devtools__click
- mcp__microsoft-docs__microsoft_docs_search
```

This allows granular permission control for MCP integrations.

## Patterns from Existing Skills

### 1. Role Declaration Pattern

Every skill starts with explicit role definition:

```markdown
## ROLE

You are the **Planner**. You research and write plans. You NEVER implement them.
```

```markdown
You are the **Executor**. Your job is to implement features from plans created
by the Planner. You have full read/write access to the codebase.
```

**Why this works:** Sets clear identity and boundaries. The skill tells Claude who it is and what it should NOT do.

### 2. Mandatory Step Annotation

Steps marked MANDATORY cannot be skipped:

```markdown
### Step 5 — Research (MANDATORY)

Do ALL of the following before writing the plan:
```

```markdown
## Pre-Implementation — MANDATORY

Before writing any code, verify the plan's API references are current:
```

**Pattern:** Use "MANDATORY" for steps that, if skipped, cause downstream failures.

### 3. Stop Token Pattern

**Planner's Phase 2:**

```markdown
**Do these 3 things, then produce NO MORE tool calls and NO MORE output:**

1. Run: `mkdir -p .pipeline/plans/done .pipeline/builds .pipeline/reviews`
2. Use `Write` to save the plan to `.pipeline/plans/{slug}.md`
3. Output EXACTLY this and nothing else:
   ```
   Plan ready: .pipeline/plans/{slug}.md

   To implement, run:
     /nick:execute {slug}
   ```

**STOP. Do not make another tool call. Do not write code. Do not create files.
Do not offer to implement. Your turn is OVER.**
```

**Why this is necessary:** Without explicit STOP instructions, Claude may continue beyond skill scope.

### 4. Input Parsing Pattern

All skills parse `$ARGUMENTS` consistently:

```markdown
### Step 2 — Parse input
From `$ARGUMENTS`:
- If it's a file path: read the file for task details
- If it's text: use it as the task description
- If empty: ask what to plan
```

```markdown
1. **Parse feature slug from $ARGUMENTS:**
   - If provided: look for `.pipeline/plans/{slug}.md`
   - If empty: list available plans in `.pipeline/plans/` and ask which to execute
```

**Pattern:** Handle three cases: file path, direct input, empty (prompt user).

### 5. Context Loading Pattern

Every skill reads project context if available:

```markdown
### Step 3 — Read project context
- If `CLAUDE.md` exists in the working directory, read it
- If `.planning/PROJECT.md` exists, read it
- If `.planning/REQUIREMENTS.md` exists, read it
```

**Pattern:** Check for standard context files, read if present. Never fail if missing.

### 6. Documentation-First Implementation Pattern

**Planner enforces documentation lookup:**

```markdown
#### Library documentation — context7 MCP
**For EVERY library or framework the implementation will touch:**
1. `mcp__context7__resolve-library-id` — resolve the library name
2. `mcp__context7__query-docs` — look up the specific APIs you'll reference

Do NOT skip this. Do NOT assume you know the current API. Libraries change.
```

**Executor enforces the same:**

```markdown
Before writing any code, verify the plan's API references are current:

If the plan references an outdated API, adapt to the current one and note the
deviation in the build summary.
```

**Why this works:** Treats documentation as source of truth, not model's training data.

### 7. Skill-Loading Guards

**Executor has conditional skill loading:**

```markdown
**React/Next.js code:** Load the `vercel-react-best-practices` skill BEFORE
writing any React component, hook, or Next.js page. Follow every pattern.

**NestJS code:** Load the `nestjs-best-practices` skill BEFORE writing any
NestJS controller, service, module, guard, pipe, or interceptor.
```

**Pattern:** IF working with framework X, THEN load skill Y BEFORE writing code.

### 8. Verification Before Action Pattern

**Executor runs checks after changes:**

```markdown
2. After each major change:
   - Run typecheck if available: `npm run typecheck` or `npx tsc --noEmit`
   - Run relevant tests: `npm run test -- {relevant-file}`
```

**Reviewer asks before expensive operations:**

```markdown
**Before running the security review, ask the user:**

> Run OWASP security review?
> 1. **Yes** — Full security audit
> 2. **No** — Skip security review
```

**Pattern:** Fast feedback loops (typecheck after change) and user consent for expensive operations.

### 9. Parallel Agent Spawning Pattern

**Generate-Workflow spawns 5 explorers in parallel:**

```markdown
Spawn ALL 5 explorer sub-agents IN PARALLEL using the Task tool. Each one focuses
on a different aspect of the feature.
```

**Pattern:** When tasks are independent, spawn all agents at once. Aggregate results after all return.

### 10. Deviation Tracking Pattern

**Executor documents deviations:**

```markdown
## Deviations from Plan
Any changes from the plan with reasoning. "None" if followed exactly.
```

**Update-Workflow marks changes inline:**

```markdown
Mark every change you make with an inline comment so the user can review:
<!-- UPDATED: constraint changed from max 255 to max 500 -->
<!-- ADDED: new field Gewicht -->
<!-- REMOVED: field Opmerking no longer in schema -->
```

**Pattern:** Always document when actual behavior differs from spec.

### 11. Handoff Message Pattern

Every skill ends with explicit handoff instructions:

```markdown
Plan ready: .pipeline/plans/{slug}.md

To implement, run:
  /nick:execute {slug}
```

```markdown
Build complete: .pipeline/builds/{slug}.md

Run in verifier window:
  /nick:verify {slug}
```

**Pattern:** Tell user what to do next and which command to run.

### 12. Error Handling with Fallback Pattern

**Verifier checks for workflow file:**

```markdown
IF the workflow file EXISTS — auto-run without asking:
[run tests]

IF the workflow file does NOT exist — ask the user:
> 1. Generate workflow + run tests
> 2. Chrome DevTools
> 3. Visual verification
> 4. Skip
```

**Pattern:** If expected resource exists, use it. If missing, offer alternatives.

## Anti-Patterns

### What NOT to Do (from guide)

1. **XML in frontmatter** — Forbidden for security (system prompt injection risk)
   ```yaml
   description: Use for <analysis> tasks  # ❌ FORBIDDEN
   ```

2. **Generic descriptions** — Won't trigger correctly
   ```yaml
   description: Helps with projects  # ❌ Too vague
   ```

3. **Missing trigger phrases** — Undertriggers
   ```yaml
   description: Creates sophisticated multi-page documentation systems  # ❌ No user phrases
   ```

4. **Too technical, no user triggers** — Undertriggers
   ```yaml
   description: Implements the Project entity model with hierarchical relationships  # ❌
   ```

5. **Including README.md in skill folder** — Against spec
   ```
   skill-name/
   ├── SKILL.md
   └── README.md  # ❌ WRONG - use references/ instead
   ```

6. **Wrong file naming**
   ```
   SKILL.MD      # ❌ Wrong case
   skill.md      # ❌ Wrong case
   Skill.md      # ❌ Wrong case
   ```

7. **Wrong folder naming**
   ```
   Notion Project Setup    # ❌ Spaces
   notion_project_setup    # ❌ Underscores
   NotionProjectSetup      # ❌ Capitals
   ```

### Anti-Patterns from Prototype Analysis

8. **Scope creep through unrestricted tools** — Violates separation of concerns
   - Don't give Planner the Edit tool
   - Don't give Verifier write access to source files
   - Don't give skills all tools "just in case"

9. **Assuming API knowledge** — Leads to outdated code
   ```markdown
   # ❌ WRONG
   Use React.useEffect(() => {...})

   # ✅ RIGHT
   Before writing any code:
   1. mcp__context7__resolve-library-id "react"
   2. mcp__context7__query-docs with library ID for useEffect
   3. Follow the current API from docs
   ```

10. **Implicit workflow steps** — Claude may skip
    ```markdown
    # ❌ WRONG
    Review the code and fix issues

    # ✅ RIGHT
    ### Step 1: Code Review — Read every file (MANDATORY)
    Read ALL files listed in the build summary. Check for:
    - Logic errors, edge cases, missing error handling
    - TypeScript strictness (no `any`, proper types)
    ```

11. **Vague output requirements** — Inconsistent results
    ```markdown
    # ❌ WRONG
    Write a build summary

    # ✅ RIGHT
    Write build summary to `.pipeline/builds/{slug}.md`:
    ```markdown
    # Build: {Feature Name}
    ## Files Created
    ...
    ``` (with complete template)
    ```

12. **No stop conditions** — Skills continue beyond scope
    ```markdown
    # ❌ WRONG (ends with just saving the plan)

    # ✅ RIGHT
    **STOP. Do not make another tool call. Do not write code.
    Do not create files. Your turn is OVER.**
    ```

13. **Mixing concerns** — One skill doing multiple jobs
    - Don't have Planner also verify runtime behavior
    - Don't have Reviewer also fix the code
    - Each skill = one role

14. **No deviation tracking** — Lost context
    - Always document when implementation differs from plan
    - Always mark changes in updated documents

15. **Missing handoff instructions** — User doesn't know what's next
    - Every skill should end with "what to do next"
    - Specify exact command to run

## Recommendations

### For GSD → Skills Migration

#### 1. Maintain Role Separation

The current GSD workflow has clear phases (plan, execute, verify, review). Preserve this in skills:

- **One skill per role:** plan, execute, verify, review
- **Tool restrictions enforce boundaries:** Planner can't edit, Verifier can't commit
- **File-based handoff:** Each skill writes artifacts for the next

#### 2. Use allowed-tools Strictly

```yaml
# Planner
allowed-tools:
  - Read, Glob, Grep, Bash, Write, Task
  - WebSearch, WebFetch
  - MCP documentation tools
  - EnterPlanMode, ExitPlanMode
  # EXCLUDE: Edit, Skill

# Executor
allowed-tools:
  - Read, Write, Edit, Glob, Grep, Bash, Task
  - WebSearch, WebFetch
  - Skill  # Can load best-practices skills
  - MCP documentation tools
  # EXCLUDE: chrome-devtools, getDiagnostics

# Verifier
allowed-tools:
  - Read, Glob, Grep, Bash, Write, Task
  - chrome-devtools tools
  # EXCLUDE: Edit, Skill

# Reviewer
allowed-tools:
  - Read, Glob, Grep, Bash, Write, Task, Skill
  - mcp__ide__getDiagnostics
  # EXCLUDE: Edit, chrome-devtools
```

#### 3. Preserve the Phase-Step Pattern

All GSD skills already use this. Keep it:

```markdown
## Phase 1: [Name]
### Step 1 — [Action]
### Step 2 — [Action] (MANDATORY)

## Phase 2: [Name]
...
```

**Why:** Clear progression, explicit ordering, MANDATORY tags prevent skipping.

#### 4. Document Templates In-Skill

All skills that generate structured output should include complete templates:

```markdown
### Step 6 — Write the plan

Compose using this template:

\`\`\`markdown
# Plan: {Feature Name}
## Summary
...
\`\`\`
```

**Why:** Ensures consistency across skill invocations.

#### 5. Explicit Stop Instructions

Every skill needs clear termination:

```markdown
**STOP. Do not make another tool call. Your turn is OVER.**
```

**Why:** Prevents skills from continuing beyond their scope.

#### 6. MCP Integration Pattern

For any skill that needs library/framework documentation:

```markdown
#### Library documentation — context7 MCP
**For EVERY library the [plan/implementation] will touch:**
1. `mcp__context7__resolve-library-id` — resolve the library name
2. `mcp__context7__query-docs` — look up the specific APIs

Do NOT skip this. Do NOT assume you know the current API.
```

**Why:** Forces documentation-first approach, prevents outdated code.

#### 7. Composability via Skill Tool

Executor should delegate to domain skills:

```markdown
### Skills — MANDATORY for relevant code

**React/Next.js code:** Load `vercel-react-best-practices` BEFORE writing components
**NestJS code:** Load `nestjs-best-practices` BEFORE writing services
**UI components:** Load `frontend-design` BEFORE writing layouts
**Security code:** Load `owasp-security` for auth/API/input handling
```

**Why:** Executor orchestrates, domain skills provide expertise.

#### 8. Skill Descriptions Should Trigger Correctly

Current GSD skills have good descriptions:
- `Plan a feature — explore codebase, look up docs, write implementation plan`
- `Execute a plan — implement code from a planner-created plan file`

**Pattern:**
```
[Action verb] + [specific scope] + [method/context]
```

**Include:**
- What the skill does
- When to use it (implicit: "plan", "execute", "verify", "review")
- Key capabilities (explore, implement, test, check quality)

#### 9. argument-hint for User Guidance

All GSD skills use `argument-hint`. This isn't in the official guide but is valuable:

```yaml
argument-hint: "<feature-slug>"
argument-hint: "<task-description or path-to-task-file>"
```

**Recommendation:** Keep this pattern. It helps users know what input the skill expects.

#### 10. File-Based Workflow Coordination

Current pattern works well:

```
.pipeline/
├── plans/
│   ├── {slug}.md          # Planner writes
│   └── done/{slug}.md     # Executor moves here
├── builds/
│   └── {slug}.md          # Executor writes
└── reviews/
    └── {slug}.md          # Reviewer writes
```

**Recommendation:** Keep this. Clear artifact trail, no need for skill-to-skill messaging.

#### 11. Parallel Agent Spawning Where Appropriate

Generate-Workflow spawns 5 explorers in parallel. This is expensive but effective for deep analysis.

**Recommendation:**
- Use parallel spawning for independent research tasks
- Use sequential spawning for dependent tasks
- Document cost in skill description or ask user before spawning

#### 12. Conditional Skill Loading

Reviewer asks before running OWASP security review. Verifier asks before generating workflows.

**Pattern:**
```markdown
**Before running [expensive operation], ask the user:**

> [Operation description]?
> 1. **Yes** — [what happens]
> 2. **No** — [what happens instead]
```

**Recommendation:** Ask before expensive operations (multi-agent spawning, security scans).

#### 13. Deviation Tracking

Executor and Update-Workflow track deviations explicitly.

**Recommendation:**
- Executor: Document deviations from plan in build summary
- Update-Workflow: Mark every change with inline comments
- Reviewer: Note plan adherence in review

#### 14. Verification Before Commit

Reviewer is the ONLY skill that commits, and ONLY after review PASS.

**Recommendation:** Keep this pattern. Prevents committing broken/non-compliant code.

#### 15. Migration Strategy

**Phase 1:** Convert existing prototypes to skills
- Add SKILL.md frontmatter to each .md file
- Move to skill folders with kebab-case names
- Test triggering with natural language

**Phase 2:** Refine tool restrictions
- Audit allowed-tools for each skill
- Remove unnecessary tools
- Test that restrictions enforce boundaries

**Phase 3:** Add MCP integration
- Ensure all skills have context7, microsoft-docs, shadcn where needed
- Add MANDATORY documentation lookup steps
- Test that skills actually look up docs

**Phase 4:** Test composability
- Verify Executor can load domain skills
- Verify agents can spawn sub-agents
- Test full plan→execute→verify→review pipeline

**Phase 5:** Optimize progressive disclosure
- Check SKILL.md body length (keep under 200 lines)
- Move detailed docs to references/
- Test that skills still trigger correctly

### Success Metrics

**Triggering:**
- Skill loads on 90%+ of relevant queries
- Skill doesn't load on unrelated queries
- No need to manually enable skills

**Execution:**
- Workflow completes without user correction
- Tool restrictions prevent scope violations
- Consistent output across runs

**Composability:**
- Executor successfully loads domain skills
- Agents spawn successfully and return results
- File-based handoffs work across skill boundaries

**Performance:**
- SKILL.md under 200 lines (move extras to references/)
- No unnecessary tool permissions
- MCP calls succeed on first try (proper setup in instructions)

**Maintenance:**
- Deviations documented
- Changes tracked
- Handoff instructions clear

---

## Notes

This research is based on:
- Complete Claude Code Skill Guide (comprehensive official documentation)
- 7 GSD skill prototypes (plan, execute, verify, review, test-flow, generate-workflow, update-workflow)

Production skills in `/Users/nick/.claude/skills/` were not accessible during this research session due to permission restrictions. Patterns documented here are derived from the guide and GSD prototypes.

The GSD prototypes are already very well-designed and follow most official best practices. The main migration work will be:
1. Converting .md files to proper SKILL.md format with frontmatter
2. Testing triggering behavior
3. Ensuring tool restrictions are enforced
4. Verifying MCP integration works as designed
