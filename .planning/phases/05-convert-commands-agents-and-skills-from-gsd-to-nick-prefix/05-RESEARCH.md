# Phase 5: Convert commands, agents, and skills from gsd- to nick- prefix - Research

**Researched:** 2026-02-16
**Domain:** File renaming, string replacement, Claude Code configuration (skills, commands, agents, hooks)
**Confidence:** HIGH

## Summary

This phase is a prefix renaming/personalization effort. The user's GSD workflow has artifacts spread across four locations: (1) the repo source tree (`skills/`), (2) the installed user skills at `~/.claude/skills/`, (3) the global agents at `~/.claude/agents/`, and (4) the gsd commands at `~/.claude/commands/gsd/`. The "gsd-" prefix appears in directory names, file names, YAML frontmatter `name:` fields, `subagent_type` values in Task() calls, command references like `/gsd:plan-phase`, agent file read paths like `~/.claude/skills/gsd-plan-phase/references/agents/gsd-planner.md`, and the `MODEL_PROFILES` table inside `gsd-tools.js`.

The existing `~/.claude/commands/nick/` directory already has 7 independent commands (plan, execute, verify, review, generate-workflow, update-workflow, test-flow) that use the `.pipeline/` directory and do NOT reference gsd-tools.js at all. These are a separate system. The gsd commands use `.planning/` with gsd-tools.js. The phase should create nick-prefixed COPIES of the gsd skills/agents that reference nick paths, NOT merge with the existing nick commands which serve a different purpose.

**Primary recommendation:** Treat this as a systematic find-and-replace operation across well-defined file sets, split by location (repo skills, installed skills, agents, hooks, settings, gsd-tools.js MODEL_PROFILES). The existing `~/.claude/commands/nick/` commands are an unrelated pipeline system and should be LEFT ALONE -- the gsd commands should get nick-prefixed skill equivalents, not be merged into the existing nick commands.

## Inventory of Artifacts Requiring Conversion

### Layer 1: Repo Source Tree (skills/)

These are version-controlled in the project repo at `/Users/nick/Documents/git/get-shit-done/skills/`.

| Current Directory | New Directory | Files Inside |
|---|---|---|
| `skills/gsd-execute-phase/` | `skills/nick-execute-phase/` | SKILL.md, references/agents/gsd-executor.md, gsd-executor-checkpoints.md, gsd-executor-tdd.md, gsd-verifier.md |
| `skills/gsd-plan-phase/` | `skills/nick-plan-phase/` | SKILL.md, references/agents/gsd-phase-researcher.md, gsd-plan-checker.md, gsd-planner.md |
| `skills/gsd-verify-work/` | `skills/nick-verify-work/` | SKILL.md, references/agents/gsd-browser-tester.md |
| `skills/gsd-review/` | `skills/nick-review/` | SKILL.md, references/agents/gsd-code-reviewer.md |

**Total: 4 directories, 4 SKILL.md files, 10 agent definition files**

### Layer 2: Installed Skills (~/.claude/skills/)

Mirror of repo source tree, installed at `~/.claude/skills/`. Same structure as Layer 1 but some files may differ slightly (repo has newer versions from Phase 4 work).

| Current Directory | New Directory |
|---|---|
| `~/.claude/skills/gsd-execute-phase/` | `~/.claude/skills/nick-execute-phase/` |
| `~/.claude/skills/gsd-plan-phase/` | `~/.claude/skills/nick-plan-phase/` |
| `~/.claude/skills/gsd-verify-work/` | `~/.claude/skills/nick-verify-work/` |
| `~/.claude/skills/gsd-review/` | `~/.claude/skills/nick-review/` |

### Layer 3: Global Agents (~/.claude/agents/)

GSD agent files that are referenced by name in commands and SKILL.md Task() prompts.

| Current File | New File | Referenced By |
|---|---|---|
| `gsd-codebase-mapper.md` | `nick-codebase-mapper.md` | `/gsd:map-codebase` command |
| `gsd-debugger.md` | `nick-debugger.md` | `/gsd:debug` command |
| `gsd-executor.md` | `nick-executor.md` | Legacy (skills have their own copy) |
| `gsd-integration-checker.md` | `nick-integration-checker.md` | Internal workflows |
| `gsd-phase-researcher.md` | `nick-phase-researcher.md` | `/gsd:research-phase` command |
| `gsd-plan-checker.md` | `nick-plan-checker.md` | Legacy (skills have their own copy) |
| `gsd-planner.md` | `nick-planner.md` | Legacy (skills have their own copy) |
| `gsd-project-researcher.md` | `nick-project-researcher.md` | `/gsd:new-project` command |
| `gsd-research-synthesizer.md` | `nick-research-synthesizer.md` | `/gsd:new-project` command |
| `gsd-roadmapper.md` | `nick-roadmapper.md` | `/gsd:new-project` command |
| `gsd-verifier.md` | `nick-verifier.md` | Legacy (skills have their own copy) |

