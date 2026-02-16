# Skills Architecture Research: Converting GSD Workflows to Claude Code Skills

Research date: 2025-02-15

---

## Key Findings

- **GSD currently has 30 workflow .md files and 11 agent types.** The workflows range from complex orchestrators (new-project, plan-phase, execute-phase) that spawn multiple agents across hundreds of lines, to simple utilities (add-phase, set-profile, help) that are under 50 lines of actionable instructions.

- **The user already has 7 working skills** at `docs/nick/` (plan, execute, verify, review, generate-workflow, update-workflow, test-flow). These follow a flat file pattern -- each is a single `.md` with YAML frontmatter -- NOT the full SKILL.md folder structure. They work because Claude Code loads `.md` files from `docs/` directories as slash commands.

- **GSD workflows use a fundamentally different architecture from the user's existing skills.** GSD workflows: (a) use XML tags (`<purpose>`, `<process>`, `<step>`, `<success_criteria>`), (b) call a Node.js CLI tool (`gsd-tools.js`) for initialization and state management, (c) spawn typed subagents (`gsd-planner`, `gsd-executor`, `gsd-verifier`, etc.) via the Task tool, (d) reference external files via `@` syntax and absolute paths, (e) maintain persistent state across commands via `.planning/STATE.md` and `config.json`.

- **The SKILL.md format has strict constraints that conflict with current GSD patterns.** No XML angle brackets in frontmatter. The body should be lean (under 5,000 words recommended). Large reference material goes in `references/`. But GSD workflows like `new-project.md` (935 lines) and `execute-phase.md` (339 lines) are very large.

- **Progressive disclosure is the critical design principle.** Level 1 (frontmatter) is always loaded -- token cost for every conversation. Level 2 (body) loads when Claude thinks the skill is relevant. Level 3 (references/) loads on demand. This means frontmatter descriptions must be precise trigger phrases, and bulky content must be pushed to references/.

- **The `allowed-tools` field in frontmatter is key for GSD.** The user's existing skills already use this well (e.g., `plan.md` restricts to Read, Glob, Grep, Bash, Write, Task, WebSearch, WebFetch, EnterPlanMode, ExitPlanMode, plus specific MCP tools). This scopes what each skill can do.

- **The `argument-hint` field is not standard SKILL.md but works in Claude Code.** The user's existing skills use `argument-hint` for specifying expected arguments (e.g., `"<feature-slug>"`). This maps to the slash command's argument pattern in Claude Code.

- **Agent spawning via Task tool works from within skills.** The user's existing skills (verify.md, generate-workflow.md) already spawn subagents via Task. This pattern carries over directly to SKILL.md format.

- **The `nick:` namespace prefix is the user's personal skill namespace.** Skills are named `nick:plan`, `nick:execute`, etc. GSD workflows use `gsd:` namespace. Both can coexist.

---

## Workflow-to-Skill Mapping Strategy

### The 30 GSD Workflows

Analyzing all 30 workflows by complexity and usage pattern:

| Category | Workflows | Count | Conversion Strategy |
|----------|-----------|-------|---------------------|
| **Core Lifecycle** | new-project, plan-phase, execute-phase, verify-phase, execute-plan | 5 | Individual skills with references/ |
| **Phase Planning** | discuss-phase, research-phase, discovery-phase, list-phase-assumptions | 4 | Individual skills (moderate size) |
| **Quick Mode** | quick | 1 | Individual skill |
| **Roadmap Mgmt** | add-phase, insert-phase, remove-phase | 3 | Group into one skill or keep individual |
| **Milestone Mgmt** | new-milestone, complete-milestone, audit-milestone, plan-milestone-gaps | 4 | Group into one or two skills |
| **Progress/Session** | progress, resume-project, pause-work | 3 | Individual skills (small) |
| **Verification/UAT** | verify-work, verify-phase | 2 | Individual skills |
| **Debugging** | diagnose-issues | 1 | Individual skill |
| **Todo** | add-todo, check-todos | 2 | Group into one skill |
| **Utility** | help, update, settings, set-profile, transition | 5 | Individual skills (small, simple) |
| **Workflow Testing** | (covered by nick: test-flow, generate-workflow, update-workflow) | 0 | Already converted |

### Recommended Approach: Hybrid (Not Pure 1:1)

