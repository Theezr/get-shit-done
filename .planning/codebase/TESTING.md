# Testing Patterns

**Analysis Date:** 2026-02-15

## Test Framework

**Runner:**
- Node.js built-in `node:test` module (no external test framework)
- Version: Node 16.7.0+ (from package.json engines field)
- Config: Uses native Node.js test runner (no configuration file needed)

**Assertion Library:**
- Node.js built-in `node:assert` module (strict assertions)

**Run Commands:**
```bash
npm test                           # Run all tests (defined in package.json)
node --test get-shit-done/bin/gsd-tools.test.js  # Run specific test file
```

## Test File Organization

**Location:**
- Co-located with source: `get-shit-done/bin/gsd-tools.test.cjs` next to `get-shit-done/bin/gsd-tools.cjs`
- Test file uses `.test.cjs` suffix to distinguish from source

**Naming:**
- Pattern: `[source-name].test.cjs`
- Test command: `npm test` → `node --test get-shit-done/bin/gsd-tools.test.js` (per package.json)

**Structure:**
```
get-shit-done/bin/
├── gsd-tools.cjs          # Source
└── gsd-tools.test.cjs      # Tests (2273 lines)
```

## Test Structure

**Suite Organization:**
```javascript
const { test, describe, beforeEach, afterEach } = require('node:test');
const assert = require('node:assert');

describe('feature name', () => {
  let tmpDir;  // Shared state

  beforeEach(() => {
    tmpDir = createTempProject();
  });

  afterEach(() => {
    cleanup(tmpDir);
  });

  test('test case description', () => {
    // Test body
  });

  test('another test case', () => {
    // Test body
  });
});
```

**Patterns:**
- `describe()` groups related tests into logical suites
- `test()` defines individual test cases
- `beforeEach()` runs setup before each test (create temp directories)
- `afterEach()` runs cleanup after each test (remove temp files)
- No nested `describe()` blocks (flat organization)

**Examples from codebase:**
```javascript
describe('history-digest command', () => {
  let tmpDir;

  beforeEach(() => {
    tmpDir = createTempProject();
  });

  afterEach(() => {
    cleanup(tmpDir);
  });

  test('empty phases directory returns valid schema', () => {
    const result = runGsdTools('history-digest', tmpDir);
    assert.ok(result.success, `Command failed: ${result.error}`);
    const digest = JSON.parse(result.output);
    assert.deepStrictEqual(digest.phases, {}, 'phases should be empty object');
  });
});
```

## Mocking

**Framework:** Manual mocking via file system (no mocking library)

**Patterns:**
```javascript
// Mock file system state by creating temp directory structure
function createTempProject() {
  const tmpDir = fs.mkdtempSync(path.join(require('os').tmpdir(), 'gsd-test-'));
  fs.mkdirSync(path.join(tmpDir, '.planning', 'phases'), { recursive: true });
  return tmpDir;
}

// Create mock data files
const phaseDir = path.join(tmpDir, '.planning', 'phases', '01-foundation');
fs.mkdirSync(phaseDir, { recursive: true });
fs.writeFileSync(
  path.join(phaseDir, '01-01-SUMMARY.md'),
  `---\nphase: "01"\nprovides:\n  - "Database"\n---`
);

// Execute command against mock filesystem
const result = runGsdTools('history-digest', tmpDir);
```

**Helper Function Pattern:**
```javascript
// Helper to run gsd-tools command in isolation
function runGsdTools(args, cwd = process.cwd()) {
  try {
    const result = execSync(`node "${TOOLS_PATH}" ${args}`, {
      cwd,
      encoding: 'utf-8',
      stdio: ['pipe', 'pipe', 'pipe'],  // Capture stdout/stderr
    });
    return { success: true, output: result.trim() };
  } catch (err) {
    return {
      success: false,
      output: err.stdout?.toString().trim() || '',
      error: err.stderr?.toString().trim() || err.message,
    };
  }
}
```

**What to Mock:**
- File system: Create temp directories with `fs.mkdtempSync()` and mock data via `fs.writeFileSync()`
- CLI execution: Spawn command in isolated temp directory via `execSync()`
- Dependencies: Create fixture files matching expected formats

**What NOT to Mock:**
- Node.js built-in modules (use real fs, path, etc.)
- External commands that are being tested (execute them directly)
- Date/time unless absolutely necessary
- Process exit codes (test actual exit behavior)

## Fixtures and Factories

**Test Data:**
```javascript
// Fixture: SUMMARY.md with frontmatter
const summaryContent = `---
phase: "01"
name: "Foundation Setup"
dependency-graph:
  provides:
    - "Database schema"
    - "Auth system"
  affects:
    - "API layer"
tech-stack:
  added:
    - "prisma"
    - "jose"
patterns-established:
  - "Repository pattern"
  - "JWT auth flow"
key-decisions:
  - "Use Prisma over Drizzle"
  - "JWT in httpOnly cookies"
