# Coding Conventions

**Analysis Date:** 2026-02-15

## Naming Patterns

**Files:**
- Kebab-case for executables and shell files: `gsd-check-update.js`, `gsd-statusline.js`, `gsd-tools.cjs`
- Kebab-case for test files: `gsd-tools.test.cjs`
- CamelCase for function/variable names within files

**Functions:**
- camelCase for all functions: `getDirName()`, `parseConfigDirArg()`, `expandTilde()`, `verifyInstalled()`
- Descriptive verb-noun pattern: `readSettings()`, `writeSettings()`, `cleanupOrphanedFiles()`, `cleanupOrphanedHooks()`
- Semantic naming for helper functions: `fileHash()`, `generateManifest()`, `saveLocalPatches()`

**Variables:**
- camelCase for all variable declarations: `explicitConfigDir`, `selectedRuntimes`, `hasGlobal`, `hasLocal`
- Boolean variables prefixed with `has`, `is`, or `can`: `hasGlobal`, `isGlobal`, `isOpencode`, `isGemini`, `forceStatusline`
- Constants in UPPER_CASE when appropriate: `PATCHES_DIR_NAME`, `MANIFEST_NAME`, `TOOLS_PATH`, `HOOKS_TO_COPY`
- Descriptive names for tracking variables: `removedCount`, `agentCount`, `hookCount`, `modified`

**Types:**
- No explicit TypeScript types (plain JavaScript)
- Objects documented via JSDoc comments where needed

## Code Style

**Formatting:**
- No enforced formatter (no .prettierrc, .eslintrc, or biome.json detected)
- Manual formatting conventions observed:
  - 2-space indentation (standard Node.js convention)
  - Line breaks between logical sections (extra newlines after function blocks)
  - String concatenation uses backticks for template literals: `` `${value}` ``

**Linting:**
- No linting rules enforced
- Code follows loose JavaScript best practices

## Import Organization

**Order:**
1. Built-in Node.js modules: `const fs = require('fs');`
2. Third-party modules: `const pkg = require('../package.json');`
3. Local modules/constants: Custom constants defined after imports

**Path Aliases:**
- No path aliases configured (using relative paths)
- Relative imports with `path.join(__dirname, ...)`

**Examples from codebase:**
```javascript
const fs = require('fs');
const path = require('path');
const os = require('os');
const readline = require('readline');
const crypto = require('crypto');

const pkg = require('../package.json');
```

## Error Handling

**Patterns:**
- Try-catch blocks for synchronous operations that may throw
- Graceful degradation with empty object/array fallbacks: `return {};` or `return [];`
- Error messaging with context clues included in catch blocks
- Process exit codes: `process.exit(0)` for success, `process.exit(1)` for failure
- Custom result objects with success flag: `{ success: true, output: result }`
- Errors logged before returning: `console.error(...)`

**Examples from codebase:**
```javascript
// From install.js - safe file reading
function readSettings(settingsPath) {
  if (fs.existsSync(settingsPath)) {
    try {
      return JSON.parse(fs.readFileSync(settingsPath, 'utf8'));
    } catch (e) {
      return {};
    }
  }
  return {};
}

// From gsd-tools.test.cjs - test result handling
try {
  const result = execSync(`node "${TOOLS_PATH}" ${args}`, { ... });
  return { success: true, output: result.trim() };
} catch (err) {
  return {
    success: false,
    output: err.stdout?.toString().trim() || '',
    error: err.stderr?.toString().trim() || err.message,
  };
}
```

## Logging

**Framework:** `console` (native Node.js)

**Patterns:**
- `console.log()` for standard output and progress messages
- `console.error()` for error messages
- Colored output using ANSI escape codes for terminal highlighting
- Color definitions stored in constants: `const cyan = '\x1b[36m'`, `const green = '\x1b[32m'`, `const yellow = '\x1b[33m'`, `const dim = '\x1b[2m'`, `const reset = '\x1b[0m'`
- Messages use template literals with color variables: `` console.log(`  ${green}✓${reset} Message`) ``
- Indentation with 2 or 4 spaces for output alignment

