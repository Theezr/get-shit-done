# GSD Workflow Customization

## What This Is

A complete customization of the GSD (Get Shit Done) workflow framework, transforming it from a generic project execution tool into a personalized development pipeline. The framework keeps its project structure (init, roadmap, phases) but replaces its internals with a skills-based architecture that uses MCP servers for real-time context, Chrome DevTools for runtime verification, and code review with best-practice skills.

## Core Value

Every command is a skill that produces correct, verified code fast — by looking up current docs (Context7, microsoft-docs) instead of guessing, testing in the browser (Chrome DevTools) instead of hoping, and minimizing token waste through progressive disclosure and path-based context passing.

## Requirements

### Validated

- GSD project structure (init → roadmap → phases) — existing, proven
- `.planning/` directory structure with phases, state tracking — existing
- Agent architecture for parallel execution — existing
- gsd-tools.js CLI for initialization, roadmap parsing, state management — existing
- Skill infrastructure in Claude Code — existing (nick:plan, nick:execute, nick:verify, nick:review)

### Active

- [ ] Convert all GSD commands to proper skills (SKILL.md format with frontmatter, progressive disclosure)
- [ ] Integrate Context7 MCP into planning and execution agents (mandatory doc lookups before referencing APIs)
- [ ] Integrate microsoft-docs MCP for Azure-related features
- [ ] Integrate Chrome DevTools MCP for runtime verification after execution
- [ ] Add code review step using best-practice skills (react, nestjs, owasp, frontend-design)
- [ ] Reduce token consumption through progressive disclosure, path-based context, slimmer workflows
- [ ] Simplify agent chain (11 types → fewer, smarter agents that use MCPs)
- [ ] Speed up execution phase (inline doc lookups, eliminate redundant agent spawns)
- [ ] Implement verify + review pipeline per phase (Chrome DevTools → code quality → commit on PASS)

### Out of Scope

- Changing the project init → roadmap → phases structure — proven, keep it
- Removing agent architecture entirely — agents stay, but get smarter
- Supporting non-JavaScript/TypeScript codebases — current tooling is JS/TS focused
- Building a GUI or web interface — CLI-only

## Context

**Current System:** GSD uses 30+ workflow .md files and 11 agent types. Commands are registered as slash commands that load workflow files. Agents generate from training data instead of looking up current docs.

**User's Pipeline:** Already has working skills (nick:plan, nick:execute, nick:verify, nick:review) that demonstrate the target pattern — mandatory Context7 lookups, Chrome DevTools verification, best-practice skill loading, strict role separation, `.pipeline/` artifacts.

**Skills Guide:** Anthropic's official guide defines the SKILL.md format — YAML frontmatter (name, description, allowed-tools), progressive disclosure (frontmatter → body → references/), composability across skills. Key insight: frontmatter is always loaded in system prompt, body loads on relevance, references load on demand.

**Token Efficiency Goals:**
- Pass file paths to agents, not content (agents have fresh 200k context)
- Use progressive disclosure (slim SKILL.md, detailed references/)
- Eliminate redundant research agents — use MCPs directly
- Reduce workflow file verbosity

**Existing Skills:** owasp-security, nestjs-best-practices, vercel-react-best-practices, web-design-guidelines, frontend-design, security-review — all already installed at ~/.claude/skills/

**Available MCPs:** Context7 (library docs), microsoft-docs (Azure docs), Chrome DevTools (browser testing/inspection), shadcn (UI components), Figma (design), IDE diagnostics

## Constraints

- **Backward Compatibility**: gsd-tools.js CLI must continue to work — it handles init, roadmap parsing, state management, commits
- **Skills Format**: Must follow Anthropic's SKILL.md specification (kebab-case name, description with triggers, allowed-tools)
- **Token Budget**: Each skill/workflow should be as lean as possible — move detail to references/
- **Agent Context**: Agents get fresh 200k context — pass paths, not content
- **MCP Availability**: Skills must gracefully handle MCPs being unavailable (fallback to training data)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Keep .planning/phases/ structure | Proven, works with gsd-tools.js | — Pending |
| All commands become skills | Skills have frontmatter, progressive disclosure, allowed-tools, composability | — Pending |
| Keep agent architecture | Parallel execution needs agents; but agents use MCPs now | — Pending |
| Both verify + review steps | Verify = runtime (Chrome), Review = code quality + commit | — Pending |
| Mandatory Context7 lookups | Never code from training data — APIs change | — Pending |
| Progressive disclosure for token savings | Frontmatter always loaded, body on relevance, references on demand | — Pending |

---
*Last updated: 2026-02-15 after initialization*