**Total: 11 agent files**

**Note:** Non-gsd agents (browser-tester.md, test-planner.md, workflow-composer.md, security-*.md) are NOT part of this conversion.

### Layer 4: Settings and Hooks (~/.claude/)

| File | gsd References |
|---|---|
| `~/.claude/settings.json` | Hook commands reference `gsd-check-update.js` and `gsd-statusline.js` |
| `~/.claude/hooks/gsd-check-update.js` | Filename, internal references to `gsd-update-check.json` cache |
| `~/.claude/hooks/gsd-statusline.js` | Filename, references `gsd-update-check.json` cache, displays `/gsd:update` text |
| `~/.claude/gsd-file-manifest.json` | Filename and all paths inside reference `get-shit-done/`, `commands/gsd/`, `agents/gsd-*` |

### Layer 5: gsd-tools.js (Node.js CLI)

| Location | gsd References |
|---|---|
| `~/.claude/get-shit-done/bin/gsd-tools.js` | MODEL_PROFILES keys use `gsd-` prefix (11 entries), internal `resolveModelInternal()` calls use `gsd-` agent type names, usage string says `gsd-tools` |

### Layer 6: Commands (~/.claude/commands/gsd/)

**29 command files** (plus 1 .bak) in `~/.claude/commands/gsd/`. These register as `/gsd:*` slash commands.

**IMPORTANT:** These are part of the `get-shit-done-cc` npm package. They are NOT custom user files. The npm package installs them. Converting them means forking/customizing the package commands.

### Layer 7: Workflows (~/.claude/get-shit-done/workflows/)

**27 workflow files** containing 166 occurrences of `gsd-` references. These are also npm package files.

## Architecture Patterns

### How Skills Reference Agents

Skills use hardcoded paths in Task() prompt strings:

```
Task(
  prompt="First, read ~/.claude/skills/gsd-plan-phase/references/agents/gsd-planner.md for your role..."
)
```

These paths must be updated to use `nick-` prefix:

```
Task(
  prompt="First, read ~/.claude/skills/nick-plan-phase/references/agents/nick-planner.md for your role..."
)
```

### How gsd-tools.js Maps Agent Types to Models

```javascript
const MODEL_PROFILES = {
  'gsd-planner':              { quality: 'opus', balanced: 'opus',   budget: 'sonnet' },
  'gsd-executor':             { quality: 'opus', balanced: 'sonnet', budget: 'sonnet' },
  // ... 11 entries total
};
```

The `init execute-phase` and `init plan-phase` commands call `resolveModelInternal(cwd, 'gsd-executor')` etc. SKILL.md files don't call `resolve-model` directly -- they read the model from the `INIT` JSON output. So the MODEL_PROFILES keys need renaming.

### How Commands Register as Slash Commands

Claude Code discovers commands by directory structure:
- `~/.claude/commands/gsd/plan-phase.md` -> `/gsd:plan-phase`
- `~/.claude/commands/nick/plan.md` -> `/nick:plan`

To get `/nick:plan-phase`, create: `~/.claude/commands/nick/plan-phase.md`

### Existing nick Commands are UNRELATED

The current `~/.claude/commands/nick/` commands (plan, execute, verify, review, etc.) are a SEPARATE pipeline system that uses `.pipeline/` directories. They do NOT use gsd-tools.js, skills, or the `.planning/` workflow. They should NOT be overwritten or replaced.

**Options:**
1. **New namespace:** Create `/nick-gsd:*` commands (avoid collision)
2. **Subdirectory:** Create `~/.claude/commands/nick/gsd/` for GSD-specific nick commands
3. **Replace existing nick commands:** NOT recommended -- they serve a different purpose
4. **Just rename skills/agents, keep /gsd: commands:** Partial approach -- skills get nick prefix but commands stay as /gsd

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Bulk file renaming | Manual mv + sed one-by-one | Scripted rename with content replacement | Too many files to do manually without errors |
| Cross-reference updating | Search and replace blindly | Targeted replacement with context awareness | `gsd-` appears in different contexts (path, name, MODEL_PROFILES key, display text) |
| Package vs. user file distinction | Modifying npm package files directly | Copy-and-rename pattern for user-owned files; leave package files as-is | npm updates would overwrite changes to package files |

