# Codebase Structure

**Analysis Date:** 2026-02-15

## Directory Layout

```
get-shit-done/
├── bin/
│   └── install.js              # Entry point for npm installation
├── commands/
│   └── gsd/                    # Command definitions (31 .md files)
│       ├── plan-phase.md
│       ├── execute-phase.md
│       ├── map-codebase.md
│       ├── new-project.md
│       └── ...                 # Other commands
├── agents/
│   └── gsd-*.md               # Specialized agent prompts (11 files)
│       ├── gsd-planner.md
│       ├── gsd-executor.md
│       ├── gsd-verifier.md
│       ├── gsd-codebase-mapper.md
│       └── ...
├── get-shit-done/
│   ├── bin/
│   │   ├── gsd-tools.cjs       # Core tools/CLI utility (170k lines)
│   │   └── gsd-tools.test.cjs  # Test suite
│   ├── templates/
│   │   ├── plan.md             # PLAN.md template
│   │   ├── summary.md          # SUMMARY.md template
│   │   ├── state.md            # STATE.md template
│   │   ├── roadmap.md          # ROADMAP.md template
│   │   ├── project.md          # PROJECT.md template
│   │   ├── verification-report.md
│   │   ├── context.md          # CONTEXT.md for user decisions
│   │   └── codebase/           # Templates for codebase analysis
│   │       ├── architecture.md
│   │       ├── structure.md
│   │       ├── stack.md
│   │       └── ...
│   ├── workflows/
│   │   ├── plan-phase.md       # Orchestrate plan creation
│   │   ├── execute-phase.md    # Orchestrate phase execution
│   │   ├── map-codebase.md     # Orchestrate codebase mapping
│   │   ├── plan-milestone-gaps.md
│   │   └── ...                 # 30+ workflow files
│   └── references/
│       ├── model-profiles.md   # Agent model assignments
│       ├── verification-patterns.md
│       ├── git-integration.md
│       ├── tdd.md
│       ├── checkpoints.md
│       └── ...
├── hooks/
│   ├── gsd-check-update.js     # Version update checker
│   └── gsd-statusline.js       # Prompt status line hook
├── scripts/
│   └── build-hooks.js          # Hook compilation script
├── package.json
├── .planning/
│   └── codebase/               # Codebase analysis documents (created by /gsd:map-codebase)
│       ├── STACK.md
│       ├── ARCHITECTURE.md
│       └── ...
└── assets/
    └── terminal.svg            # UI/marketing assets
```

## Directory Purposes

**`bin/`:**
- Purpose: Package entry point and installer
- Contains: Node.js executable script that handles multi-platform installation
- Key files: `install.js` — reads package.json, prompts for runtime/location, creates symlinks or copies files

**`commands/gsd/`:**
- Purpose: User-facing command definitions
- Contains: 31 markdown files (one per command), each with YAML frontmatter + workflow reference
- Each file maps a `/gsd:command-name` entry to an orchestrator workflow
- Examples:
  - `plan-phase.md` → orchestrates planning workflow
  - `execute-phase.md` → orchestrates execution workflow
  - `new-project.md` → initializes new project
- Format: YAML header defines agent, allowed tools, description; markdown body references `@~/.claude/get-shit-done/workflows/...`

**`agents/`:**
- Purpose: Specialized agent prompts for domain work
- Contains: 11 markdown files with detailed role definitions and execution logic
- Agents: gsd-planner, gsd-executor, gsd-verifier, gsd-codebase-mapper, gsd-debugger, gsd-roadmapper, gsd-phase-researcher, gsd-project-researcher, gsd-plan-checker, gsd-integration-checker, gsd-research-synthesizer
- Format: YAML header (name, description, tools) + markdown with `<role>`, `<discovery_levels>`, `<execution_flow>` sections
- Usage: Spawned by workflows via Task tool with `subagent_type="gsd-NAME"`

**`get-shit-done/bin/`:**
- Purpose: Core utilities for GSD operations
- Key files:
  - `gsd-tools.cjs` — 40+ subcommands (state, phase, roadmap, commit, scaffold, verify, template, etc.)
  - `gsd-tools.test.cjs` — Node test file (~2000 lines) covering all major commands
- Accessed by: All workflows call this binary via `node ~/.claude/get-shit-done/bin/gsd-tools.cjs`

**`get-shit-done/templates/`:**
- Purpose: Pre-filled markdown document templates
- Core templates (project-level):
  - `state.md` — Project state with decisions, blockers, session continuity
  - `roadmap.md` — Phase breakdown with goals, success criteria, plans
  - `project.md` — Project description, requirements, context
  - `requirements.md` — Requirements list with validation status
  - `plan.md` — Executable task breakdown template (filled per-plan by planner)
  - `summary.md` — Execution summary with tasks, deviations, metrics
  - `verification-report.md` — Artifact/link verification checklist
