# Codebase Concerns

**Analysis Date:** 2026-02-15

## Tech Debt

**Monolithic gsd-tools.cjs file:**
- Issue: Core utility file contains 4,825 lines with 85 functions, handling CLI parsing, file I/O, YAML frontmatter parsing, git operations, and complex business logic all in one file
- Files: `get-shit-done/bin/gsd-tools.cjs`
- Impact: Extremely difficult to test, debug, or modify individual features. Changes to frontmatter parsing risk breaking state management. Adding new commands requires navigating ~85 existing functions. No module boundaries between concerns.
- Fix approach: Refactor into module structure: `gsd-tools/cli.js` (argument parsing), `gsd-tools/frontmatter.js` (YAML operations), `gsd-tools/state.js` (STATE.md mutations), `gsd-tools/git.js` (git wrapper), `gsd-tools/commands/` (individual command handlers). Import as needed rather than monolithic export.

**Implicit command routing with limited validation:**
- Issue: gsd-tools.cjs uses a massive switch statement (lines ~4640-4822) with 50+ cases but no centralized command registry or validation. Missing arguments fail silently with generic error messages.
- Files: `get-shit-done/bin/gsd-tools.cjs` (main switch statement, loadConfig, MODEL_PROFILES)
- Impact: Users get vague errors like "phase identifier required" without context about which command was called. New command additions require editing the monolithic switch statement. No discoverability of available subcommands without reading code.
- Fix approach: Create command registry object mapping command names to handler objects with { handler, args: ['required', 'args'], optionalArgs: [] }. Use registry to validate, route, and auto-generate help text.

**String-based YAML frontmatter parsing:**
- Issue: Custom regex-based YAML parser (lines ~253-326, ~328-399) written from scratch without schema validation. Handles nested objects, arrays, and indentation manually using a stack-based approach.
- Files: `get-shit-done/bin/gsd-tools.cjs` (extractFrontmatter, reconstructFrontmatter, spliceFrontmatter)
- Impact: Fragile to malformed YAML. No validation that parsed content matches expected schema. Edge cases in escaping (quotes, colons, hashes) handled inconsistently. Reconstructed YAML may not round-trip correctly. Changes to schema require updating parsing logic.
- Fix approach: Migrate to mature YAML library (js-yaml, yaml) with schema validation. Define schemas as JSON Schema / TypeScript interfaces. Validate after parsing, reject invalid documents early.

**Synchronous git operations blocking CLI:**
- Issue: Four execSync calls to git (lines ~213, ~229, ~1433, ~2812) without timeouts. If git hangs, CLI blocks indefinitely.
- Files: `get-shit-done/bin/gsd-tools.cjs` (isGitIgnored, execGit, cmdCommit)
- Impact: User experience degrades if git is slow (network paths, large repos). No feedback to user during long operations. Process can be killed but doesn't clean up state.
- Fix approach: Use child_process with explicit timeouts (30s default, configurable). Wrap git operations in async/await with timeout promises. Add spinner/progress feedback during waits.

**Manual path normalization for Windows compatibility:**
- Issue: One-off platform check in install.js (line ~1068: `result.indexOf('\\') > -1`) but inconsistent path handling throughout. gsd-tools uses path.join() correctly but some workflows/hooks may use hardcoded `/` paths.
- Files: `bin/install.js` (getOpencodeGlobalDir, expandTilde), `get-shit-done/bin/gsd-tools.cjs` (execGit path escaping)
- Impact: Cross-platform compatibility fragile. Windows users may encounter path resolution failures in edge cases. No systematic path normalization strategy.
- Fix approach: Add path utility module: `{ normalize, join, resolve, isAbsolute, sep }` wrapping Node's path module with consistent behavior. Use everywhere instead of mixing path.join() with string interpolation.

## Known Bugs

**ROADMAP progress table counts out of sync with reality:**
- Symptoms: ROADMAP.md shows "3/5 Complete" but only 2 SUMMARY files exist on disk. Happens when plan counts are edited manually or phases are removed.
- Files: `get-shit-done/bin/gsd-tools.cjs` (cmdPhaseComplete, cmdRoadmapUpdateProgress), `get-shit-done/workflows/complete-milestone.md`
- Trigger: Manually editing ROADMAP.md plan counts, then running `/gsd:complete-milestone` without re-syncing
- Workaround: Run `node get-shit-done/bin/gsd-tools.cjs roadmap analyze` to see real counts, then `roadmap update-plan-progress <phase>` to fix
- Fix: Auto-sync counts from disk (already partially implemented in ~line 915-940). Make it idempotent and call before any milestone operation.