**Examples from codebase:**
```javascript
console.log(`  ${green}✓${reset} Installed ${count} commands to command/`);
console.log(`  ${yellow}⚠${reset} Skipping statusline (already configured)`);
console.error(`  ${yellow}Cannot specify both --global and --local${reset}`);
```

## Comments

**When to Comment:**
- Function documentation via JSDoc blocks (see below)
- Complex logic that isn't self-evident: mapping tables, transformation rules, backward compatibility workarounds
- Section dividers for major code sections (e.g., "Local Patch Persistence")
- Explanations of non-obvious behavior or design decisions

**JSDoc/TSDoc:**
- Used consistently for public functions and exported helpers
- Format: `/** [description] */` with optional `@param` and `@returns` tags
- Multi-line format for functions with parameters

**Examples from codebase:**
```javascript
/**
 * Get the config directory path relative to home directory for a runtime
 * Used for templating hooks that use path.join(homeDir, '<configDir>', ...)
 * @param {string} runtime - 'claude', 'opencode', or 'gemini'
 * @param {boolean} isGlobal - Whether this is a global install
 */
function getConfigDirFromHome(runtime, isGlobal) {
  // ...
}

/**
 * Expand ~ to home directory (shell doesn't expand in env vars passed to node)
 */
function expandTilde(filePath) {
  // ...
}

/**
 * Save any locally modified GSD files before they get wiped
 */
saveLocalPatches(targetDir);
```

**Comment Style:**
- Single-line comments: `// Comment` (not slashes on separate lines)
- Section headers: `// ──────────────────────────────────────────────────────` with title below
- Inline: `// Comment explaining the next line of code`

## Function Design

**Size:**
- Functions range from 3-50 lines
- Small helpers for single operations (5-10 lines)
- Complex operations split into helpers to improve readability

**Parameters:**
- Direct parameters for primary inputs
- Options objects with destructuring for optional parameters (though rarely used)
- Callbacks for async operations (see `promptRuntime()`, `handleStatusline()`)
- Default parameters: `function getGlobalDir(runtime, explicitDir = null)`

**Return Values:**
- Simple values for straightforward operations
- Objects for compound results: `{ success, output, error }`
- Objects with configuration: `{ settingsPath, settings, statuslineCommand, runtime }`
- null/undefined for "not found" cases
- Early returns to reduce nesting

**Examples from codebase:**
```javascript
// Simple return
function getDirName(runtime) {
  if (runtime === 'opencode') return '.opencode';
  if (runtime === 'gemini') return '.gemini';
  return '.claude';
}

// Early returns reduce nesting
if (!nextArg || nextArg.startsWith('-')) {
  console.error(`  ${yellow}--config-dir requires a path argument${reset}`);
  process.exit(1);
}
return nextArg;

// Compound return object
return {
  success: true,
  output: result.trim(),
};
```

## Module Design

**Exports:**
- Single exports via direct module.exports
- No explicit exports in these files (files are executed scripts)
- Functions declared at top level and called from main logic blocks

**Structure Pattern:**
- Constants and configuration at top
- Helper function definitions in logical groups
- Main logic at bottom

**Examples from codebase:**
```javascript
// Colors defined at top
const cyan = '\x1b[36m';
const green = '\x1b[32m';

// Constants after imports
const MODEL_PROFILES = { ... };

// Helpers grouped logically
function parseIncludeFlag(args) { ... }
function safeReadFile(filePath) { ... }

// Main logic at bottom
if (hasGlobal && hasLocal) {
  console.error(...);
  process.exit(1);
}
```

**Grouping:**
- Related helpers grouped together with comments
- Helper definitions before usage
- Main script flow clearly marked at end

---

*Convention analysis: 2026-02-15*