## Common Pitfalls

### Pitfall 1: Modifying npm Package Files
**What goes wrong:** Editing files in `~/.claude/get-shit-done/` or `~/.claude/commands/gsd/` directly -- they get overwritten on `npm update`.
**Why it happens:** These files are installed by the `get-shit-done-cc` npm package.
**How to avoid:** Only modify user-owned files: skills (repo + installed), agents, hooks, settings. For commands, create new nick-prefixed files rather than modifying gsd-prefixed package files.
**Warning signs:** Files appearing in `gsd-file-manifest.json` are package-managed.

### Pitfall 2: Breaking Cross-References
**What goes wrong:** Renaming a directory without updating all internal path references causes "file not found" errors in subagent prompts.
**Why it happens:** Skills reference their own agent files by absolute path: `~/.claude/skills/gsd-plan-phase/references/agents/gsd-planner.md`
**How to avoid:** For each renamed file/directory, grep all related files for the old path and update.
**Warning signs:** Task() prompts that still contain `gsd-` paths after renaming.

### Pitfall 3: gsd-tools.js MODEL_PROFILES Mismatch
**What goes wrong:** SKILL.md calls `init plan-phase` which internally calls `resolveModelInternal(cwd, 'gsd-planner')` -- but MODEL_PROFILES keys have been changed to `nick-planner`.
**Why it happens:** gsd-tools.js is a package file. The init commands have hardcoded agent type strings.
**How to avoid:** Either: (a) leave gsd-tools.js as-is and accept gsd- MODEL_PROFILES keys, OR (b) add nick- aliases to MODEL_PROFILES alongside gsd- entries, OR (c) fork gsd-tools.js into a user-owned copy.
**Warning signs:** Model resolution returning undefined after renaming.

### Pitfall 4: Sandbox Restrictions on ~/.claude/ Writes
**What goes wrong:** Claude Code sandbox prevents writing to `~/.claude/agents/` and `~/.claude/skills/`.
**Why it happens:** Phase 04-03 noted: "Global ~/.claude/agents/ sync deferred (sandbox restriction)"
**How to avoid:** User must manually copy files, or executor runs with sandbox disabled for the copy step, or use a post-execution script.
**Warning signs:** "Operation not permitted" errors when writing to `~/.claude/`.

### Pitfall 5: Existing nick Commands Collision
**What goes wrong:** Creating `/nick:plan-phase` command when `/nick:plan` already exists for a different purpose -- user confusion.
**Why it happens:** The existing nick commands are an independent pipeline system.
**How to avoid:** Choose a clear naming convention that separates the two systems. Options: keep `/gsd:*` for phase workflow commands, or use a different subcommand naming pattern to avoid collision.

## Conversion Scope Analysis

### Files Owned by User (safe to modify)

| Location | Count | Type |
|---|---|---|
| `skills/` (repo) | 4 dirs, 14 files | Skill definitions + agent refs |
| `~/.claude/skills/` (installed) | 4 dirs, 14 files | Installed skill copies |
| `~/.claude/agents/gsd-*.md` | 11 files | Global agent definitions |
| `~/.claude/hooks/gsd-*.js` | 2 files | Hook scripts |
| `~/.claude/settings.json` | 1 file | Hook path references |

**Total user-owned: ~46 files**

### Files Owned by npm Package (risky to modify)

| Location | Count | Type |
|---|---|---|
| `~/.claude/commands/gsd/` | 29 files | Slash commands |
| `~/.claude/get-shit-done/workflows/` | 27 files | Workflow definitions |
| `~/.claude/get-shit-done/bin/gsd-tools.js` | 1 file | CLI utility |
| `~/.claude/gsd-file-manifest.json` | 1 file | Package manifest |

**Total package-owned: ~58 files (166+ gsd- occurrences in workflows alone)**

### String Replacement Categories

Each occurrence of `gsd-` falls into one of these categories:

| Category | Example | Replacement | Count (est.) |
|---|---|---|---|
| Directory name | `skills/gsd-plan-phase/` | `skills/nick-plan-phase/` | 8 dirs |
| File name | `gsd-planner.md` | `nick-planner.md` | ~25 files |
| YAML `name:` field | `name: gsd-executor` | `name: nick-executor` | ~14 |
| Path in prompt | `~/.claude/skills/gsd-plan-phase/references/agents/gsd-planner.md` | `~/.claude/skills/nick-plan-phase/references/agents/nick-planner.md` | ~30 |
| `/gsd:` command ref | `/gsd:plan-phase` | depends on namespace decision | ~50+ |
| MODEL_PROFILES key | `'gsd-planner'` | depends on approach | 11 |
| `subagent_type` value | `subagent_type="gsd-debugger"` | `subagent_type="nick-debugger"` | ~10 |
| Display text | `GSD > PHASE` | `NICK > PHASE` or keep | few |
| Cache file name | `gsd-update-check.json` | `nick-update-check.json` | 2 |

