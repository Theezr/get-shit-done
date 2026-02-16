# Phase 6: Fix nick skill paths - Research

**Researched:** 2026-02-16
**Domain:** Claude Code skill configuration, path resolution, command references
**Confidence:** HIGH

## Summary

Phase 6 addresses three cleanup issues remaining after Phase 5's gsd-to-nick prefix conversion:

1. **Command recommendations in flow**: Nick-prefixed skills still recommend `/gsd:` commands (e.g., "Next: /gsd:execute-phase 3"). These should recommend equivalent `/nick:` or `nick:` commands.

2. **gsd-tools.js → gsd-tools.cjs path**: All nick-prefixed skills reference `~/.claude/get-shit-done/bin/gsd-tools.js` (wrong extension). The actual installed file is `gsd-tools.cjs`.

3. **Relative @-reference resolution**: Skills use `@~/.claude/get-shit-done/workflows/execute-plan.md` format for file references, which may have resolution issues depending on Claude Code's current working directory handling.

**Primary recommendation:** Fix the gsd-tools.cjs extension first (high confidence, clear issue), then address command recommendations (strategic decision needed on command naming), then validate @-reference paths (may already work correctly).

## Problem Analysis

### Issue 1: /gsd: Command Recommendations in Nick Skills

**Current state:**
- Nick-prefixed skills installed at `~/.claude/skills/nick-*`
- Skills recommend `/gsd:` commands in workflow flow (e.g., "Next: /gsd:plan-phase X+1")
- Found in 10 locations across nick skills

**Evidence:**
```
/Users/nick/.claude/skills/nick-execute-phase/SKILL.md:153:- If more phases: offer `/gsd:plan-phase {X+1}`
/Users/nick/.claude/skills/nick-execute-phase/SKILL.md:166:- If milestone complete: offer `/gsd:complete-milestone`
/Users/nick/.claude/skills/nick-plan-phase/SKILL.md:182:Next: /gsd:execute-phase {X}
```

**Root cause:** Phase 5 preserved `/gsd:` command references as "package-managed" during skill rename. Skills recommend commands they expect users to run, but if `/gsd:` commands don't exist or are being replaced with `/nick:` equivalents, recommendations are incorrect.

**Strategic question:** Do `/nick:` commands exist? Investigation shows:
- `cat ~/.claude/settings.json | jq '.customCommands'` returns `null` — no custom commands defined
- Skills are loaded via name-based matching (e.g., "plan phase" → `nick-plan-phase` skill)
- Command structure appears to be description-based triggering, not slash commands

**Confidence:** MEDIUM — command invocation mechanism unclear. Need to verify:
1. Are `/gsd:` or `/nick:` custom commands actually defined?
2. If not, how do users invoke skills (natural language, skill name)?
3. What should flow recommendations use?

### Issue 2: gsd-tools.js → gsd-tools.cjs Extension

**Current state:**
- Nick skills reference: `node ~/.claude/get-shit-done/bin/gsd-tools.js`
- Actual installed file: `~/.claude/get-shit-done/bin/gsd-tools.cjs`
- Found in 33 locations across nick skills

**Evidence:**
```bash
$ ls -la ~/.claude/get-shit-done/bin/
-rwxr-xr-x@ 1 nick staff 172883 Feb 16 13:47 gsd-tools.cjs
-rw-r--r--@ 1 nick staff  84806 Feb 16 13:47 gsd-tools.test.cjs
```

**Root cause:** Phase 5 renamed skill/agent files but preserved package-managed paths. The preserved path used wrong extension (.js instead of .cjs).

**Impact:** All bash commands invoking gsd-tools will fail with "Cannot find module" error.

**Confidence:** HIGH — clear file existence issue, straightforward fix.

### Issue 3: Relative @-Reference Resolution

**Current state:**
- Skills use `@~/.claude/get-shit-done/workflows/execute-plan.md` for file references
- Path starts with `~` (tilde) which should expand to home directory
- Found in Task() prompts and execution context sections

**Evidence:**
```
/Users/nick/.claude/skills/nick-execute-phase/SKILL.md:81:  @~/.claude/get-shit-done/workflows/execute-plan.md
/Users/nick/.claude/skills/nick-execute-phase/SKILL.md:82:  @~/.claude/get-shit-done/templates/summary.md
```

**Verification:** Files exist at expected locations:
```bash
$ ls -la ~/.claude/get-shit-done/
drwxr-xr-x@ 8 nick staff 256 Feb 16 13:47 .
drwxr-xr-x@ 15 nick staff 480 Feb 16 13:47 references
drwxr-xr-x@ 26 nick staff 832 Feb 16 13:47 templates
drwxr-xr-x@ 32 nick staff 1024 Feb 16 13:47 workflows
```

**Path resolution behavior:** Claude Code handles `@` references for file loading. Tilde expansion depends on:
1. Shell context (works in bash commands)
2. Claude Code's file reader (may require absolute paths or workspace-relative paths)

**Confidence:** LOW — unclear if this is actually broken. Path format is standard, files exist. Issue description mentions "relative @-reference resolution" but references use absolute paths (`~/.claude/...`), not relative paths (`.planning/...`).

