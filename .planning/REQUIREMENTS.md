# Requirements: GSD Workflow Customization

**Defined:** 2026-02-15
**Core Value:** Every command is a skill that produces correct, verified code fast — by looking up current docs instead of guessing, testing in the browser instead of hoping, and minimizing token waste.

## v1 Requirements

### Skills Conversion

- [ ] **SKILL-01**: plan-phase command is a proper SKILL.md with frontmatter, allowed-tools, and progressive disclosure (body + references/)
- [ ] **SKILL-02**: execute-phase command is a proper SKILL.md with the same structure
- [ ] **SKILL-03**: New verify skill performs Chrome DevTools runtime testing (snapshot → interact → verify pattern)
- [ ] **SKILL-04**: New review skill checks code quality, loads best-practice skills, and commits on PASS
- [ ] **SKILL-05**: Agent definition files live in skill references/ directories, not as standalone agent .md files

### MCP Integration

- [ ] **MCP-01**: Plan skill mandates Context7 lookups (resolve → query) for every library the implementation touches
- [ ] **MCP-02**: Plan skill uses microsoft-docs MCP for Azure-related features (search → code samples → fetch)
- [ ] **MCP-03**: Plan skill uses shadcn MCP for UI component planning (getComponents → getComponent)
- [ ] **MCP-04**: Execute skill verifies plan's API references via Context7 before implementing
- [ ] **MCP-05**: Execute skill loads relevant best-practice skills (react, nestjs, owasp, frontend-design) before writing code
- [ ] **MCP-06**: Verify skill uses Chrome DevTools MCP for automated workflow tests and runtime inspection
- [ ] **MCP-07**: Review skill uses IDE diagnostics MCP for TypeScript error checking
- [ ] **MCP-08**: All MCP-using skills handle unavailability gracefully (fallback to WebSearch, confidence tagging)

### Token Optimization

- [ ] **TOKEN-01**: Agents receive file paths, not injected content (agents read files with their fresh 200k context)
- [ ] **TOKEN-02**: SKILL.md bodies are lean orchestration only (~500-1,500 tokens), detailed instructions in references/
- [ ] **TOKEN-03**: Research findings written to RESEARCH.md once; downstream agents read the file instead of re-querying MCPs
- [ ] **TOKEN-04**: gsd-tools.js supports phase-specific extraction (e.g., requirements for one phase, not full file)
- [ ] **TOKEN-05**: Agent prompts split into core + conditional sections (TDD, checkpoints, auth gates loaded only when needed)

### Pipeline Flow

- [ ] **PIPE-01**: Phase execution follows plan → execute → verify → review pipeline
- [ ] **PIPE-02**: Planner never implements code (read-only + write plan)
- [ ] **PIPE-03**: Executor never commits (reviewer commits after PASS)
- [ ] **PIPE-04**: Verifier never modifies source files (reports findings only)
- [ ] **PIPE-05**: Reviewer commits only after review result is PASS
- [ ] **PIPE-06**: Explicit allowed-tools in frontmatter enforces role separation

## v2 Requirements

### Supporting Commands

- **SUPP-01**: new-project command converted to skill
- **SUPP-02**: progress command converted to skill
- **SUPP-03**: settings/help commands converted to skills
- **SUPP-04**: Phase management (add/insert/remove) converted to skills
- **SUPP-05**: Session management (pause/resume) converted to skills
- **SUPP-06**: map-codebase and debug commands converted to skills

### Advanced Features

- **ADV-01**: Figma MCP integration in planning (design-to-code)
- **ADV-02**: Confidence tag system framework-wide (HIGH/MEDIUM/LOW propagation)
- **ADV-03**: Test delegation optimization (Sonnet planner + Haiku browser-tester)
- **ADV-04**: Workflow generation from verify skill (5 explorer agents + composer)

## Out of Scope

| Feature | Reason |
|---------|--------|
| GUI / web interface | CLI-only tool |
| Non-JS/TS codebase support | Current tooling is JS/TS focused |
| Removing gsd-tools.js | Still needed for init, roadmap parsing, state management |
| Removing .planning/ structure | Proven, works with existing CLI |
| Supporting other AI platforms | Claude Code specific for now |
| Backward compatibility with old GSD commands | Clean break, not incremental migration |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| SKILL-01 | Phase 1 | Pending |
| SKILL-02 | Phase 1 | Pending |
| SKILL-03 | Phase 3 | Pending |
| SKILL-04 | Phase 3 | Pending |
| SKILL-05 | Phase 1 | Pending |
| MCP-01 | Phase 2 | Pending |
| MCP-02 | Phase 2 | Pending |
| MCP-03 | Phase 2 | Pending |
| MCP-04 | Phase 2 | Pending |
| MCP-05 | Phase 2 | Pending |
| MCP-06 | Phase 3 | Pending |
| MCP-07 | Phase 3 | Pending |
| MCP-08 | Phase 4 | Pending |
| TOKEN-01 | Phase 1 | Pending |
| TOKEN-02 | Phase 1 | Pending |
| TOKEN-03 | Phase 4 | Pending |
| TOKEN-04 | Phase 4 | Pending |
| TOKEN-05 | Phase 4 | Pending |
| PIPE-01 | Phase 3 | Pending |
| PIPE-02 | Phase 1 | Pending |
| PIPE-03 | Phase 1 | Pending |
| PIPE-04 | Phase 3 | Pending |
| PIPE-05 | Phase 3 | Pending |
| PIPE-06 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 19 total
- Mapped to phases: 19 (roadmap created)
- Unmapped: 0

---
*Requirements defined: 2026-02-15*
