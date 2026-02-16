# Architecture

**Analysis Date:** 2026-02-15

## Pattern Overview

**Overall:** Orchestrator-Agent Command System

GSD is a command-driven orchestration platform that spawns specialized AI agents to handle different phases of spec-driven development. The architecture emphasizes:
- **Separation of concerns:** Each command is a thin orchestrator; agents do substantive work
- **Context minimalism:** Orchestrators stay lean (~10-15% context), agents get fresh full context (200k tokens)
- **Document-driven state:** All state lives in git-committed markdown files (ROADMAP.md, STATE.md, PLAN.md, SUMMARY.md)
- **Atomic commits:** Each task produces a git commit; SUMMARY.md tracks what was done and why

**Key Characteristics:**
- Single-actor workflow (one developer + Claude)
- Phase-based decomposition with decimal phase insertion for urgent work
- Wave-based parallel execution within phases
- Spec-driven: PLAN.md IS the execution prompt, not a document that becomes one
- Quality curve awareness: plans complete within ~50% context to maintain quality
- Checkpoint protocol for long runs, with resume capability

## Layers

**CLI/Commands Layer:**
- Purpose: Parse user input, display banners, manage workflow entry points
- Location: `commands/gsd/` (31 command definition files)
- Contains: YAML frontmatter + markdown workflow templates
- Depends on: gsd-tools.cjs for state/model resolution
- Used by: Claude Code runtime as `/gsd:command-name` entry points
- Example: `commands/gsd/plan-phase.md` → invokes `/gsd:plan-phase` with arguments

**Workflow/Orchestration Layer:**
- Purpose: Coordinate multi-agent workflows, validate inputs, manage branching/state transitions
- Location: `get-shit-done/workflows/` (30+ workflow files)
- Contains: Step-by-step process definitions, context loading, agent spawning, result collection
- Depends on: gsd-tools.cjs for compound operations (init, roadmap parsing, commit)
- Used by: Commands invoke workflow files via `@~/.claude/get-shit-done/workflows/FILE.md` references
- Example: `plan-phase.md` orchestrates researcher → planner → checker agents

**Agent Layer:**
- Purpose: Execute domain-specific work (planning, execution, verification, research)
- Location: `agents/` (11 specialized agents)
- Contains: Agent prompts with role definition, discovery protocols, validation logic
- Depends on: Reading project files (PLAN.md, ROADMAP.md, STATE.md, etc.)
- Used by: Workflows spawn agents via Task tool with `subagent_type="gsd-NAME"` and `model="..."`
- Agents write results directly: PLAN.md, SUMMARY.md, RESEARCH.md, etc.
- Examples:
  - `gsd-planner.md`: Creates PLAN.md with task breakdown, waves, must-haves
  - `gsd-executor.md`: Executes plans, commits per-task, produces SUMMARY.md
  - `gsd-verifier.md`: Validates SUMMARY.md, checks artifacts/links

**Tools/Utilities Layer:**
- Purpose: Atomic operations for state, phase, git, templates, validation
- Location: `get-shit-done/bin/gsd-tools.cjs` (170k+ lines)
- Contains: 40+ subcommands (state, phase, roadmap, commit, template, verify, etc.)
- Depends on: Node.js fs, path, execSync for git operations
- Used by: Workflows call via `node ~/.claude/get-shit-done/bin/gsd-tools.cjs <command> [args]`
- Key operations:
  - `state load/update/patch` — Read/write STATE.md fields atomically
  - `phase add/insert/remove/complete` — Phase lifecycle management
  - `roadmap get-phase` — Extract phase sections
  - `commit <message> --files` — Create planning commits
  - `scaffold phase-dir/context/uat` — Create directory structure
  - `verify plan-structure/references/commits` — Validation suite

**Project State Layer:**
- Purpose: Persistent project context across sessions
- Location: `.planning/` (git-tracked markdown files)
- Contains: ROADMAP.md, STATE.md, REQUIREMENTS.md, PROJECT.md, CONFIG.json
- Per-phase: `phases/01-phase-name/` contains PLAN.md (x N plans), SUMMARY.md (x N), RESEARCH.md, CONTEXT.md, VERIFICATION.md
- Used by: All agents read from here; workflows write back

**Template Layer:**
- Purpose: Pre-filled document scaffolds with configurable fields
- Location: `get-shit-done/templates/` (24 template files + codebase subdirectory)
- Contains: PLAN.md, SUMMARY.md, VERIFICATION.md, STATE.md, ROADMAP.md, PROJECT.md templates
- Codebase subdirectory: Templates for architecture/stack/testing/concerns documents
- Used by: gsd-tools.cjs `scaffold` and `template fill` commands

