# Research Summary: GSD Workflow Customization

## Key Findings Across All Research

### 1. Token Savings: 60-75% reduction achievable
- **Biggest win:** Switch from content injection to path-based passing. GSD currently embeds full file contents (ROADMAP, REQUIREMENTS, STATE, RESEARCH) into every agent prompt — 12,000-20,000 tokens per agent, duplicated 4x across agents in a planning cycle.
- **Second win:** Progressive disclosure via skills format. Lean SKILL.md body (~1,500 tokens), detailed instructions in references/ (loaded only by agents that need them).
- **Third win:** Eliminate redundant agents. Research findings go in RESEARCH.md; downstream agents read the file instead of re-researching.

### 2. Skills Architecture: 1:1 mapping, lean bodies, heavy references/
- Every `/gsd:` command becomes one skill folder (SKILL.md + references/)
- SKILL.md body: orchestration logic only (~500-1,500 tokens)
- references/: agent prompts, templates, detailed instructions (loaded on demand)
- Frontmatter: precise triggers, explicit allowed-tools (no wildcards)
- 30 skill frontmatters cost ~6,000-9,000 tokens in system prompt (acceptable)

### 3. MCP Integration: Replace training-data guessing with real-time lookups
- **Context7:** Mandatory 2-step (resolve → query) for every library. Cost: 700-2,500 tokens per lookup. Far cheaper than the 15,000-45,000 tokens GSD wastes on content-passing.
- **Chrome DevTools:** 25+ tools. Pipeline: snapshot → interact → verify. Delegate to test-planner (Sonnet) → browser-tester (Haiku) for cost efficiency.
- **microsoft-docs:** 3-step (search → code samples → fetch). Only when Azure involved.
- **Graceful degradation:** Pre-flight check, fallback to WebSearch, confidence tags (HIGH/MEDIUM/LOW).

### 4. Gap Analysis: Keep structure, simplify internals
- **Corrected from raw analysis:** User wants to KEEP project structure (init → roadmap → phases), progress tracking, map-codebase, and debug. The gap analysis over-aggressively removed these.
- **Actually remove:** 11 agent types → 3-4 (planner, executor, verifier + optional reviewer). Eliminate: research-synthesizer, plan-checker, project-researcher, codebase-mapper, integration-checker, debugger (as separate agents).
- **Actually simplify:** Phase planning goes from 3 agents (researcher → planner → checker) to 1 smart agent that uses MCPs inline. Execution keeps wave-based parallelism but agents use Context7 instead of guessing.
- **Actually add:** Chrome DevTools verification, code review with best-practice skills, Figma MCP for design-to-code.

## Corrected Workflow Classification

| Category | Workflows | Action |
|----------|-----------|--------|
| **Core (keep + convert to skills)** | new-project, plan-phase, execute-phase, progress, map-codebase, debug, settings, help | Convert to SKILL.md format, add MCP integration |
| **Phase mgmt (keep, simplify)** | add-phase, insert-phase, remove-phase, discuss-phase | Convert to skills, slim down |
| **Milestone (keep, simplify)** | new-milestone, complete-milestone | Convert to skills |
| **Session (keep, simplify)** | pause-work, resume-project | Convert to skills |
| **New (add)** | verify (Chrome DevTools), review (code quality) | New skills from user's existing nick: skills |
| **Remove** | transition, update, discovery-phase, list-phase-assumptions, audit-milestone, plan-milestone-gaps, diagnose-issues, verify-phase (replaced by verify), verify-work (replaced by verify), quick (plan→execute IS the quick path) | No longer needed |

## Smart Addons for Token Efficiency / Speed

1. **Scoped Context7 queries** — "useEffect cleanup for async" not "React docs" — reduces returned tokens 5-10x
2. **Research caching** — Researcher writes RESEARCH.md; planner/executor READ it instead of re-querying MCPs
3. **Conditional references/** — Agent prompts split into core + optional sections (TDD, checkpoints, auth gates). Load only what the plan needs.
4. **gsd-tools.js enhancements** — Add phase-specific extraction: `gsd-tools extract-requirements --phase 3` returns only the requirements for that phase, not the full file
5. **Confidence tags** — HIGH (Context7 verified) / MEDIUM (WebFetch official docs) / LOW (training data). Downstream agents add validation steps for LOW confidence references.
6. **Test delegation cost optimization** — test-planner on Sonnet (reasoning matters), browser-tester on Haiku (rote execution). 3-5x cheaper testing.

## Recommended Phase Structure

Given "quick" depth (3-5 phases):

1. **Foundation** — Convert core skills (plan-phase, execute-phase) + add MCP integration + token optimization
2. **Verification Pipeline** — Add verify + review skills, Chrome DevTools integration
3. **Supporting Skills** — Convert remaining commands (progress, settings, help, phase mgmt, session mgmt)
4. **Polish** — gsd-tools.js enhancements, test the full pipeline end-to-end

---
*Research completed: 2026-02-15*
*Sources: SKILLS-ARCHITECTURE.md, TOKEN-OPTIMIZATION.md, MCP-INTEGRATION.md, GAP-ANALYSIS.md*