**1:1 mapping for all core and moderately complex workflows.** Every GSD command that users invoke directly as `/gsd:something` should have a corresponding skill. This preserves the mental model users already have. The mapping is:

```
/gsd:new-project      ->  gsd-new-project/SKILL.md
/gsd:plan-phase       ->  gsd-plan-phase/SKILL.md
/gsd:execute-phase    ->  gsd-execute-phase/SKILL.md
/gsd:quick            ->  gsd-quick/SKILL.md
/gsd:progress         ->  gsd-progress/SKILL.md
... etc for all 30 commands
```

**Group only where commands share >80% of their logic.** The roadmap commands (add-phase, insert-phase, remove-phase) are distinct enough in behavior that grouping would create confusing trigger conditions. Keep them separate.

**Exception: The `help` command becomes unnecessary.** Skills have built-in discovery through frontmatter descriptions. The help text could become a `references/command-reference.md` shared across skills, or simply be dropped since Claude can list available skills.

### The 11 Agent Types as Shared References

The 11 agent `.md` files are NOT commands users invoke -- they are role definitions loaded by workflows when spawning subagents:

```
agents/
  gsd-codebase-mapper.md
  gsd-debugger.md
  gsd-executor.md
  gsd-integration-checker.md
  gsd-phase-researcher.md
  gsd-plan-checker.md
  gsd-planner.md
  gsd-project-researcher.md
  gsd-research-synthesizer.md
  gsd-roadmapper.md
  gsd-verifier.md
```

These should NOT become skills. They should remain as `references/` files that skills load when spawning Task agents. The key pattern (already used in GSD workflows) is:

```
Task(
  prompt="First, read /path/to/agents/gsd-planner.md for your role...",
  ...
)
```

In the skill format, this becomes:

```
Task(
  prompt="First, read references/agents/gsd-planner.md for your role...",
  ...
)
```

---

## SKILL.md Structure Recommendations

### Anatomy of a Converted GSD Skill

```
gsd-plan-phase/
  SKILL.md                           # Frontmatter + lean orchestration body
  references/
    agents/
      gsd-phase-researcher.md        # Agent role: spawned for research
      gsd-planner.md                 # Agent role: spawned for planning
      gsd-plan-checker.md            # Agent role: spawned for verification
    templates/
      plan.md                        # PLAN.md template
    ui-brand.md                      # Display formatting rules
    verification-patterns.md         # Patterns for plan checking
```

### Frontmatter Design

```yaml
---
name: gsd-plan-phase
description: >
  Create detailed execution plans for a roadmap phase. Orchestrates research,
  planning, and verification agents. Use when user says "plan phase",
  "create plan for phase", "/gsd:plan-phase", or wants to plan the next
  phase of their project. Requires .planning/ directory with ROADMAP.md.
argument-hint: "<phase-number> [--research] [--skip-research] [--gaps] [--skip-verify]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
---
```

Key frontmatter decisions:

- **`name`**: Use `gsd-` prefix for the framework namespace. Kebab-case per spec.
- **`description`**: Must include WHAT (create plans for a phase), WHEN (trigger phrases including the old slash command), and SCOPE (requires .planning/). Under 1024 chars.
- **`argument-hint`**: Preserve from current skills -- this is Claude Code-specific and useful.
- **`allowed-tools`**: List only what the orchestrator needs. Subagents spawned via Task get their own tool access.

### Body Structure: The Orchestrator Pattern

GSD workflows are orchestrators -- they coordinate agents, not do the work themselves. The SKILL.md body should reflect this:

```markdown
---
(frontmatter)
---

# Plan Phase

You orchestrate phase planning. You spawn agents, don't do the work yourself.

## Prerequisites
- .planning/ directory must exist (run /gsd:new-project first)
- ROADMAP.md with defined phases

## Step 1: Initialize
(init context via gsd-tools.js -- keep exact bash commands)

## Step 2: Research (conditional)
(spawn researcher agent from references/agents/gsd-phase-researcher.md)

## Step 3: Plan
(spawn planner agent from references/agents/gsd-planner.md)

## Step 4: Verify (conditional)
(spawn checker agent from references/agents/gsd-plan-checker.md)

## Step 5: Report
(present results and next steps)
```

### What Goes in Body vs References