**Reference Layer:**
- Purpose: Configuration, calculation, and validation specifications
- Location: `get-shit-done/references/` (13 reference files)
- Contains:
  - `model-profiles.md` — Maps agent types to model tiers (quality/balanced/budget)
  - `decimal-phase-calculation.md` — Phase numbering logic
  - `tdd.md` — Test-driven development patterns within GSD
  - `verification-patterns.md` — What to validate in SUMMARY.md, artifacts, links
  - `git-integration.md` — Commit message format, branch strategy
  - `checkpoints.md` — Long-run checkpoint protocol for >30min execution
  - `planning-config.md` — Config.json schema and behavior
- Used by: Workflows reference for specifications; agents read for patterns

**Installation/Bootstrap Layer:**
- Purpose: Set up GSD hooks and commands in user's Claude Code/.claude/ directory
- Location: `bin/install.js`, `hooks/gsd-*.js`
- Contains: Installation logic for multiple runtimes (Claude Code, OpenCode, Gemini)
- Creates: `.claude/get-shit-done/` symlink/copy, hooks for status line and update checking
- Used by: First-time setup via `npx get-shit-done-cc@latest`

## Data Flow

**Workflow Initiation:**

1. User runs `/gsd:command-name` with arguments in Claude Code
2. Command file (`commands/gsd/COMMAND.md`) defines entry point
3. Command file references workflow file via `@~/.claude/get-shit-done/workflows/WORKFLOW.md`
4. Claude Code loads and executes workflow

**Orchestration Flow:**

1. Workflow calls `init` command via gsd-tools.cjs to load all context
   ```bash
   INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs init plan-phase "$PHASE")
   ```
   → Returns JSON: models, config, file paths, computed values

2. Workflow validates inputs, creates/checks directories, reads key files
   ```bash
   PHASE_INFO=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs roadmap get-phase "$PHASE")
   ```
   → Returns JSON: phase_number, phase_name, goal, success_criteria

3. Workflow spawns specialized agent(s) with focused context
   ```bash
   Task(subagent_type="gsd-researcher", model="claude-opus-4-6")
   # Agent gets fresh 200k context
   ```

4. Agent executes work, reads/writes project files directly
   - Reads: ROADMAP.md, REQUIREMENTS.md, PROJECT.md, CONTEXT.md
   - Writes: RESEARCH.md, PLAN.md, SUMMARY.md, VERIFICATION.md
   - Commits: via `git add && git commit`

5. Workflow collects agent output, validates, commits planning docs
   ```bash
   node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "docs: plan phase 1" --files .planning/phases/01-phase/PLAN*.md
   ```

6. Workflow presents results, offers next steps

**State Management Flow:**

1. Projects begin with `/gsd:new-project`
   - Creates `.planning/STATE.md`, `ROADMAP.md`, `REQUIREMENTS.md`, `PROJECT.md`
   - Creates phase 1 directory structure

2. During execution, STATE.md is the checkpoint
   ```bash
   node ~/.claude/get-shit-done/bin/gsd-tools.cjs state update current_phase "1"
   node ~/.claude/get-shit-done/bin/gsd-tools.cjs state record-metric --phase 1 --plan 01-01 --duration 12min
   ```

3. ROADMAP.md tracks phase-level progress (checklist)
   - Phase header shows: `- [x] Phase 1: Name` (checked when complete)

4. Per-phase summaries accumulate in SUMMARY.md
   - One SUMMARY per plan
   - Tracks what was built, decisions made, time spent

5. Milestones compress history: `/gsd:complete-milestone` archives all completed phases into MILESTONES.md

**State Management:**

