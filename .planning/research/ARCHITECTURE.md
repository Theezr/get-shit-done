# GSD Codebase Architecture & Migration Map

## File Structure Overview

### Current System (Workflows + Agents + Tools)

```
get-shit-done/
├── bin/
│   ├── install.js                    # NPM installer (multi-runtime support)
│   └── gsd-tools.cjs                 # Core CLI utilities (55k+ lines)
│
├── get-shit-done/                    # Installed to ~/.claude/get-shit-done/
│   ├── workflows/                    # 30 workflow files (orchestrators)
│   │   ├── help.md
│   │   ├── progress.md
│   │   ├── plan-phase.md
│   │   ├── execute-phase.md
│   │   ├── new-project.md
│   │   └── ...
│   │
│   ├── templates/                    # 15+ templates
│   │   ├── summary.md
│   │   ├── milestone.md
│   │   ├── project.md
│   │   └── ...
│   │
│   ├── references/                   # 14 reference docs
│   │   ├── checkpoints.md
│   │   ├── planning-config.md
│   │   ├── model-profiles.md
│   │   └── ...
│   │
│   └── bin/
│       └── gsd-tools.cjs             # Symlinked from root
│
├── commands/gsd/                     # 29 command files (Claude Code slash commands)
│   ├── help.md
│   ├── progress.md
│   ├── plan-phase.md
│   └── ...
│
└── agents/                           # 11 agent files (spawned via Task tool)
    ├── gsd-executor.md
    ├── gsd-planner.md
    ├── gsd-phase-researcher.md
    ├── gsd-debugger.md
    └── ...
```

### Purpose of Each Directory

**`workflows/`** — Orchestration logic installed to `~/.claude/get-shit-done/workflows/`. Referenced via `@~/.claude/get-shit-done/workflows/X.md` syntax. These are NOT directly invoked by users.

**`commands/gsd/`** — User-facing slash commands (`/gsd:help`, `/gsd:progress`). These load corresponding workflow files and execute them. Installed to `~/.claude/commands/gsd/`.

**`agents/`** — Sub-agent personalities spawned via Task() calls. Each has role, tools, execution flow. Installed to `~/.claude/agents/`.

**`templates/`** — Markdown/JSON scaffolding for SUMMARY.md, PLAN.md, etc.

**`references/`** — Domain knowledge documents (@-referenced in workflows/agents).

**`bin/`** — Installation script and gsd-tools.cjs CLI.

## Command Registration

Commands become slash commands through Claude Code's command directory structure:

1. **Command File Location**: `commands/gsd/X.md`
2. **Invocation**: User types `/gsd:X`
3. **Execution**: Claude reads `commands/gsd/X.md`, which references `@~/.claude/get-shit-done/workflows/X.md`
4. **Frontmatter**: YAML metadata (name, description, tools)
5. **Installation**: `bin/install.js` copies files to `~/.claude/` directories during `npx get-shit-done-cc`

**Flow**: `/gsd:help` → `commands/gsd/help.md` → `@~/.claude/get-shit-done/workflows/help.md` → outputs reference content

## Command Inventory

### Simple Commands (no agents, direct execution)

| Command | Purpose | Complexity |
|---------|---------|------------|
| `/gsd:help` | Display reference documentation | TRIVIAL - just outputs static content |
| `/gsd:join-discord` | Show Discord invite link | TRIVIAL - static output |
| `/gsd:set-profile <profile>` | Quick-switch model profile | SIMPLE - updates config.json |
| `/gsd:pause-work` | Create continuation context | SIMPLE - writes .continue-here file |
| `/gsd:list-phase-assumptions <N>` | Show planning assumptions | SIMPLE - conversational output |

### Medium Commands (1-2 agents, light orchestration)