- Codebase templates (`codebase/` subdirectory):
  - `stack.md` — Technology stack analysis
  - `architecture.md` — System design and patterns
  - `structure.md` — Directory layout
  - `conventions.md` — Coding patterns
  - `testing.md` — Test structure
  - `integrations.md` — External services
  - `concerns.md` — Technical debt
- Usage: gsd-tools.cjs `scaffold` and `template fill` commands populate these

**`get-shit-done/workflows/`:**
- Purpose: Step-by-step orchestration logic for multi-agent workflows
- Contains: 30+ markdown files defining workflow steps
- Key workflows:
  - `plan-phase.md` — Research → Plan → Verify loop
  - `execute-phase.md` — Load plans → Group waves → Execute in parallel → Verify
  - `map-codebase.md` — Spawn 4 parallel mapper agents for codebase analysis
  - `new-project.md` — Initialize project structure and documents
  - `complete-milestone.md` — Archive phases and create MILESTONES.md
- Structure: Each workflow has `<step>` sections with bash commands, JSON parsing, agent spawning
- Used by: Commands reference workflows via `@~/.claude/get-shit-done/workflows/FILE.md`

**`get-shit-done/references/`:**
- Purpose: Specifications and calculation logic
- Key files:
  - `model-profiles.md` — Maps agent types to model tiers (quality/balanced/budget)
  - `decimal-phase-calculation.md` — Algorithm for phase numbering with insertions
  - `verification-patterns.md` — Validation rules for SUMMARY.md, artifacts, links
  - `git-integration.md` — Commit message format, branch naming, strategies
  - `checkpoints.md` — Protocol for long-run execution (>30 min) with resume
  - `tdd.md` — Test-driven development patterns within GSD
  - `planning-config.md` — CONFIG.json schema and defaults
  - `continuation-format.md` — Resume file format for session continuity
- Used by: Workflows reference these for specifications; agents read for patterns

**`hooks/`:**
- Purpose: Post-command integrations and updates
- Files:
  - `gsd-check-update.js` — Checks npm registry for newer version after command runs
  - `gsd-statusline.js` — Displays GSD status in CLI prompt (optional enhancement)
- Deployment: Built to `hooks/dist/` and included in npm package

**`scripts/`:**
- Purpose: Build and deployment helpers
- Files: `build-hooks.js` — Compiles hook scripts for distribution
- Run via: `npm run build:hooks`

**`.planning/codebase/`:**
- Purpose: Codebase analysis documents created by `/gsd:map-codebase`
- Created by: 4 parallel gsd-codebase-mapper agents
- Contains: STACK.md, ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, INTEGRATIONS.md, CONCERNS.md
- Usage: Referenced by future phases during planning/execution

**`assets/`:**
- Purpose: UI and marketing materials
- Contains: SVG terminal UI examples, logos
- Not used by GSD runtime; for documentation

## Key File Locations

**Entry Points:**
- `bin/install.js` — npm install entry point; creates symlinks in ~/.claude/ or ./.claude/
- `get-shit-done/bin/gsd-tools.cjs` — Core CLI utility for all state/phase/commit operations
- `commands/gsd/*.md` — User-facing command entry points (loaded by Claude Code runtime)

**Configuration & State:**
- `package.json` — NPM metadata, version, files to package
- `get-shit-done/references/planning-config.md` — CONFIG.json schema
- `.planning/config.json` — User config (branching strategy, models, settings) — created by `/gsd:new-project`

**Core Logic:**
- `get-shit-done/workflows/` — Multi-step orchestration logic
- `get-shit-done/bin/gsd-tools.cjs` — Atomic operations (40+ subcommands)
- `agents/gsd-*.md` — Domain-specific agent implementations

**Testing:**
- `get-shit-done/bin/gsd-tools.test.cjs` — Node test suite; run via `npm test`

## Naming Conventions

**Files:**
- Commands: `command-name.md` (kebab-case)
- Agents: `gsd-agent-type.md` (kebab-case, always prefixed `gsd-`)
- Workflows: `workflow-name.md` (kebab-case)
- Templates: `template-name.md` (lowercase)
- References: `reference-name.md` (kebab-case, lowercase)

**Directories:**
- Installation targets: `.claude/`, `.opencode/`, `.gemini/` (dot-prefixed by runtime)
- Phase directories: `phases/01-phase-slug/`, `phases/01.1-subphase/` (padded numbers, slugified name)
- Plan files: `PLAN-01-wave-1.md`, `PLAN-02-wave-1.md` (UPPERCASE, plan number, wave)
- Summary files: `SUMMARY-01-wave-1.md` (UPPERCASE)