---

# Summary content here
`;

fs.writeFileSync(path.join(phaseDir, '01-01-SUMMARY.md'), summaryContent);
```

**Location:**
- Inline fixtures in test file (no separate fixture directory)
- String templates for file content
- Created dynamically in tests, not stored as separate files

**Factory Functions:**
```javascript
// Factory to create test directory structure
function createTempProject() {
  const tmpDir = fs.mkdtempSync(path.join(require('os').tmpdir(), 'gsd-test-'));
  fs.mkdirSync(path.join(tmpDir, '.planning', 'phases'), { recursive: true });
  return tmpDir;
}

// Cleanup factory
function cleanup(tmpDir) {
  fs.rmSync(tmpDir, { recursive: true, force: true });
}
```

## Coverage

**Requirements:** Not detected / Not enforced

**Test Count:**
- Single test file: `gsd-tools.test.cjs` (2273 lines)
- 60+ test cases across 11 describe blocks covering:
  - `history-digest` command (10+ tests)
  - `phases list` command (7+ tests)
  - `roadmap get-phase` command (6+ tests)
  - `phase next-decimal` command (6+ tests)
  - `phase-plan-index` command (7+ tests)
  - `state-snapshot` command (10+ tests)
  - Additional edge cases and compatibility tests

**View Coverage:**
```bash
# Not configured (no coverage tool detected)
# Manual coverage tracking via test case counts
```

## Test Types

**Unit Tests:**
- Scope: Individual CLI commands and helper functions
- Approach: Execute command, parse output, assert on parsed result
- Strategy: Each command tested with multiple scenarios (success, error, edge cases)
- Example: `test('empty phases directory returns valid schema', ...)`

**Integration Tests:**
- Scope: Full command flow with file system interaction
- Approach: Create realistic directory structures, run command, verify file system changes
- Strategy: Test commands that read/write multiple files in sequence
- Example: Testing SUMMARY.md parsing with nested YAML structure

**E2E Tests:**
- Framework: Not used
- Manual testing likely required for full workflow validation

## Common Patterns

**Async Testing:**
```javascript
// Node.js test runner supports async functions directly
test('async operation', async () => {
  // Can use await directly
});

// Synchronous tests are more common in this codebase
test('sync operation', () => {
  // Direct test code
});
```

**Error Testing:**
```javascript
test('handles malformed input gracefully', () => {
  // Create malformed fixture
  fs.writeFileSync(
    path.join(phaseDir, '01-02-SUMMARY.md'),
    `---
broken: [unclosed
---`
  );

  // Execute and expect success (graceful degradation)
  const result = runGsdTools('history-digest', tmpDir);
  assert.ok(result.success, `Command should succeed despite malformed files: ${result.error}`);

  // Verify partial results
  const digest = JSON.parse(result.output);
  assert.ok(digest.phases['01'], 'Phase 01 should exist');
});
```

**Assertion Patterns:**
```javascript
// Basic existence check
assert.ok(result.success, `Command failed: ${result.error}`);

// Deep equality (for objects/arrays)
assert.deepStrictEqual(digest.phases, {}, 'phases should be empty object');
assert.deepStrictEqual(
  digest.phases['01'].provides.sort(),
  ['Auth system', 'Database schema'],
  'provides should contain nested values'
);

// String equality
assert.strictEqual(digest.decisions.length, 2, 'Should have 2 decisions');

// Array membership
assert.ok(
  digest.decisions.some(d => d.decision === 'Use Prisma over Drizzle'),
  'Should contain first decision'
);
```

## Test Data Patterns

**Common Fixtures:**
1. Empty directory structures (test graceful handling of missing files)
2. Well-formed YAML frontmatter (test parsing)
3. Malformed YAML (test error handling)
4. Nested/deeply-structured YAML (test complex data extraction)
5. Multiple files in sequence (test aggregation)
6. Backward compatibility formats (test legacy support)

**Example: Multiple phases test**
```javascript
test('multiple phases merged into single digest', () => {
  // Create phase 01
  const phase01Dir = path.join(tmpDir, '.planning', 'phases', '01-foundation');
  fs.mkdirSync(phase01Dir, { recursive: true });
  fs.writeFileSync(path.join(phase01Dir, '01-01-SUMMARY.md'), `---\n...`);

  // Create phase 02
  const phase02Dir = path.join(tmpDir, '.planning', 'phases', '02-api');
  fs.mkdirSync(phase02Dir, { recursive: true });
  fs.writeFileSync(path.join(phase02Dir, '02-01-SUMMARY.md'), `---\n...`);

  // Test aggregation
  const result = runGsdTools('history-digest', tmpDir);
  const digest = JSON.parse(result.output);
  assert.ok(digest.phases['01'], 'Phase 01 should exist');
  assert.ok(digest.phases['02'], 'Phase 02 should exist');
});
```

---

*Testing analysis: 2026-02-15*