| Content Type | Location | Rationale |
|-------------|----------|-----------|
| Orchestration steps (1-13 steps) | Body | Core instructions, loaded when skill activates |
| Agent role definitions | references/agents/ | Only loaded when spawning that specific agent |
| Templates (plan, summary, verification) | references/templates/ | Only loaded when writing output |
| UI branding rules | references/ui-brand.md | Shared across skills, loaded when displaying |
| Detailed prompt templates | references/prompts/ | Large prompt strings for agent spawning |
| Error handling patterns | references/ | Edge cases, not always needed |
| Questioning techniques | references/questioning.md | Only for new-project and discuss-phase |

### Token Budget Guideline

- **Frontmatter**: ~200-300 tokens (always loaded for ALL conversations)
- **Body**: Target 1,500-3,000 tokens (loaded when skill activates)
- **References**: Unlimited (loaded on demand per file)

For the largest workflows like `new-project.md` (935 lines), the body should contain ONLY the step sequence and decision logic. All prompt templates, agent instructions, and detailed configuration should move to references/. This can reduce body size from ~4,000 tokens to ~1,500 tokens.

### Handling the XML-to-Markdown Conversion

Current GSD workflows use XML tags (`<purpose>`, `<process>`, `<step>`, etc.). SKILL.md does not allow XML in frontmatter and bodies should use standard Markdown. The conversion:

| GSD XML | SKILL.md Markdown |
|---------|-------------------|
| `<purpose>...</purpose>` | Frontmatter `description` + first paragraph of body |
| `<process>` | Body sections with `## Step N:` headings |
| `<step name="...">` | `## Step N: Name` |
| `<required_reading>` | Opening paragraph: "Read these files first:" |
| `<success_criteria>` | `## Success Criteria` section at bottom of body |
| `<offer_next>` | Inline at end of final step |

### Handling the `@` File Reference Syntax

GSD workflows use `@` references to load files: `@.planning/STATE.md`. In skills, this needs to convert to explicit Read tool calls or references/ bundled files:

- **Agent definitions**: Bundle in `references/agents/` -- they ship with the skill
- **Project files** (STATE.md, ROADMAP.md, etc.): Read via tool calls at runtime -- they are dynamic per-project
- **Templates**: Bundle in `references/templates/` -- they are static

---

## Composition Patterns

### How Skills Chain Together

GSD has a natural lifecycle: `new-project -> plan-phase -> execute-phase -> verify-phase -> complete-milestone`. Each skill should be independent but aware of the chain.

**Pattern 1: Handoff via Output Instructions**

Each skill ends with explicit next-step instructions (already present in GSD):

```markdown
## Step N: Completion

Display:
  Plan ready. Next: /gsd:execute-phase {phase}
```

The user manually invokes the next skill. This is the cleanest pattern and matches how the existing skills work.

**Pattern 2: Skill Invocation from Within a Skill**

The `Skill` tool allows one skill to invoke another. The user's `execute.md` already has `Skill` in its allowed-tools. This enables:

```markdown
## Post-Implementation
If all tests pass, invoke the review skill:
  Skill("nick:review", args="{slug}")
```

However, this should be used sparingly. Each skill invocation loads a new context, and chaining multiple skills in a single conversation burns context fast.

**Pattern 3: Shared State via File System**

GSD already uses this pattern extensively -- `.planning/STATE.md`, `config.json`, plan files, summary files. Skills share state through the filesystem:

- `plan-phase` writes PLAN.md files
- `execute-phase` reads PLAN.md, writes SUMMARY.md
- `verify-phase` reads SUMMARY.md, writes VERIFICATION.md
- `progress` reads everything, routes to the next action

This filesystem-as-communication-bus pattern works perfectly with skills and should be preserved exactly as-is.

**Pattern 4: Progress as Router**

The `progress` skill acts as a state machine router:

```
progress reads state -> determines what's next -> suggests specific skill
```

This is the recommended entry point for users who don't know which skill to run. It replaces the need for skills to auto-chain.

### Composition Anti-Patterns to Avoid

- **Do NOT have skills auto-invoke the next skill.** Users should retain control of the workflow pace. Each `/clear` between major steps is intentional -- it resets context.
- **Do NOT embed one skill's full instructions inside another.** Use references/ for shared content.
- **Do NOT create a "meta-skill" that orchestrates all other skills.** That is what `progress` does through routing, not through direct invocation.

---

## Agent Spawning from Skills

### How the Task Tool Works Within Skills