**Project-level Documents:**
- All UPPERCASE: ROADMAP.md, STATE.md, REQUIREMENTS.md, PROJECT.md, PLAN.md, SUMMARY.md
- Timestamps: append `-YYYY-MM-DD` if keeping history (e.g., `STATE.md.bak`)

**Command Arguments:**
- Flags: kebab-case with leading dashes (e.g., `--skip-research`, `--gaps`)
- Phases: numeric or decimal (e.g., `1`, `2.1`)

## Where to Add New Code

**New Command:**
1. Create `commands/gsd/new-command.md` with YAML header and `@~/.claude/get-shit-done/workflows/...` reference
2. Create corresponding workflow at `get-shit-done/workflows/new-command.md` with step definitions
3. Register in gsd-tools.cjs if needs state/phase operations

**New Agent:**
1. Create `agents/gsd-new-agent.md` with:
   - YAML header: name, description, tools
   - `<role>` section defining responsibilities
   - `<discovery_levels>` for initial context reading
   - `<execution_flow>` for main work steps
2. Reference in workflow via Task tool: `subagent_type="gsd-new-agent"`
3. Add model profile entry in `get-shit-done/references/model-profiles.md`

**New Workflow Step:**
1. Edit relevant workflow .md file in `get-shit-done/workflows/`
2. Add `<step name="step-name">` section with:
   - Description of what happens
   - Bash commands (if any) for state operations
   - Agent spawning (if any) with full context loading
   - Error handling and validation

**New Subcommand (gsd-tools.cjs):**
1. Edit `get-shit-done/bin/gsd-tools.cjs`
2. Add handler function following existing pattern (parse args, validate, execute, output JSON)
3. Register in main switch statement (line ~500)
4. Update JSDoc comment at top with new subcommand

**New Template:**
1. Create `get-shit-done/templates/template-name.md`
2. Use `<!-- PLACEHOLDER: description -->` for fields to be filled
3. Register scaffold command in gsd-tools.cjs to generate instances

## Special Directories

**`.planning/` (Project Root):**
- Purpose: All project-level planning documents
- Generated: By `/gsd:new-project`
- Committed: YES — everything in .planning/ is git-tracked
- Contents:
  - `STATE.md` — Current session state, decisions, blockers
  - `ROADMAP.md` — Phase definitions and progress
  - `REQUIREMENTS.md` — Project requirements with validation
  - `PROJECT.md` — Project description and core value
  - `CONTEXT.md` — Optional user decisions from `/gsd:discuss-phase`
  - `VERIFICATION.md` — Optional gap list from previous verification
  - `RESEARCH.md` — Optional research from `/gsd:research-phase`
  - `config.json` — User settings (models, branching, etc.)
  - `phases/` — Phase working directories
  - `codebase/` — Codebase analysis documents (from `/gsd:map-codebase`)

**`.planning/phases/XX-phase-slug/`:**
- Purpose: All artifacts for a single phase
- Generated: By `/gsd:add-phase` or `/gsd:new-project` for phase 1
- Committed: YES
- Contents:
  - `PLAN-01-wave-1.md` — First plan, first wave
  - `PLAN-02-wave-1.md` — Second plan, first wave
  - `PLAN-03-wave-2.md` — Third plan, second wave
  - `SUMMARY-01-wave-1.md` — Execution summary for plan 01
  - `SUMMARY-02-wave-1.md` — Execution summary for plan 02
  - `RESEARCH.md` — Domain research (from researcher agent)
  - `CONTEXT.md` — User decisions (from `/gsd:discuss-phase`)
  - `VERIFICATION.md` — Gap list (from verifier agent)

**`hooks/dist/`:**
- Purpose: Compiled hook scripts
- Generated: By `npm run build:hooks`
- Committed: YES — includes compiled hooks for distribution
- Not modified manually; output of build process

## Import Patterns

**In commands/workflows/agents:**
- File references: `@~/.claude/get-shit-done/templates/plan.md` or `@.planning/ROADMAP.md`
- Data flow: Pass via environment variables (JSON strings) or bash variables
- Context: Loaded via `node ~/.claude/get-shit-done/bin/gsd-tools.cjs init <workflow> [args]`

**In gsd-tools.cjs:**
- All imports are Node.js built-ins: `require('fs')`, `require('path')`, `require('child_process')`
- No external npm dependencies (intentionally minimal)
- Patterns: File I/O, JSON parsing, shell command execution

---

*Structure analysis: 2026-02-15*