| Command | Agents Spawned | Purpose |
|---------|---------------|---------|
| `/gsd:progress` | None (pure orchestration) | Check status, route to next action |
| `/gsd:resume-work` | None | Restore session context |
| `/gsd:add-todo [desc]` | None | Capture idea as pending todo |
| `/gsd:check-todos [area]` | None | List/select/route todo |
| `/gsd:add-phase <desc>` | None | Append phase to roadmap |
| `/gsd:insert-phase <N> <desc>` | None | Insert decimal phase |
| `/gsd:remove-phase <N>` | None | Delete + renumber phases |
| `/gsd:settings` | None | Interactive config editor |
| `/gsd:update` | None | Version check + install |
| `/gsd:discuss-phase <N>` | None (questioning) | Capture user vision |
| `/gsd:research-phase <N>` | gsd-phase-researcher (1) | Ecosystem research |

### Complex Commands (3+ agents, heavy orchestration)

| Command | Agents Spawned | Max Agent Depth |
|---------|---------------|-----------------|
| `/gsd:new-project` | gsd-project-researcher (4 parallel), gsd-roadmapper (1), gsd-plan-checker (opt) | 6 total |
| `/gsd:new-milestone` | Same as new-project | 6 total |
| `/gsd:map-codebase` | gsd-codebase-mapper (7 parallel) | 7 total |
| `/gsd:plan-phase <N>` | gsd-phase-researcher (1), gsd-planner (1), gsd-plan-checker (1), planner-revision (up to 3) | Up to 6 |
| `/gsd:execute-phase <N>` | gsd-executor (1 per plan), continuation agents (dynamic) | Unbounded |
| `/gsd:execute-plan <phase-plan>` | gsd-executor (1), continuation agents (dynamic) | Unbounded |
| `/gsd:quick` | gsd-planner (1), gsd-executor (1) | 2 total |
| `/gsd:debug [issue]` | gsd-debugger (1), continuation agents | Unbounded |
| `/gsd:verify-work <N>` | gsd-verifier (1 per deliverable) | Up to 10+ |
| `/gsd:audit-milestone` | gsd-verifier (1), gsd-integration-checker (1) | 2 total |
| `/gsd:plan-milestone-gaps` | gsd-planner (1) | 1 total |
| `/gsd:complete-milestone <ver>` | None (archival) | 0 |

## Agent Inventory

### Agent: gsd-executor
- **Role**: Execute PLAN.md files atomically with per-task commits
- **Used by**: execute-phase, execute-plan, quick
- **Complexity**: HIGH - handles TDD, checkpoints, deviation rules, auth gates
- **Key Features**: Deviation auto-fix, checkpoint protocol, continuation handling
- **Files**: 420 lines

### Agent: gsd-planner
- **Role**: Create executable PLAN.md files from phase goals
- **Used by**: plan-phase, plan-milestone-gaps, quick
- **Complexity**: VERY HIGH - dependency graphs, wave assignment, must_haves derivation
- **Key Features**: Goal-backward methodology, vertical slicing, TDD detection
- **Files**: 1160 lines

### Agent: gsd-phase-researcher
- **Role**: Research "how to implement this phase"
- **Used by**: plan-phase (optional)
- **Complexity**: MEDIUM - ecosystem research, Context7 integration
- **Files**: ~150 lines (estimated)

### Agent: gsd-project-researcher
- **Role**: Deep domain research for new projects
- **Used by**: new-project, new-milestone (spawns 4 parallel)
- **Complexity**: MEDIUM - parallel research synthesis
- **Files**: ~200 lines (estimated)

### Agent: gsd-research-synthesizer
- **Role**: Merge parallel research findings
- **Used by**: new-project, new-milestone
- **Complexity**: LOW - consolidation only
- **Files**: ~100 lines (estimated)

### Agent: gsd-roadmapper
- **Role**: Create ROADMAP.md from requirements
- **Used by**: new-project, new-milestone
- **Complexity**: HIGH - phase breakdown, dependency analysis
- **Files**: ~400 lines (estimated)

### Agent: gsd-codebase-mapper
- **Role**: Explore existing codebase and document
- **Used by**: map-codebase (spawns 7 parallel)
- **Complexity**: MEDIUM - file tree analysis
- **Files**: ~200 lines (estimated)