The Task tool spawns a subagent with its own context window. This is GSD's core pattern -- the orchestrator (skill) stays lean while subagents do heavy work.

**Current GSD pattern (from workflows):**

```
Task(
  prompt="First, read /Users/nick/.claude/agents/gsd-planner.md...",
  subagent_type="gsd-planner",
  model="{planner_model}",
  description="Plan Phase {phase}"
)
```

**Converted skill pattern:**

```
Task(
  prompt="First, read references/agents/gsd-planner.md...",
  subagent_type="general-purpose",
  description="Plan Phase {phase}"
)
```

Key differences in the skill version:

1. **Path resolution**: Agent files bundled in `references/agents/` instead of absolute paths. The skill folder is the working directory when the skill runs, so relative paths from the skill root work.

2. **`subagent_type`**: Claude Code supports `"general-purpose"` as the standard type. The custom types (`gsd-planner`, `gsd-executor`) in current GSD are applied via the agent definition file, not the subagent_type parameter. Verify whether custom subagent_type values affect behavior -- if not, standardize on `"general-purpose"` and let the prompt define the role.

3. **Model selection**: The `model` parameter may not be available through the Task tool in all environments. GSD currently uses `gsd-tools.js` to resolve model profiles (quality/balanced/budget) into specific model names. This indirection should be preserved -- the skill reads config.json and passes the appropriate model name.

### Parallel Agent Spawning

GSD's most powerful pattern is parallel Task spawning (e.g., 5 explorer agents in generate-workflow, 4 researchers in new-project). Skills support this because the Task tool can be called multiple times:

```markdown
## Step 6: Research (Parallel)

Spawn ALL 4 researchers in parallel using Task:

**Researcher 1 (Stack):**
Task(prompt="Read references/agents/gsd-project-researcher.md... research stack...", description="Stack research")

**Researcher 2 (Features):**
Task(prompt="Read references/agents/gsd-project-researcher.md... research features...", description="Features research")

(etc.)

Wait for all 4 to complete before proceeding.
```

### Agent Instructions as References

Each agent type's instructions should be a separate file in `references/agents/`:

```
references/
  agents/
    gsd-planner.md           # 200-400 lines of role + instructions
    gsd-executor.md          # Role + task execution patterns
    gsd-verifier.md          # Verification protocol
    gsd-phase-researcher.md  # Research methodology
    gsd-project-researcher.md
    gsd-plan-checker.md
    gsd-roadmapper.md
    gsd-research-synthesizer.md
    gsd-codebase-mapper.md
    gsd-integration-checker.md
    gsd-debugger.md
```

These are loaded only when the orchestrating skill needs to spawn that specific agent type. A skill like `gsd-quick` only needs `gsd-planner.md` and `gsd-executor.md`. A skill like `gsd-plan-phase` needs `gsd-phase-researcher.md`, `gsd-planner.md`, and `gsd-plan-checker.md`.

### Shared vs Skill-Specific References

Some references are used by multiple skills. Options:

**Option A: Duplicate in each skill's references/**

Pros: Self-contained, portable. Cons: Maintenance burden, drift risk.

**Option B: Shared references directory at project level**

```
.claude/skills/
  shared-references/
    agents/
    templates/
    ui-brand.md
  gsd-plan-phase/
    SKILL.md
    references/ -> symlinks or explicit reads from shared
  gsd-execute-phase/
    SKILL.md
    references/ -> symlinks or explicit reads from shared
```

Pros: Single source of truth. Cons: Less portable.

**Recommendation: Option B with absolute paths.** GSD is installed to a known location (`~/.claude/get-shit-done/`). Skills can reference shared files via absolute paths in their instructions, just as workflows do today. This is the same pattern GSD already uses. For distribution as standalone skills (e.g., via GitHub), Option A would be needed -- but GSD is distributed as a single package, not individual skills.

---

## Token Efficiency Analysis

### Current Cost Model (Workflows)

Current GSD workflows are loaded as slash commands. The full workflow file is loaded into context when invoked. For `plan-phase.md` (377 lines), that is approximately 4,000 tokens loaded at invocation. Agent files are loaded by subagents in their own context.

### Projected Cost Model (Skills)

With skills, the cost profile changes:

