---
phase: 05-convert-commands-agents-and-skills-from-gsd-to-nick-prefix
plan: 02
subsystem: agents
tags: [agents, hooks, settings, prefix-rename, nick]

# Dependency graph
requires: []
provides:
  - 11 nick-prefixed global agent files at ~/.claude/agents/ and agents/
  - 2 nick-prefixed hook scripts at ~/.claude/hooks/ and hooks/
  - Updated settings.json referencing nick-prefixed hooks
affects: [05-03, agents, hooks, settings]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "nick- prefix for user-owned agent and hook files"
    - "Preserved gsd-tools.js/get-shit-done/gsd: package references during rename"

key-files:
  created:
    - agents/nick-codebase-mapper.md
    - agents/nick-debugger.md
    - agents/nick-executor.md
    - agents/nick-integration-checker.md
    - agents/nick-phase-researcher.md
    - agents/nick-plan-checker.md
    - agents/nick-planner.md
    - agents/nick-project-researcher.md
    - agents/nick-research-synthesizer.md
    - agents/nick-roadmapper.md
    - agents/nick-verifier.md
    - hooks/nick-check-update.js
    - hooks/nick-statusline.js
  modified:
    - ~/.claude/settings.json

key-decisions:
  - "Created nick-prefixed copies in both repo (agents/, hooks/) and installed location (~/.claude/) for version control"
  - "Preserved all package-managed references: gsd-tools.js, /gsd: commands, get-shit-done paths, gsd-file-manifest.json"

patterns-established:
  - "nick- prefix convention for user-owned agent files"
  - "nick-update-check.json as cache filename for nick-prefixed hooks"

# Metrics
duration: 2min
completed: 2026-02-16
---

# Phase 05 Plan 02: Global Agents and Hooks Rename Summary

**11 global agent files and 2 hook scripts renamed from gsd- to nick- prefix with settings.json updated to reference new hook paths**

## Performance

- **Duration:** 2 min
- **Started:** 2026-02-16T12:10:56Z
- **Completed:** 2026-02-16T12:13:39Z
- **Tasks:** 2
- **Files modified:** 14 (11 agents + 2 hooks + settings.json)

## Accomplishments
- Created 11 nick-prefixed agent files with all internal gsd- agent/skill references updated to nick-
- Created 2 nick-prefixed hook files (nick-check-update.js, nick-statusline.js) with updated cache file references
- Updated ~/.claude/settings.json to reference nick-prefixed hook paths
- Preserved all package-managed references (gsd-tools.js, /gsd: commands, get-shit-done directory paths)
- Hooks execute without error after renaming

## Task Commits

Each task was committed atomically:

1. **Task 1: Create nick-prefixed global agent files** - `e31d665` (feat)
2. **Task 2: Create nick-prefixed hooks and update settings.json** - `48199b4` (feat)

## Files Created/Modified
- `agents/nick-codebase-mapper.md` - Global codebase mapper agent with nick- prefix
- `agents/nick-debugger.md` - Global debugger agent with nick- prefix
- `agents/nick-executor.md` - Global executor agent with nick- prefix
- `agents/nick-integration-checker.md` - Global integration checker agent with nick- prefix
- `agents/nick-phase-researcher.md` - Global phase researcher agent with nick- prefix
- `agents/nick-plan-checker.md` - Global plan checker agent with nick- prefix
- `agents/nick-planner.md` - Global planner agent with nick- prefix
- `agents/nick-project-researcher.md` - Global project researcher agent with nick- prefix
- `agents/nick-research-synthesizer.md` - Global research synthesizer agent with nick- prefix
- `agents/nick-roadmapper.md` - Global roadmapper agent with nick- prefix
- `agents/nick-verifier.md` - Global verifier agent with nick- prefix
- `hooks/nick-check-update.js` - Update check hook with nick-update-check.json cache
- `hooks/nick-statusline.js` - Statusline hook with nick-update-check.json read
- `~/.claude/settings.json` - Hook command paths updated to nick- prefix

## Decisions Made
- Created nick-prefixed copies in both repo (agents/, hooks/) and installed location (~/.claude/agents/, ~/.claude/hooks/) to maintain version control alongside active installation
- Preserved all package-managed references: gsd-tools.js, /gsd: commands, get-shit-done paths, gsd-file-manifest.json unchanged

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Created repo copies alongside installed copies**
- **Found during:** Task 1
- **Issue:** Plan specified only ~/.claude/agents/ as target, but repo also contains agent source files in agents/ directory for version control
- **Fix:** Created nick-prefixed copies in both locations (repo agents/ directory and ~/.claude/agents/)
- **Files modified:** agents/nick-*.md (11 files in repo)
- **Verification:** Both locations have 11 nick-prefixed files
- **Committed in:** e31d665

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Required for version control consistency. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All global agents have nick- prefix copies ready
- Hooks functional with nick- prefix
- settings.json references updated
- Ready for plan 03 (skill/workflow/command renames)

## Self-Check: PASSED

All 13 repo files verified present. Both commit hashes (e31d665, 48199b4) confirmed in git log. All 11 installed agent files, 2 installed hooks, and settings.json references verified.

---
*Phase: 05-convert-commands-agents-and-skills-from-gsd-to-nick-prefix*
*Completed: 2026-02-16*