### Agent: gsd-debugger
- **Role**: Systematic debugging with persistent state
- **Used by**: debug
- **Complexity**: MEDIUM - investigation cycles
- **Files**: ~250 lines (estimated)

### Agent: gsd-verifier
- **Role**: UAT validation + gap diagnosis
- **Used by**: verify-work, audit-milestone
- **Complexity**: MEDIUM - goal-backward verification
- **Files**: ~300 lines (estimated)

### Agent: gsd-plan-checker
- **Role**: Validate PLAN.md files for completeness
- **Used by**: plan-phase (optional)
- **Complexity**: MEDIUM - structured validation
- **Files**: ~200 lines (estimated)

### Agent: gsd-integration-checker
- **Role**: Verify cross-phase wiring at milestone end
- **Used by**: audit-milestone
- **Complexity**: MEDIUM - dependency trace
- **Files**: ~200 lines (estimated)

## gsd-tools.cjs API Surface

### Categories of Functions (100+ total)

**Initialization (compound commands)**
- `init execute-phase <phase>` — All context for execute workflow
- `init plan-phase <phase>` — All context for plan workflow
- `init new-project` — All context for new-project workflow
- `init progress` — All context for progress workflow
- 10+ other init commands

**State Management**
- `state load` — Load config + STATE.md
- `state advance-plan` — Increment plan counter
- `state update-progress` — Recalculate progress bar
- `state add-decision` — Add decision to STATE.md
- `state record-metric` — Record execution metrics
- 8+ other state commands

**Roadmap Operations**
- `roadmap get-phase <N>` — Extract phase from ROADMAP.md
- `roadmap analyze` — Full parse with disk status
- `roadmap update-plan-progress <N>` — Update progress table

**Phase Operations**
- `phase add <desc>` — Append phase to roadmap
- `phase insert <after> <desc>` — Insert decimal phase
- `phase remove <N>` — Delete + renumber
- `phase complete <N>` — Mark done
- `phase next-decimal <N>` — Calculate decimal phase number

**Verification Suite**
- `verify plan-structure <file>` — Check PLAN.md structure
- `verify phase-completeness <N>` — Check summaries exist
- `verify artifacts <file>` — Check must_haves.artifacts
- `verify key-links <file>` — Check must_haves.key_links
- `verify commits <h1> <h2>...` — Batch verify commit hashes

**Frontmatter CRUD**
- `frontmatter get <file>` — Extract YAML as JSON
- `frontmatter set <file> --field <k> --value <v>` — Update field
- `frontmatter merge <file> --data '{json}'` — Merge JSON
- `frontmatter validate <file> --schema <plan|summary>` — Validate

**Template Operations**
- `template fill summary --phase N --plan M` — Pre-filled SUMMARY.md
- `template fill plan --phase N --type execute` — Pre-filled PLAN.md
- `template fill verification --phase N` — Pre-filled VERIFICATION.md

**Utilities**
- `generate-slug <text>` — Convert to kebab-case
- `current-timestamp [format]` — Get ISO/filename timestamp
- `commit <msg> --files f1 f2` — Git commit planning docs
- `history-digest` — Aggregate all SUMMARY.md data
- `summary-extract <path> --fields one_liner,decisions` — Parse SUMMARY
- `state-snapshot` — Structured STATE.md parse
- `websearch <query>` — Brave API search (optional)

## Dependencies Between Commands

### Command Chains (typical flows)

**New Project Flow**
```
/gsd:new-project
  → spawns gsd-project-researcher (4x parallel)
  → spawns gsd-roadmapper (1x)
  → creates PROJECT.md, REQUIREMENTS.md, ROADMAP.md
  → suggests: /gsd:plan-phase 1
```