| Level | When Loaded | Token Cost | What to Put Here |
|-------|-------------|-----------|------------------|
| Frontmatter | Every conversation start | ~200-300 tokens x 30 skills = 6,000-9,000 tokens | Name + description + triggers |
| Body | When skill is invoked | ~1,500-3,000 tokens per skill | Orchestration steps only |
| References | When body reads them | Variable per file | Agent defs, templates, prompts |

**Critical concern: 30 skill frontmatters in system prompt.**

If all 30 GSD skills are installed, that is ~6,000-9,000 tokens of frontmatter always present. This is within acceptable range (Claude Code handles 20-50 skills per the guide), but the descriptions must be efficient. Each description should be 2-3 sentences, not paragraphs.

**Optimization: Disable rarely-used skills.**

Users can disable skills they rarely use (transition, set-profile, join-discord). Recommend a "core" set of ~15 skills enabled by default and an "extended" set that users opt into.

### Body Size Targets

| Skill | Current Workflow Lines | Target Body Tokens | Strategy |
|-------|----------------------|-------------------|----------|
| gsd-new-project | 935 | 2,500 | Heavy use of references/ |
| gsd-plan-phase | 377 | 2,000 | Agent prompts to references/ |
| gsd-execute-phase | 339 | 2,000 | Agent prompts to references/ |
| gsd-verify-phase | 290 | 1,500 | Templates to references/ |
| gsd-progress | 386 | 1,500 | Route logic is body, display is references/ |
| gsd-quick | 231 | 1,200 | Already relatively lean |
| gsd-help | 471 | 200 | Almost all content is reference text |
| gsd-add-phase | 112 | 800 | Small enough to be entirely in body |
| gsd-settings | 146 | 800 | Small enough to be entirely in body |

---

## Recommendations (Prioritized)

### Priority 1: Establish the Skill Folder Structure

Create the canonical folder layout before converting any workflows:

```
commands/gsd/                    # Current: workflow .md files (source of truth)
skills/gsd/                      # New: converted SKILL.md folders
  shared/                        # Shared across skills
    agents/                      # 11 agent role definitions
    templates/                   # Plan, summary, verification, etc.
    references/                  # ui-brand, questioning, etc.
  gsd-new-project/
    SKILL.md
  gsd-plan-phase/
    SKILL.md
  gsd-execute-phase/
    SKILL.md
  ... (28 more)
```

### Priority 2: Convert 3 Core Skills First as Templates

Start with one from each complexity tier:

1. **Simple**: `gsd-add-phase` -- Small utility, no agents, tests the basic conversion.
2. **Medium**: `gsd-progress` -- State reading + routing, no agents, tests decision logic.
3. **Complex**: `gsd-plan-phase` -- Full orchestrator with 3 agent types, research/plan/verify loop.

These three will establish patterns for all remaining conversions.

### Priority 3: Define Shared References Layout

Before converting complex skills, establish where shared content lives:

- Agent definitions (used by multiple skills)
- Templates (plan, summary, verification formats)
- UI branding (display conventions)
- gsd-tools.js CLI reference (common bash patterns)

### Priority 4: Write Frontmatter Description Catalog

Draft all 30 descriptions before converting bodies. The description is the most important field -- it determines triggering. Test each description by asking: "If I were a user, what would I say to trigger this?"

Example catalog:

```yaml
gsd-new-project: >
  Initialize a new GSD project with questioning, research, requirements, and roadmap.
  Use when user says "new project", "start project", "initialize project",
  or "/gsd:new-project". Creates .planning/ directory structure.

gsd-plan-phase: >
  Create execution plans for a roadmap phase with optional research and verification.
  Use when user says "plan phase", "create plans", or "/gsd:plan-phase".
  Requires existing .planning/ROADMAP.md.

gsd-execute-phase: >
  Execute all plans in a phase using wave-based parallel agents.
  Use when user says "execute phase", "run phase", "build phase",
  or "/gsd:execute-phase". Requires planned phase with PLAN.md files.
```

### Priority 5: Convert Remaining Skills in Dependency Order

After the 3 templates are proven, convert in this order:

**Wave 1 (no dependencies):** help, settings, set-profile, update, progress, pause-work, resume-project
**Wave 2 (state readers):** add-todo, check-todos, add-phase, insert-phase, remove-phase
**Wave 3 (agent orchestrators):** discuss-phase, research-phase, discovery-phase, list-phase-assumptions, quick
**Wave 4 (complex orchestrators):** new-project, execute-phase, verify-phase, execute-plan
**Wave 5 (milestone):** new-milestone, complete-milestone, audit-milestone, plan-milestone-gaps, verify-work, diagnose-issues