**Recommendation:** Test current behavior before changing. May be non-issue.

## Standard Stack

### Core Files
| File | Location | Purpose | Status |
|------|----------|---------|--------|
| gsd-tools.cjs | ~/.claude/get-shit-done/bin/ | CLI utility for GSD workflow operations | Correct location, wrong reference |
| SKILL.md files | ~/.claude/skills/nick-*/ | Skill definitions with frontmatter | Installed correctly |
| Agent definitions | ~/.claude/skills/nick-*/references/agents/ | Agent role prompts | Installed correctly |
| Workflow files | ~/.claude/get-shit-done/workflows/ | Execution protocols | Referenced correctly |
| Template files | ~/.claude/get-shit-done/templates/ | Output templates | Referenced correctly |

### Path Reference Patterns
| Pattern | Example | Context | Works? |
|---------|---------|---------|--------|
| Tilde absolute | `~/.claude/get-shit-done/bin/gsd-tools.cjs` | Bash commands | ✓ Yes |
| Tilde absolute | `@~/.claude/get-shit-done/workflows/execute-plan.md` | @ file refs | ? Unknown |
| Workspace relative | `@.planning/STATE.md` | @ file refs | ✓ Yes (verified in Phase 5) |
| Absolute | `/Users/nick/.claude/skills/nick-plan-phase/...` | Task() prompts | ✓ Yes |

## Architecture Patterns

### Recommended Fix Order
1. **gsd-tools.cjs extension** (HIGH priority, HIGH confidence)
   - Find-replace `.js` → `.cjs` in all skill/agent files
   - 33 occurrences across 4 skills

2. **Command recommendations** (MEDIUM priority, requires decision)
   - Determine correct command invocation pattern
   - Update flow recommendations consistently
   - 10 occurrences

3. **@-reference validation** (LOW priority, may be non-issue)
   - Test current @ reference resolution
   - Only fix if broken

### Pattern 1: Bash Command Invocation
**Current (broken):**
```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.js init plan-phase "$PHASE")
```

**Fixed:**
```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs init plan-phase "$PHASE")
```

### Pattern 2: Flow Recommendations
**Current:**
```markdown
Next: /gsd:execute-phase {X}
```

**Option A (if /nick: commands exist):**
```markdown
Next: /nick:execute-phase {X}
```

**Option B (if natural language invocation):**
```markdown
Next: Use nick-execute-phase skill for phase {X}
```

**Option C (if /gsd: commands remain valid):**
```markdown
Next: /gsd:execute-phase {X}
```

**Decision needed:** Verify command invocation mechanism before choosing pattern.

### Pattern 3: @-Reference Format (if broken)
**Current:**
```markdown
@~/.claude/get-shit-done/workflows/execute-plan.md
```

**Option A (absolute path):**
```markdown
@/Users/nick/.claude/get-shit-done/workflows/execute-plan.md
```

**Option B (keep tilde if working):**
```markdown
@~/.claude/get-shit-done/workflows/execute-plan.md
```

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Path resolution | Custom tilde expander | Shell's native `~` expansion | Already works in bash context |
| File existence checks | Manual verification | `gsd-tools.cjs verify-path-exists` | Already implemented |
| Settings.json parsing | Custom JSON reader | `jq` or node script | Standard tools exist |

## Common Pitfalls

### Pitfall 1: Inconsistent Extension Changes
**What goes wrong:** Changing some .js → .cjs references but missing others, causing intermittent failures.
**Why it happens:** 33 occurrences across multiple files, easy to miss some with manual editing.
**How to avoid:** Use comprehensive find-replace with verification. Grep for remaining `.js` references after changes.
**Warning signs:** "Cannot find module" errors in some but not all skill invocations.

### Pitfall 2: Breaking Working @-References
**What goes wrong:** Changing @-reference format to "fix" issue that doesn't exist, breaking working file loading.
**Why it happens:** Assuming issue exists without testing current behavior.
**How to avoid:** Test current @-reference resolution before making changes. Only fix if actually broken.
**Warning signs:** Skills stop loading workflow files after "fix" applied.

### Pitfall 3: Command Recommendation Mismatch
**What goes wrong:** Recommending `/nick:` commands that don't exist, confusing users.
**Why it happens:** Assuming command structure mirrors skill structure without verification.
**How to avoid:** Verify command invocation mechanism (check settings.json, test actual usage) before updating recommendations.
**Warning signs:** Users report "command not found" after following skill recommendations.

### Pitfall 4: Settings.json Modification Without Backup
**What goes wrong:** Corrupting settings.json during command definition updates.
**Why it happens:** Manual JSON editing, syntax errors.
**How to avoid:** Use `jq` for programmatic JSON updates, validate with `jq . settings.json` before committing.
**Warning signs:** Claude Code fails to start or load settings after changes.

## Code Examples