**Planning Flow**
```
/gsd:plan-phase 1
  → spawns gsd-phase-researcher (optional)
  → spawns gsd-planner
  → spawns gsd-plan-checker (optional)
  → revision loop (up to 3x)
  → creates PLAN.md files
  → suggests: /gsd:execute-phase 1
```

**Execution Flow**
```
/gsd:execute-phase 1
  → waves 1-N in parallel
  → each wave spawns gsd-executor per plan
  → continuation agents on checkpoints
  → creates SUMMARY.md files
  → updates STATE.md, ROADMAP.md
  → suggests: /gsd:verify-work 1 OR /gsd:progress
```

**Continuation Flow**
```
User: /clear
User: /gsd:progress
  → reads STATE.md, ROADMAP.md
  → detects position
  → offers next action (execute/plan/complete)
```

### Direct Dependencies

| Command | Calls These Commands | Via |
|---------|---------------------|-----|
| execute-phase | execute-plan (implicit) | Orchestrator loop |
| progress | (suggests next command) | User routing |
| new-project | (suggests plan-phase) | User routing |
| verify-work | plan-phase --gaps (if failures) | Conditional spawn |
| audit-milestone | plan-milestone-gaps (if gaps) | User routing |

### File Dependencies

All commands depend on:
- `.planning/config.json` — settings
- `.planning/STATE.md` — position tracking
- `.planning/ROADMAP.md` — phase structure
- `gsd-tools.cjs` — all utility functions

## Migration Priority

### Phase 1: Infrastructure (Critical Path)
1. **gsd-tools.cjs → skill helpers** — Extract core utilities into standalone modules
   - Why: All workflows depend on this
   - Complexity: HIGH (5500+ lines, 100+ functions)
   - Strategy: Create `/helpers/` directory, modularize by domain

2. **Template system** — Keep as-is or migrate to skill assets/
   - Why: SUMMARY.md, PLAN.md scaffolding is universal
   - Complexity: LOW (already structured)
   - Strategy: Copy to skill's `templates/` directory

3. **Reference docs** — Keep as-is or migrate to skill references/
   - Why: Foundational knowledge (@-referenced everywhere)
   - Complexity: LOW (14 files, no logic)
   - Strategy: Copy to skill's `references/` directory

### Phase 2: Simple Commands (Quick Wins)
4. **help.md** → Standalone skill
   - Why: Zero dependencies, pure output
   - Complexity: TRIVIAL
   - Agents: 0
   - Files: 1

5. **set-profile.md** → Standalone skill
   - Why: Simple config update
   - Complexity: SIMPLE
   - Agents: 0
   - Files: 1

6. **join-discord.md** → Standalone skill
   - Why: Static output
   - Complexity: TRIVIAL
   - Agents: 0
   - Files: 1

### Phase 3: Medium Commands (Orchestration Only)
7. **progress.md** → Standalone skill with helper imports
   - Why: Pure orchestration, no agents
   - Complexity: MEDIUM
   - Agents: 0
   - Dependencies: gsd-tools.cjs (state, roadmap, summary parsing)
   - Files: 1 + helpers

8. **resume-work.md** → Standalone skill
   - Why: Simple context restoration
   - Complexity: SIMPLE
   - Agents: 0
   - Files: 1

9. **add-phase.md, insert-phase.md, remove-phase.md** → Phase management skill
   - Why: Related operations, group into single skill
   - Complexity: MEDIUM (remove-phase is complex due to renumbering)
   - Agents: 0
   - Files: 1 skill covering 3 operations

10. **add-todo.md, check-todos.md** → Todo management skill
    - Why: Related operations
    - Complexity: SIMPLE
    - Agents: 0
    - Files: 1 skill covering 2 operations

### Phase 4: Agent-Based Commands (High Value, High Complexity)
11. **plan-phase.md** → Skill with sub-agents
    - Why: Core workflow, high usage
    - Complexity: VERY HIGH
    - Agents: 3-6 (researcher, planner, checker, revisions)
    - Strategy: Skill orchestrates, agents preserved as separate skill files
    - Files: 1 orchestrator + 3 agent skills