### Priority 6: Establish Testing Protocol

For each converted skill, verify:

1. **Trigger test**: Does the skill load when expected? (Test 5 trigger phrases)
2. **Non-trigger test**: Does it NOT load for unrelated queries? (Test 3 unrelated phrases)
3. **Functional test**: Does the core workflow complete? (Run the actual command)
4. **Agent spawning test**: Do subagents receive correct references? (Check Task prompts)
5. **Token efficiency**: Is the body under 3,000 tokens? Are references loaded only when needed?

### Priority 7: Address Open Questions

These require experimentation during implementation:

1. **Path resolution in skills**: Do `references/agents/gsd-planner.md` paths resolve relative to the SKILL.md location? Or do we need absolute paths? Test with the first conversion.

2. **`subagent_type` parameter**: Does Claude Code use custom subagent_type values from Task calls? If not, simplify to all `"general-purpose"`.

3. **`argument-hint` field**: This is used in the user's existing skills but is not in the official SKILL.md spec (Reference B). Verify it works or find the equivalent.

4. **Coexistence**: Can `/gsd:plan-phase` (workflow command) and the `gsd-plan-phase` skill coexist? Or does one need to be removed when the other is installed? Test overlap behavior.

5. **`allowed-tools` for Task tool**: When a skill specifies `allowed-tools: [Task]`, do the spawned subagents inherit the skill's tool restrictions, or do they get full access? This affects whether agent definitions need their own tool scoping.

6. **The `nick:` vs `gsd:` namespace question**: The user's personal skills use `nick:plan`, `nick:execute`, etc. GSD framework skills would use `gsd:plan-phase`, etc. These serve different purposes (personal workflow vs framework commands). Both should coexist, but the user should decide whether to maintain two parallel sets or converge.

---

## Appendix: Workflow Inventory

| # | Workflow | Lines | Agents Spawned | Complexity | Shared References Needed |
|---|---------|-------|----------------|------------|--------------------------|
| 1 | new-project | 935 | 4 researchers + synthesizer + roadmapper | Very High | 6 agents, templates, questioning |
| 2 | plan-phase | 377 | researcher + planner + checker | High | 3 agents, templates, ui-brand |
| 3 | execute-phase | 339 | N executors + verifier | High | 2 agents, templates |
| 4 | execute-plan | ~200 | (used by execute-phase) | Medium | templates |
| 5 | verify-phase | 290 | (spawned by execute-phase) | Medium | verification-patterns, templates |
| 6 | new-milestone | ~800 | Similar to new-project | Very High | Same as new-project |
| 7 | discover-phase | 290 | None (self-contained) | Medium | templates |
| 8 | discuss-phase | ~200 | None | Medium | questioning |
| 9 | research-phase | ~200 | researcher | Medium | 1 agent |
| 10 | quick | 231 | planner + executor | Medium | 2 agents |
| 11 | progress | 386 | None | Medium | None |
| 12 | help | 471 | None | Low (static) | None |
| 13 | settings | 146 | None | Low | None |
| 14 | add-phase | 112 | None | Low | None |
| 15 | insert-phase | ~120 | None | Low | None |
| 16 | remove-phase | ~120 | None | Low | None |
| 17 | add-todo | ~150 | None | Low | None |
| 18 | check-todos | ~150 | None | Low | None |
| 19 | set-profile | ~50 | None | Very Low | None |
| 20 | pause-work | ~100 | None | Low | None |
| 21 | resume-project | ~100 | None | Low | None |
| 22 | update | ~80 | None | Very Low | None |
| 23 | verify-work | ~200 | None | Medium | templates |
| 24 | complete-milestone | ~200 | None | Medium | templates |
| 25 | audit-milestone | ~200 | integration-checker | Medium | 1 agent |
| 26 | plan-milestone-gaps | ~150 | None | Medium | None |
| 27 | list-phase-assumptions | ~100 | None | Low | None |
| 28 | diagnose-issues | ~200 | debugger | Medium | 1 agent, templates |
| 29 | map-codebase | ~200 | codebase-mapper | Medium | 1 agent |
| 30 | transition | ~100 | None | Low | None |