- **Transient during execution:** Phase directory PLAN.md, per-task checkpoint marker
- **Persistent across sessions:** All .planning/*.md files committed to git
- **Recovery:** Agents can resume from checkpoint; STATE.md guides position
- **Rollback:** Incomplete phases can be reverted; git history is audit trail

## Key Abstractions

**Phase:**
- Purpose: Represent a unit of work with goal, success criteria, and plans
- Examples: `Phase 1: Core API Structure`, `Phase 2.1: Urgent Fix`
- Pattern: Phases are numbered (1, 2, 3) or decimal (2.1) for insertions
- Stored in: ROADMAP.md (metadata) + `phases/01-phase-name/` (work files)

**Plan:**
- Purpose: Executable task breakdown for Claude with objective, context, tasks, success criteria
- Examples: `01-01: Database schema`, `01-02: API endpoints`
- Pattern: One PLAN.md = one execution session (~50% context used)
- Stored in: `.planning/phases/XX-name/PLAN-YY-WAVE-Z.md`
- Used by: gsd-executor reads and executes

**Wave:**
- Purpose: Group plans that can execute in parallel (dependency-ordered)
- Pattern: Plans in Wave 1 have no dependencies; Wave 2 can assume Wave 1 complete, etc.
- Example: Wave 1 might have [schema, migrations], Wave 2 might have [endpoints, validation]
- Computed by: gsd-planner during planning

**Must-Have:**
- Purpose: Non-negotiable requirements derived backward from phase goal
- Pattern: 5-10 specific artifacts/behaviors that MUST be true for phase success
- Example: For "API structure" phase: "Auth middleware installed and tested", "Error handling middleware present"
- Used by: gsd-executor validates these exist; gsd-verifier checks completeness

**Checkpoint:**
- Purpose: Named pause point in long execution (>30 min expected)
- Pattern: Marked in PLAN.md as `<!-- CHECKPOINT: name -->` after atomic work complete
- Usage: If execution interrupted, resume from checkpoint = skip already-done work
- Stored in: Phase STATE.md records checkpoint reached

**Discovery:**
- Purpose: Agent's initial understanding pass (reading existing code, architecture)
- Pattern: Distinct from planning; agents run discovery BEFORE creating plans
- Example: gsd-planner discovery reads existing codebase structure, finds patterns
- Used by: Speeds iteration and prevents broken assumptions

## Entry Points

**Manual Entry:**
- Location: User runs `/gsd:COMMAND` in Claude Code runtime
- Triggers: Command markdown file loaded from installed `.claude/get-shit-done/commands/gsd/`
- Responsibilities: Parse arguments, display workflow banner, invoke workflow

**Programmatic Entry:**
- Location: Workflows spawn agents via Task tool with `subagent_type="gsd-NAME"`
- Triggers: Agent prompt loaded from installed `.claude/get-shit-done/agents/gsd-NAME.md`
- Responsibilities: Load full context, execute domain work, write results, report status

**Automated Entry (Hooks):**
- Location: `hooks/gsd-check-update.js` (post-command hook)
- Triggers: After command execution, checks for new GSD versions
- Responsibilities: Notify user if update available

## Error Handling

**Strategy:** Fail loudly with recovery paths; prefer early validation over mid-execution errors

**Patterns:**

1. **Phase Validation (Early):**
   ```bash
   PHASE_INFO=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs roadmap get-phase "$PHASE")
   if [ "$FOUND" != "true" ]; then
     # Error: phase not in ROADMAP → show available phases, exit
   fi
   ```

2. **State Consistency (Guard):**
   - Workflows check STATE.md exists before proceeding
   - If missing but `.planning/` exists: offer to reconstruct or continue
   - If both missing: error → redirect to `/gsd:new-project`

3. **Verification Loop (Iteration):**
   - gsd-planner creates PLAN.md
   - gsd-plan-checker validates structure + referenced files
   - If validation fails: agent revises, max 3 iterations
   - If still fails after 3: error with details → manual review needed

4. **Execution Deviation (Automatic Recovery):**
   - gsd-executor detects if task completed differently than planned
   - Records deviation in SUMMARY.md task entry
   - Continues to next task (non-blocking)
   - Flag in SUMMARY: tasks with deviations reviewed in verification

5. **Secret Detection (Pre-commit):**
   - Workflows scan generated .planning/ files for credential patterns before git commit
   - If found: SECURITY ALERT, pause, require manual review

6. **Missing Files (Agent Reference):**
   - Agents reference files with `@path` syntax
   - Orchestrator validates all refs exist before spawning
   - If missing: error with context about what's needed

## Cross-Cutting Concerns

**Logging:**
- Approach: No centralized logger; agents output directly to stdout
- Pattern: ASCII banners for section headers, `echo` for progress, bash commands produce output
- Visibility: User sees agent thinking/progress in real-time

**Validation:**
- Approach: Multi-phase validation (early parse, mid-workflow, post-execution)
- Pattern: gsd-tools.cjs `verify` subcommands handle file structure validation
- Location: `references/verification-patterns.md` defines what's valid

**Authentication:**
- Approach: Token stored in user's config (Claude Code handles secure storage)
- Pattern: Model selection via profile resolution; no explicit auth prompts
- Location: `get-shit-done/references/model-profiles.md` defines model access

**Branching:**
- Approach: Optional git branches for phases or milestones
- Pattern: Configured in CONFIG.json `branching_strategy`
- Options: "none" (main), "phase" (feature/phase-X), "milestone" (feature/v1.0)
- Implementation: Workflow calls `git checkout -b` before spawning agents

**Parallelization:**
- Approach: Plans in same wave can run in parallel (multi-agent)
- Pattern: Workflow spawns multiple Task tools simultaneously with `run_in_background=true`
- Coordination: Agents wait for wave completion before proceeding
- Configured: CONFIG.json `parallelization` flag

---

*Architecture analysis: 2026-02-15*