12. **execute-phase.md + execute-plan.md** → Execution skill
    - Why: Core workflow, high usage
    - Complexity: VERY HIGH
    - Agents: 1-N (executor + continuations)
    - Strategy: Main skill + executor sub-skill
    - Files: 1 orchestrator + 1 agent skill

13. **quick.md** → Quick task skill
    - Why: Simplified plan+execute flow
    - Complexity: MEDIUM
    - Agents: 2 (planner, executor)
    - Strategy: Reuse planner + executor from #11 and #12
    - Files: 1 orchestrator

### Phase 5: Advanced Workflows (Lower Frequency)
14. **new-project.md** → Project initialization skill
    - Why: High complexity, but infrequent use
    - Complexity: VERY HIGH
    - Agents: 6+ (4 researchers, 1 roadmapper, 1 checker)
    - Files: 1 orchestrator + 3 agent skills

15. **debug.md** → Debugging skill
    - Why: Persistent state handling is complex
    - Complexity: HIGH
    - Agents: 1+ (debugger + continuations)
    - Files: 1 orchestrator + 1 agent skill

16. **verify-work.md** → UAT skill
    - Why: Goal-backward verification
    - Complexity: MEDIUM
    - Agents: 1-N (verifier per deliverable)
    - Files: 1 orchestrator + 1 agent skill

17. **map-codebase.md** → Codebase mapping skill
    - Why: Brownfield project support
    - Complexity: MEDIUM
    - Agents: 7 parallel (mapper instances)
    - Files: 1 orchestrator + 1 agent skill

### Phase 6: Milestone Operations (Rare but Important)
18. **new-milestone.md** → Reuse new-project infrastructure
19. **audit-milestone.md** → Audit skill
20. **plan-milestone-gaps.md** → Reuse planner
21. **complete-milestone.md** → Archival skill

## What to Eliminate