## Recommended Approach

### Phase 5 should be scoped to user-owned files only

Given that 58+ files are package-managed (commands, workflows, gsd-tools.js), the pragmatic approach is:

1. **Rename skills** (repo + installed): directory names, file names, all internal references
2. **Rename agents** (~/.claude/agents/): file names, internal `name:` fields
3. **Update hooks**: rename files, update internal references and settings.json
4. **Leave commands as /gsd:*** -- they work, and the skills system is the primary interface now
5. **Leave gsd-tools.js as-is** -- it's a package file; MODEL_PROFILES keys don't leak to users

### Alternative: Full fork approach

If the user wants to fully eliminate `gsd-` references:
1. Fork gsd-tools.js into a user-owned `nick-tools.js`
2. Create nick-prefixed commands that parallel gsd commands
3. Update all workflow files
4. This is a much larger effort (~100+ files, 300+ replacements)

## Code Examples

### Skill SKILL.md Frontmatter Rename

```yaml
# Before:
---
name: gsd-execute-phase
description: >
  Execute all plans in a phase...
---

# After:
---
name: nick-execute-phase
description: >
  Execute all plans in a phase...
---
```

### Task() Prompt Path Update

```python
# Before:
Task(
  prompt="First, read ~/.claude/skills/gsd-plan-phase/references/agents/gsd-planner.md for your role..."
)

# After:
Task(
  prompt="First, read ~/.claude/skills/nick-plan-phase/references/agents/nick-planner.md for your role..."
)
```

### Agent YAML Name Update

```yaml
# Before (in references/agents/gsd-planner.md):
---
name: gsd-planner
description: Creates executable phase plans...
---

# After (in references/agents/nick-planner.md):
---
name: nick-planner
description: Creates executable phase plans...
---
```

### Settings.json Hook Path Update

```json
// Before:
"command": "node \"/Users/nick/.claude/hooks/gsd-check-update.js\""

// After:
"command": "node \"/Users/nick/.claude/hooks/nick-check-update.js\""
```

## Open Questions

1. **Command namespace decision**
   - What we know: `/gsd:*` commands come from the npm package. `/nick:*` commands exist for a different pipeline system.
   - What's unclear: Does the user want nick-prefixed slash commands for the GSD workflow, or is renaming skills/agents sufficient?
   - Recommendation: Keep `/gsd:*` commands as-is since they're package-managed. Skills are the primary interface. If the user wants nick commands, create them as a separate plan.

2. **gsd-tools.js handling**
   - What we know: MODEL_PROFILES uses `gsd-` keys internally. The `init` subcommands call `resolveModelInternal` with hardcoded `gsd-` agent types.
   - What's unclear: Should we fork gsd-tools.js or add nick- aliases?
   - Recommendation: Leave gsd-tools.js as-is. The `gsd-` prefix in MODEL_PROFILES is an internal implementation detail that doesn't leak to users through the skills interface.

3. **Cleanup of old gsd-prefixed files**
   - What we know: After creating nick-prefixed copies, old gsd-prefixed skills/agents will still exist.
   - What's unclear: Delete the old files or keep them as fallback?
   - Recommendation: Delete old gsd-prefixed user files after verifying nick-prefixed versions work. Keep gsd commands/workflows (package-managed).

## Sources

### Primary (HIGH confidence)
- Direct filesystem inspection of `~/.claude/skills/`, `~/.claude/agents/`, `~/.claude/commands/`, `~/.claude/hooks/`, `~/.claude/settings.json`
- Direct content analysis of all SKILL.md files, agent definition files, and gsd-tools.js
- grep analysis of cross-references across all file sets

### Secondary (MEDIUM confidence)
- Phase 04-03 decision about sandbox restrictions on `~/.claude/agents/` writes

## Metadata

**Confidence breakdown:**
- Inventory completeness: HIGH - exhaustive filesystem scan of all relevant directories
- Cross-reference mapping: HIGH - grep-verified all gsd- occurrences in user-owned files
- Package vs. user ownership: HIGH - verified against gsd-file-manifest.json
- Sandbox constraints: MEDIUM - based on Phase 04-03 decision, not independently verified

**Research date:** 2026-02-16
**Valid until:** 2026-03-16 (stable domain -- file layout unlikely to change)