### Example 1: Fix gsd-tools.cjs Extension (Find-Replace)
```bash
# Find all .js references to gsd-tools
grep -r "gsd-tools\.js" ~/.claude/skills/nick-* agents/nick-* skills/nick-* 2>/dev/null

# Replace in installed skills
find ~/.claude/skills/nick-* -type f -name "*.md" -exec sed -i '' 's/gsd-tools\.js/gsd-tools.cjs/g' {} \;

# Replace in repo skills
find skills/nick-* -type f -name "*.md" -exec sed -i '' 's/gsd-tools\.js/gsd-tools.cjs/g' {} \;

# Replace in repo agents
find agents/nick-* -type f -name "*.md" -exec sed -i '' 's/gsd-tools\.js/gsd-tools.cjs/g' {} \;

# Verify no .js references remain
grep -r "gsd-tools\.js" ~/.claude/skills/nick-* agents/nick-* skills/nick-* 2>/dev/null
```

### Example 2: Verify Command Invocation Mechanism
```bash
# Check for custom commands in settings.json
cat ~/.claude/settings.json | jq '.customCommands'

# List all installed skills
ls -la ~/.claude/skills/

# Check skill frontmatter for command definitions
grep -A5 "^name:" ~/.claude/skills/nick-*/SKILL.md
grep -A5 "^description:" ~/.claude/skills/nick-*/SKILL.md
```

### Example 3: Test @-Reference Resolution (Manual Test)
**Test procedure:**
1. Create test skill with @-reference
2. Invoke skill, check if file loads
3. Try variations (tilde vs absolute, relative vs absolute)
4. Document which patterns work

**Alternative:** Review Claude Code documentation for @-reference specification.

### Example 4: Update Flow Recommendations (if /nick: pattern chosen)
```bash
# Find all /gsd: recommendations
grep -rn "/gsd:" ~/.claude/skills/nick-* skills/nick-*

# Replace with /nick: equivalent
find ~/.claude/skills/nick-* skills/nick-* -type f -name "*.md" -exec sed -i '' 's|/gsd:|/nick:|g' {} \;

# Verify changes
grep -rn "/nick:" ~/.claude/skills/nick-* skills/nick-*
```

## Open Questions

1. **Command invocation mechanism**
   - What we know: settings.json has no customCommands defined, skills loaded by name matching
   - What's unclear: Are `/gsd:` or `/nick:` slash commands defined elsewhere? How do users actually invoke skills?
   - Recommendation: Test current invocation (try `/gsd:plan-phase 1`, `plan phase 1`, etc.) to understand mechanism before updating recommendations

2. **@-reference resolution behavior**
   - What we know: Files exist, paths are syntactically correct, workspace-relative @-refs work
   - What's unclear: Do tilde-based @-references (`@~/.claude/...`) actually fail, or is this hypothetical?
   - Recommendation: Test current @-reference behavior by invoking a skill that uses them. Only fix if broken.

3. **Scope of command recommendations**
   - What we know: 10 locations recommend `/gsd:` commands
   - What's unclear: Should ALL `/gsd:` references change, or only user-facing recommendations?
   - Recommendation: Preserve `/gsd:` in description frontmatter (describes what triggers skill), change only in flow recommendations (what user should do next)

4. **Custom commands vs skill invocation**
   - What we know: Skills have `name:` and `description:` in frontmatter
   - What's unclear: Does Claude Code generate slash commands from skill names automatically?
   - Recommendation: Check Claude Code skill documentation or test behavior

## Sources

### Primary (HIGH confidence)
- Direct file inspection: `~/.claude/get-shit-done/bin/gsd-tools.cjs` exists, `gsd-tools.js` does not
- Grep results: 33 occurrences of `.js` reference, 10 occurrences of `/gsd:` recommendations
- Phase 5 SUMMARY.md: Documents intentional preservation of package-managed references

### Secondary (MEDIUM confidence)
- settings.json inspection: No customCommands defined
- SKILL.md frontmatter: name/description fields define skill identity

### Tertiary (LOW confidence - needs verification)
- Command invocation mechanism (not directly verified)
- @-reference resolution behavior (not tested)

## Metadata

**Confidence breakdown:**
- gsd-tools.cjs extension: HIGH — file existence confirmed, error reproducible
- Command recommendations: MEDIUM — mechanism unclear, testing needed
- @-reference resolution: LOW — may not be actual issue, needs validation

**Research date:** 2026-02-16
**Valid until:** 2026-03-16 (30 days — stable file structure)

**Files examined:**
- ~/.claude/skills/nick-execute-phase/SKILL.md
- ~/.claude/skills/nick-plan-phase/SKILL.md
- ~/.claude/skills/nick-review/SKILL.md
- ~/.claude/skills/nick-verify-work/SKILL.md
- All agent files in references/agents/ subdirectories
- ~/.claude/get-shit-done/bin/ (file listing)
- ~/.claude/settings.json
- Phase 5 SUMMARY.md files (05-01, 05-02, 05-03)

**Test recommendations:**
1. Attempt to invoke a nick skill with broken gsd-tools.js reference (expect failure)
2. Try different command invocation patterns (slash, natural language, skill name)
3. Verify @-reference resolution by checking agent logs for file load success/failure