### Redundant Files
- **commands/gsd/*.md** — ELIMINATE after migration
  - Reason: Skills replace slash command files
  - Action: Delete entire `commands/gsd/` directory post-migration

- **workflows/*.md** (partially) — CONSOLIDATE
  - Reason: Many workflows are thin wrappers that just reference agents
  - Action: Merge simple orchestration into skill SKILL.md files

### Duplicate Logic
- **Workflow + Command duplication** — Each command has two files (command/X.md + workflow/X.md)
  - Reason: Historical artifact of Claude Code command structure
  - Action: Single SKILL.md per feature post-migration

### Installation Complexity
- **bin/install.js** — SIMPLIFY
  - Reason: Multi-runtime support (Claude/OpenCode/Gemini) adds complexity
  - Action: Skills auto-install via Claude's skill system

### Configuration Fragmentation
- **Multiple config files** — config.json has nested structure
  - Action: Flatten or consolidate

## What to Preserve

### Critical Infrastructure

**1. gsd-tools.cjs** — ESSENTIAL (100% preserve)
- All workflows depend on this
- 100+ utility functions
- Strategy: Extract into modular helpers for skills

**2. Templates** — PRESERVE (100%)
- SUMMARY.md, PLAN.md, MILESTONE.md templates
- Universal across all workflows
- Strategy: Copy to skill `templates/` directory

**3. References** — PRESERVE (100%)
- checkpoints.md, planning-config.md, model-profiles.md, etc.
- Domain knowledge referenced via @-syntax
- Strategy: Copy to skill `references/` directory

**4. Agents** — PRESERVE (100%)
- gsd-executor.md, gsd-planner.md, gsd-verifier.md, etc.
- These are the personalities/roles
- Strategy: Convert to sub-skills or preserve as agent files

### Core Workflows to Preserve As-Is

**Progress tracking** — STATE.md + ROADMAP.md integration is robust
**Phase management** — Decimal phase insertion/removal logic is solid
**Checkpoint protocol** — Auth gates, human-verify, decision checkpoints
**Deviation rules** — Auto-fix vs ask-user heuristics
**Goal-backward methodology** — must_haves derivation in planner

### File Formats to Preserve

**.planning/ directory structure** — User-facing, well-documented
**PLAN.md XML task format** — `<task type="auto">` structure
**SUMMARY.md frontmatter** — Dependency tracking, metrics
**Frontmatter schemas** — Validation logic in gsd-tools.cjs

## Migration Strategy Recommendations

### Approach 1: Incremental (RECOMMENDED)
1. Start with simple commands (help, set-profile) as proof-of-concept
2. Extract gsd-tools.cjs into modular helpers
3. Migrate medium commands (progress, phase management)
4. Tackle complex agent-based workflows (plan-phase, execute-phase)
5. Ship each batch independently, maintain backward compatibility

**Pros**: Lower risk, faster feedback, can pause/adjust
**Cons**: Longer timeline, maintain dual systems temporarily

### Approach 2: Big Bang
1. Design complete skill structure upfront
2. Migrate all 30 commands in one pass
3. Ship entire system at once
4. Deprecate old system immediately

**Pros**: Clean cut, no dual maintenance
**Cons**: High risk, long dark period, hard to test incrementally

### Approach 3: Hybrid (ALTERNATIVE)
1. Migrate infrastructure first (tools, templates, references)
2. Create skill versions of top 5 commands (help, progress, plan-phase, execute-phase, quick)
3. Ship "GSD Core" skill pack
4. Migrate remaining commands as separate skills over time
5. Deprecate old system after core is proven

**Pros**: Balances speed and safety, high-value features first
**Cons**: Requires careful dependency management

### Skill Pack Structure (Recommended for Hybrid)

```
gsd-core/
├── SKILL.md                  # Main orchestrator
├── scripts/
│   ├── init-project.py       # Extracted from workflows
│   ├── manage-state.py       # State operations
│   └── analyze-progress.py   # Progress calculations
├── helpers/
│   ├── state.js              # Extracted from gsd-tools.cjs
│   ├── roadmap.js
│   ├── phase.js
│   └── commit.js
├── templates/
│   ├── summary.md
│   ├── plan.md
│   └── milestone.md
├── references/
│   ├── checkpoints.md
│   ├── planning-config.md
│   └── model-profiles.md
└── agents/
    ├── executor-skill/
    │   └── SKILL.md
    ├── planner-skill/
    │   └── SKILL.md
    └── verifier-skill/
        └── SKILL.md
```

## Key Design Decisions for Migration

### 1. Agent Architecture
**Question**: Keep agents as separate files or embed in main skill?

**Option A**: Agents as sub-skills
- Each agent gets own SKILL.md in `agents/` subdirectory
- Main skill spawns via Task() tool
- Preserves current architecture exactly

**Option B**: Embed agents in main skill
- Agent prompts become `<agent_prompt>` sections in SKILL.md
- Spawn via Claude's native agent support
- More self-contained

**Recommendation**: Option A (sub-skills) for consistency with current system

### 2. Tool Dependency Strategy
**Question**: How to handle gsd-tools.cjs 100+ function dependency?

**Option A**: Full JavaScript execution
- Ship gsd-tools.cjs in skill's `scripts/`
- Use Bash tool to call functions
- Zero refactoring needed

**Option B**: Rewrite in Python
- Extract core logic, rewrite as skill scripts
- More portable, easier to maintain
- High upfront cost

**Option C**: Hybrid (extract only critical paths)
- Keep complex logic (frontmatter parsing, state updates)
- Inline simple logic (slug generation, timestamps)
- Balanced effort

**Recommendation**: Option A initially, migrate to Option C over time

### 3. Template Access
**Question**: How do skills access templates?

**Option A**: Bundle in skill
- Copy templates to `templates/` directory
- Self-contained
- Each skill has own copy (duplication)

**Option B**: Shared template directory
- Single source of truth
- Skills reference via @-syntax
- Requires global template location

**Recommendation**: Option A (bundled) for portability

### 4. State Management
**Question**: How to preserve .planning/ directory integration?

**Critical Requirements**:
- STATE.md must remain single source of truth
- ROADMAP.md must support concurrent updates (waves)
- Progress bar must reflect disk state (SUMMARY.md counts)

**Recommendation**: Skills continue using .planning/ structure, no changes needed

### 5. Multi-Command Skills vs Single-Command Skills
**Question**: Group related commands or keep separate?

**Grouping Candidates**:
- Phase management (add/insert/remove) → Single "phase-manager" skill
- Todo management (add/check) → Single "todo-manager" skill
- Milestone operations (new/audit/complete) → Single "milestone-manager" skill

**Keep Separate**:
- Core workflows (plan-phase, execute-phase) — too large to group
- Help/settings/update — utility commands

**Recommendation**: Group where commands share >50% logic, separate otherwise

## Success Metrics for Migration

### Technical Metrics
- [ ] All 30 commands migrated to skills
- [ ] Zero functionality regressions
- [ ] gsd-tools.cjs refactored into <10 modular helpers
- [ ] Agent architecture preserved (Task spawning works)
- [ ] Checkpoint protocol functions identically
- [ ] State/roadmap/progress tracking unchanged

### User Experience Metrics
- [ ] Installation simplified (no npx get-shit-done-cc needed)
- [ ] Commands work identically (user sees no breaking changes)
- [ ] Documentation updated (skill-based flow documented)
- [ ] Migration guide published

### Code Quality Metrics
- [ ] Eliminate 50%+ code duplication (command vs workflow files)
- [ ] Reduce installation script complexity by 70%
- [ ] Improve test coverage (skills easier to test than workflows)

## Risk Assessment

### High Risk Areas
1. **gsd-tools.cjs refactor** — 5500 lines, 100+ functions, risk of breaking state management
2. **Agent spawning** — Task() tool behavior must be preserved exactly
3. **Checkpoint protocol** — Continuation handling is complex, easy to break
4. **Parallel execution** — Wave-based parallelization relies on file ownership tracking

### Mitigation Strategies
- **Incremental migration** — Ship simple commands first, validate infrastructure
- **Extensive testing** — Test every command in isolation before go-live
- **Backward compatibility** — Keep old system running until skills proven
- **Rollback plan** — Document how to revert if migration fails

## Timeline Estimate (Hybrid Approach)

### Phase 1: Infrastructure (2-3 weeks)
- Extract gsd-tools.cjs helpers
- Setup skill pack structure
- Migrate templates + references

### Phase 2: Simple Commands (1 week)
- help, set-profile, join-discord
- Proof of concept validation

### Phase 3: Core Workflows (3-4 weeks)
- progress, plan-phase, execute-phase
- Most critical for daily use

### Phase 4: Remaining Commands (3-4 weeks)
- Todo, phase, milestone management
- Debugging, verification, auditing

### Phase 5: Testing & Documentation (1-2 weeks)
- End-to-end testing
- Migration guide
- Deprecation plan

**Total**: 10-14 weeks for complete migration

## Open Questions for Skill Designer

1. **Agent spawning**: How do skills spawn sub-agents? Via Task() tool or new mechanism?
2. **Tool access**: Can skills call arbitrary bash commands (for gsd-tools.cjs)?
3. **State persistence**: Can skills write to .planning/ directory outside skill folder?
4. **Progressive disclosure**: How do skills load heavy references (@-syntax equivalent)?
5. **Frontmatter validation**: Is there a skill-level schema validation system?
6. **Parallel execution**: Can skills spawn multiple parallel sub-skills (wave execution)?
7. **Checkpoint protocol**: Can skills pause/resume with user interaction?
8. **Version management**: How do skills handle migration/upgrades of .planning/ structure?

---

**Document Status**: DRAFT v1.0
**Author**: Claude (Project Researcher)
**Date**: 2026-02-15
**Next Steps**: Review with skill designer, answer open questions, finalize migration strategy
