# External Integrations

**Analysis Date:** 2026-02-15

## APIs & External Services

**Web Search:**
- Brave Search API - Optional web search capability
  - SDK/Client: Native Node.js `fetch()` (no SDK package required)
  - Endpoint: `https://api.search.brave.com/res/v1/web/search`
  - Auth: `BRAVE_API_KEY` environment variable or `~/.gsd/brave_api_key` file
  - Implementation: `get-shit-done/bin/gsd-tools.cjs` - `cmdWebsearch()` function
  - Status: Optional feature, gracefully disabled if key not configured

**Package Registry:**
- npm registry - For version checking and updates
  - Command: `npm view get-shit-done-cc version` (in `hooks/gsd-check-update.js`)
  - Used for: Detecting available GSD updates
  - Timeout: 10 seconds

## AI Runtimes (Integration Targets)

**Claude Code:**
- Config location: `~/.claude/` (global) or `./.claude/` (local)
- Installation: Commands copied to `~/.claude/commands/gsd/`
- Agents: Copied to `~/.claude/agents/gsd-*.md`
- Hooks: Installed to `~/.claude/hooks/`
- Settings: Updates to `~/.claude/settings.json`
- Tools: Marked with `allowed-tools:` frontmatter (comma-separated format)

**OpenCode:**
- Config location: `~/.config/opencode/` (XDG Base Directory compliant)
- Installation: Commands flattened to `~/.config/opencode/command/gsd-*.md`
- Environment vars: `OPENCODE_CONFIG_DIR`, `OPENCODE_CONFIG`, `XDG_CONFIG_HOME`
- Permissions: Configured in `~/.config/opencode/opencode.json`
- Tools: Converted to object format `{tool: true}`
- Conversion: Tool names mapped (e.g., `AskUserQuestion` → `question`, `SlashCommand` → `skill`)

**Gemini CLI:**
- Config location: `~/.gemini/` (global) or `./.gemini/` (local)
- Installation: Commands converted to `.toml` format
- Agents: Converted to `.md` files with YAML array format for tools
- Experimental: Enables experimental agents in `settings.json`
- Tools: Converted to Gemini built-in names (e.g., `Read` → `read_file`, `Bash` → `run_shell_command`)

## Data Storage

**Filesystem Only:**
- Project state: `.planning/` directory hierarchy
- User configuration: Home directory config folders (`~/.claude/`, `~/.config/opencode/`, `~/.gemini/`)
- No database, no cloud sync

**File Types Stored:**
- `PROJECT.md` - Project definition
- `REQUIREMENTS.md` - Requirements extracted
- `ROADMAP.md` - Phase roadmap
- `STATE.md` - Current workflow state
- `SUMMARY.md` - Phase summaries (per phase)
- `PLAN*.md` - Implementation plans
- Phase directories: `.planning/phases/N/`

**Manifest & Patch System:**
- `gsd-file-manifest.json` - SHA256 hashes of installed files (enables safe updates)
- `gsd-local-patches/` - User modifications backed up before updates
- `backup-meta.json` - Metadata about backed-up patches

## Configuration & Defaults

**System Configuration:**
- `.planning/config.json` - Project-specific settings
  - Model profile selection (quality/balanced/budget)
  - Workflow flags (research, plan_check, verifier, auto_advance)
  - Parallelization settings
  - Git integration settings
  - Brave Search enablement

**User Defaults (Optional):**
- `~/.gsd/defaults.json` - User-level default settings for new projects
- Applied when creating new projects
- Overridable per-project

**Templates:**
- `get-shit-done/templates/config.json` - Default config template with workflow gates and safety settings

## Version Management

**GSD Version:**
- Stored in: `get-shit-done/VERSION` (after installation)
- Read by: Update checker hook and installer
- Current package version: v1.19.1 (from `package.json`)

**Update Checking:**
- Hook: `hooks/gsd-check-update.js` - Runs on session start
- Cache: `~/.claude/cache/gsd-update-check.json`
- Comparison: Installed version vs. latest on npm registry
- No automatic installation (user prompted in UI)

## Git Integration

**Commit Operations:**
- `state commit <message> [--files f1 f2]` - Git commits planning docs
- Supports custom attribution via `~/.claude/settings.json` or `~/.gemini/settings.json`
- Attribution pattern: `Co-Authored-By:` line (customizable per runtime)

**Branching (Optional):**
- Strategies: `none`, `phase`, `milestone` (configured in `.planning/config.json`)
- Templates:
  - Phase: `gsd/phase-{phase}-{slug}` (customizable)
  - Milestone: `gsd/{milestone}-{slug}` (customizable)

## Webhooks & Callbacks

**None configured by default.**

GSD is a local CLI tool with no outbound webhooks. Integration with external services:
- **Inbound:** Only manual CLI commands from Claude Code/OpenCode/Gemini
- **Outbound:** Only npm registry (version check) and optional Brave Search API

## Environment Variables

**For Web Search:**
- `BRAVE_API_KEY` - Brave Search API key (alternative: `~/.gsd/brave_api_key` file)

**For Runtime Configuration:**
- `CLAUDE_CONFIG_DIR` - Override Claude Code config directory (default: `~/.claude`)
- `OPENCODE_CONFIG_DIR` - Override OpenCode config directory (default: `~/.config/opencode`)
- `XDG_CONFIG_HOME` - XDG Base Directory for OpenCode (if `OPENCODE_CONFIG_DIR` not set)
- `OPENCODE_CONFIG` - OpenCode config file path (uses its directory)
- `GEMINI_CONFIG_DIR` - Override Gemini config directory (default: `~/.gemini`)
- `HOME` - User home directory (standard)

**For Installation:**
- `--config-dir <path>` / `-c <path>` - Override config directory during install

## Security & Secrets

**Sensitive Data Handling:**
- No credentials embedded in code
- API keys stored in user home directory, never in project `.planning/`
- `.gitignore` prevents accidental commits of `.claude/`, `.config/opencode/`, `.gemini/`
- Manifest system enables detection of modified files (security audit via hash comparison)

**Safe Update Pattern:**
- Before update, user modifications are backed up to `gsd-local-patches/`
- Allows safe re-application after update
- User prompted to merge modifications after upgrade

---

*Integration audit: 2026-02-15*