**Frontmatter escaping edge case with colons in values:**
- Symptoms: Values containing colons (e.g., `created: 2026-02-15T10:30:45Z`) round-trip incorrectly through extractFrontmatter → reconstructFrontmatter
- Files: `get-shit-done/bin/gsd-tools.cjs` (reconstructFrontmatter lines ~381-387, extractFrontmatter lines ~281-301)
- Trigger: Create PLAN.md with ISO timestamp in custom field, run any frontmatter mutation, timestamp gets quoted inconsistently
- Workaround: Manual YAML escaping in PLAN.md frontmatter
- Fix: Use mature YAML library that handles quoting rules per YAML spec.

**Verifier doesn't fail on missing VERIFICATION.md:**
- Symptoms: `/gsd:verify-work` reports "Verification complete" even when no VERIFICATION.md exists, if phase directory exists
- Files: `get-shit-done/bin/gsd-tools.cjs` (cmdInitVerifyWork), `get-shit-done/workflows/verify-phase.md`
- Trigger: Run verify-work immediately after execute-phase without creating VERIFICATION.md
- Workaround: Manually create VERIFICATION.md or re-run verify-phase to scaffold it
- Fix: Make VERIFICATION.md required before verification can pass. Check for file existence at workflow start.

## Security Considerations

**Command injection in git operations:**
- Risk: execSync escaping (line ~225-227) uses regex `/^[a-zA-Z0-9._\-/=:@]+$/` to decide whether to quote arguments. Non-matching args get shell-escaped with `'` + replace(/'/g, "'\\''") pattern, but this is manual.
- Files: `get-shit-done/bin/gsd-tools.cjs` (execGit function)
- Current mitigation: Node.js child_process.execSync() in stdio:pipe mode doesn't inherit stdin, limiting stdin-based injection. Arguments passed as string, not array (worse), but regex escaping attempts to prevent word splitting.
- Recommendations:
  1. Migrate to array-based git invocation: `execSync('git', [arg1, arg2], {...})` (safer than string concat)
  2. Audit each git call individually (four locations: lines ~213, ~229, ~1433, ~2812)
  3. Add git command allowlist if possible (expect only specific verbs: status, add, commit, log, check-ignore)

**File I/O without input validation:**
- Risk: Path parameters in functions like cmdVerifyReferences (line ~2284) and others use path.isAbsolute() check but don't validate against traversal attacks (`..` in paths).
- Files: `get-shit-done/bin/gsd-tools.cjs` (cmdVerifyReferences, cmdScaffold, multiple file operations)
- Current mitigation: Running as user in user's home directory limits blast radius, but symlink attacks possible
- Recommendations:
  1. Resolve all paths with path.resolve() and check startsWith(basePath) before operating
  2. Use fs.readFileSync() with explicit error handling (currently some try/catch blocks empty lines ~546, ~548, ~1001, ~2526)
  3. Reject paths containing `..` or symlinks outside project

**Environment variable injection in templates:**
- Risk: scaffold commands (cmdScaffold, ~lines 3560-3620) embed phase/name into template strings without escaping. If names contain `${...}` expressions, could be interpolated.
- Files: `get-shit-done/bin/gsd-tools.cjs` (cmdScaffold function)
- Current mitigation: Phase numbers are validated (regex on line ~245), but name comes from user input unvalidated
- Recommendations:
  1. Use template literals as data, not as interpolation targets
  2. Validate phase name: alphanumeric + spaces/dashes only
  3. Escape special chars in reconstructed YAML values

## Performance Bottlenecks

**Frontmatter parsing on every file read:**
- Problem: extractFrontmatter is called for every PLAN.md, SUMMARY.md, STATE.md, but does full regex scan + stack parsing (lines ~253-326) even for 10KB+ files
- Files: `get-shit-done/bin/gsd-tools.cjs` (called from cmdPhasePlanIndex ~1825, cmdStateSnapshot ~1954, cmdSummaryExtract ~2025, plus init commands)
- Cause: No memoization, no lazy parsing, regex match entire file content. Stack-based parser processes every line.
- Improvement path:
  1. Cache parsed YAML for immutable files (SUMMARY.md never changes after creation)
  2. Use streaming parser for large files
  3. Profile actual performance (is it actually slow? 4825 lines suggests monolithic design causes perception of slowness even if individual operations fast)

**No batching for git status checks:**
- Problem: isGitIgnored function (line ~211) calls `git check-ignore` once per file, spawning subprocess each time. If checking 100 files, 100 git spawns.
- Files: `get-shit-done/bin/gsd-tools.cjs` (isGitIgnored, called from history-digest and other list operations)
- Cause: One-off design, no batch operation
- Improvement path: Replace with `git check-ignore --stdin` and batch all checks in one spawn. Requires refactoring history-digest logic to collect all paths first.

**History digest loads all SUMMARY.md files into memory:**
- Problem: cmdHistoryDigest (lines ~745-770) reads all phase SUMMARY.md files in loop, extracting frontmatter for each. No pagination or streaming.
- Files: `get-shit-done/bin/gsd-tools.cjs` (cmdHistoryDigest)
- Cause: Simple implementation for current use case (typical projects have <50 phases), but doesn't scale
- Improvement path: Implement pagination (--limit, --offset), or streaming JSON lines output

## Fragile Areas

**ROADMAP.md phase header detection:**
- Files: `get-shit-done/bin/gsd-tools.cjs` (cmdRoadmapGetPhase ~890-920)
- Why fragile: Regex looks for `/^#+\s*([0-9.]+).*$/` but CHANGELOG.md mentions recent fix for accepting both `##` and `###` (line 2:23 in CHANGELOG). Code now handles both (line ~891: `if (!/^#{2,3}\s*\d+/.test(line))`), but only 2-3 levels tested. What if user uses `#` or `####`?
- Safe modification:
  1. Add schema validation: max heading level = 3, min = 2
  2. Write test cases for edge cases (single #, four ####, etc.)
  3. Document expected format in template
- Test coverage: gsd-tools.test.cjs has 22 tests but none for malformed ROADMAP parsing

**Phase numbering logic:**
- Files: `get-shit-done/bin/gsd-tools.cjs` (normalizePhaseName ~244-251, cmdPhaseNextDecimal ~960-988, cmdPhaseAdd ~1005-1033, cmdPhaseInsert ~1035-1118)
- Why fragile: Decimal phase math (1.1 → 1.2 → 2) has multiple implementations. normalizePhaseName pads phase part (01 vs 1) but doesn't validate decimal part. What if someone manually edits ROADMAP to phase "1.15" — does next-decimal handle it correctly?
- Safe modification:
  1. Centralize phase arithmetic: single function handles padding, comparison, increment
  2. Write property tests: next-decimal(X) > X, next-decimal(1.5) = 2, next-decimal(2) = 2.1, etc.
  3. Validate on ROADMAP write: reject malformed phase numbers

**File matching with regex patterns:**
- Files: `get-shit-done/bin/gsd-tools.cjs` (cmdVerifyPhaseCompleteness ~2251-2256 uses `/-PLAN\.md$/i`, `/-SUMMARY\.md$/i`)
- Why fragile: Case-insensitive match but gsd always creates UPPER case. What if user renames file to `-plan.md` (lowercase)? Would break matching. What about Windows path separators in filenames?
- Safe modification:
  1. Enforce strict uppercase for all file operations: fail if file exists but differently-cased
  2. Add validation step that lists all markdown files and flags unexpected names
  3. Document file naming rules in templates

## Scaling Limits

**Phase directory structure limit (~50 phases):**
- Current capacity: Tested with typical ~20-50 phases. history-digest loads all summaries into memory, no pagination
- Limit: At 200+ phases, memory footprint grows, git operations slow down (git log filters entire history)
- Scaling path: Implement phase pagination in history-digest, lazy-load SUMMARY.md, batch git operations

**Config file size:**
- Current capacity: .planning/config.json expected to be <5KB. No limits enforced.
- Limit: If user adds 100s of custom decision entries or blocker lists, JSON parsing gets slow
- Scaling path: Add config archive feature (move old decisions to archive.json), implement streaming JSON parser if needed

**Workflow file size:**
- Current capacity: Individual workflow .md files are 1-30KB. gsd-tools doesn't read workflows, but install.js copies them
- Limit: Large projects with 200+ files in commands/agents/workflows directory will slow installer
- Scaling path: Lazy-load workflow files on-demand instead of copying all

## Dependencies at Risk

**No external dependencies in gsd-tools.cjs:**
- Risk: Using built-in fs, path, execSync, but custom YAML parser is fragile. One regex bug could break all state management.
- Impact: Cannot easily update YAML format without rewriting parser. No community auditing of format handling.
- Migration plan: Add js-yaml or yaml package. Implement gradual migration: new files use mature parser, old files still readable with custom parser.

**install.js has tight coupling to config file formats:**
- Risk: Hard-coded JSON.parse paths for .claude/config.json, .opencode/config.json, .gemini/config.json. If Claude/OpenCode changes format, installer breaks.
- Impact: New Claude Code releases may use different config location or schema
- Migration plan:
  1. Implement config version detection
  2. Add migration guide for version bumps
  3. Use defensive parsing: fallback to defaults if schema unrecognized

## Missing Critical Features

**No rollback mechanism for failed phase operations:**
- Problem: If `/gsd:execute-phase` fails mid-wave, files may be partially created. No way to undo or rollback to known good state.
- Blocks: Users cannot safely retry failing phases, must manually clean up
- Fix: Implement phase transaction: create temporary branch, rollback on failure, only merge on success

**No duplicate detection for state management:**
- Problem: If user runs `/gsd:new-phase` twice with same name, creates duplicate phase directory or ROADMAP entries. No idempotency check.
- Blocks: State corruption if operations retried
- Fix: Check existing phase before creating, fail gracefully if duplicate found

**No migration system for config schema changes:**
- Problem: CHANGELOG lists many breaking changes (e.g., ISSUES.md → phase-scoped UAT, line CHANGELOG:717). No automatic migration for existing projects.
- Blocks: Users must manually update old projects when upgrading GSD
- Fix: Create .planning/schema-version.json, implement migration functions for each version bump

## Test Coverage Gaps

**No tests for frontmatter round-tripping with edge cases:**
- What's not tested: Values with special characters (colons, quotes, hashes), deeply nested objects, large arrays
- Files: `get-shit-done/bin/gsd-tools.cjs` (extractFrontmatter, reconstructFrontmatter, spliceFrontmatter) but only 22 tests in gsd-tools.test.cjs
- Risk: Silent data corruption on round-trip. User edits frontmatter, field value changes unexpectedly.
- Priority: **High** - state management depends on this

**No tests for Windows path handling:**
- What's not tested: Path operations on Windows (backslash vs forward slash), symlinks, UNC paths
- Files: `bin/install.js`, `get-shit-done/bin/gsd-tools.cjs` (path operations throughout)
- Risk: Install fails on Windows, paths broken in generated config
- Priority: **High** - users report Windows issues but untested

**No tests for concurrent CLI invocations:**
- What's not tested: Two gsd-tools calls running simultaneously (race condition on .planning/config.json read/write)
- Files: `get-shit-done/bin/gsd-tools.cjs` (config loading, file writes)
- Risk: Config corruption if user runs `/gsd:something` while another command mid-execution
- Priority: **Medium** - unlikely in single-user interactive CLI, but possible with automation

**No tests for malformed input documents:**
- What's not tested: Invalid PLAN.md (missing fields, malformed frontmatter), corrupted SUMMARY.md, ROADMAP with duplicate phases
- Files: `get-shit-done/bin/gsd-tools.cjs` (verify commands, parsing functions)
- Risk: Tools crash or silently ignore corruption instead of reporting errors
- Priority: **Medium** - recovery harder if corruption not detected early

## Code Quality Issues

**Loose error handling with empty catch blocks:**
- Locations: Lines 546, 548, 1001, 2526 in gsd-tools.cjs have `catch {}` with no logging
- Impact: Silent failures — users don't know if operation succeeded or failed. Bugs become hard to diagnose.
- Fix: Replace `catch {}` with either: (1) `catch (e) { console.warn('Context:', e.message); }` or (2) make catch-able error intentional with comment explaining why

**Implicit undefined returns:**
- Pattern: Functions like cmdVerifyReferences (line ~2282) and others don't explicitly return on error paths. Callers get undefined, which is falsy but non-informative.
- Impact: Callers can't distinguish "success with empty result" from "failure"
- Fix: Return explicit objects: `{ success: false, error: '...' }` or throw Error, don't return undefined

**Type uncertainty in YAML parser output:**
- Issue: extractFrontmatter returns object with mixed types (strings, arrays, objects). Callers must type-check every field. E.g., `fm.depends_on` could be string or array.
- Files: `get-shit-done/bin/gsd-tools.cjs` (used in cmdStateAdvancePlan ~1106, cmdPhasePlanIndex ~1829, everywhere)
- Impact: Type errors on field access, defensive coding required everywhere, easy to miss edge case
- Fix: Validate and coerce types right after parsing. Define strict schema, reject non-conforming documents.

---

*Concerns audit: 2026-02-15*
