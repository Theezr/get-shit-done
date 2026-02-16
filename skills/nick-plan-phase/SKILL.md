---
name: nick-plan-phase
description: >
  Create detailed execution plans for a roadmap phase. Orchestrates research,
  planning, and verification agents. Use when user says "plan phase",
  "create plan for phase", "/gsd:plan-phase", or wants to plan the next
  phase of their project. Requires .planning/ directory with ROADMAP.md.
argument-hint: "<phase-number> [--research] [--skip-research] [--gaps] [--skip-verify]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - WebFetch
  - mcp__context7__resolve-library-id
  - mcp__context7__query-docs
  - mcp__microsoft-docs__microsoft_docs_search
  - mcp__microsoft-docs__microsoft_code_sample_search
  - mcp__microsoft-docs__microsoft_docs_fetch
---

# Plan Phase Orchestrator

Orchestrates research, planning, and verification agents to create executable PLAN.md files for a roadmap phase. Agents read files themselves via paths -- no content injection.

## 1. Initialize

```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs init plan-phase "$PHASE")
```

Parse from JSON: `phase_dir`, `padded_phase`, `phase_number`, `phase_name`, `phase_slug`, `has_research`, `has_context`, `has_plans`, `plan_count`, `planning_exists`, `roadmap_exists`, `research_enabled`, `plan_checker_enabled`, `researcher_model`, `planner_model`, `checker_model`, `commit_docs`.

If `planning_exists` is false: Error -- run `/gsd:new-project` first.

## 2. Parse Arguments

Extract from $ARGUMENTS: phase number (integer or decimal like `2.1`), flags (`--research`, `--skip-research`, `--gaps`, `--skip-verify`).

If no phase number: detect next unplanned phase from roadmap.

If `phase_found` is false: validate phase exists in ROADMAP.md, create directory:
```bash
mkdir -p ".planning/phases/${padded_phase}-${phase_slug}"
```

Validate phase:
```bash
PHASE_INFO=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs roadmap get-phase "${PHASE}")
```

Extract `phase_number`, `phase_name`, `goal` from JSON.

## 3. Handle Research (conditional)

**Skip if:** `--gaps` flag, `--skip-research` flag, `research_enabled=false` (without `--research` override), or any RESEARCH*.md files exist (without `--research` flag).

**Determine strategy:** Read `research_strategy` from init JSON.

**If strategy is `single` (default):** Spawn one researcher as before:

```
Task(
  prompt="First, read ~/.claude/skills/nick-plan-phase/references/agents/nick-phase-researcher.md for your role and instructions.

  <objective>Research Phase {phase_number}: {phase_name}
  Answer: 'What do I need to know to PLAN this phase well?' Use mandatory Context7 lookups for every library, microsoft-docs for Azure services, and shadcn fallback for UI components.</objective>

  <files_to_read>
  - .planning/STATE.md
  - .planning/ROADMAP.md (Phase {N} section)
  - .planning/REQUIREMENTS.md (Phase {N} requirements)
  - {phase_dir}/*-CONTEXT.md (if exists)
  </files_to_read>

  <output>Write to: {phase_dir}/{padded_phase}-RESEARCH.md</output>",
  subagent_type="general-purpose",
  model="{researcher_model}"
)
```

**If strategy is `parallel`:** Analyze the phase goal to identify 2-4 distinct domains. Typical domain splits:
- backend/api, frontend/ui, config/infra
- data-model, business-logic, integration
- Or any natural 2-4 domain groupings based on the phase goal

For each domain, spawn a researcher in parallel:

```
# Spawn all researchers concurrently using Task()
# Each gets a scoped focus and writes to a domain-specific file

Task(
  prompt="First, read ~/.claude/skills/nick-plan-phase/references/agents/nick-phase-researcher.md for your role and instructions.

  <objective>Research Phase {phase_number}: {phase_name}
  Answer: 'What do I need to know to PLAN this phase well?'

  DOMAIN FOCUS: {domain_name} -- Focus your research ONLY on {domain_description}. Other domains are being researched by parallel agents. Do NOT research outside your domain scope.</objective>

  <domain_scope>{domain_name}: {domain_description}</domain_scope>

  <files_to_read>
  - .planning/STATE.md
  - .planning/ROADMAP.md (Phase {N} section)
  - .planning/REQUIREMENTS.md (Phase {N} requirements)
  - {phase_dir}/*-CONTEXT.md (if exists)
  </files_to_read>

  <output>Write to: {phase_dir}/{padded_phase}-RESEARCH-{domain_slug}.md</output>",
  subagent_type="general-purpose",
  model="{researcher_model}"
)
```

Wait for ALL researchers to complete. If any returns `## RESEARCH BLOCKED`, display blocker and offer options. If all return `## RESEARCH COMPLETE`, continue.

Handle return: `## RESEARCH COMPLETE` -> continue. `## RESEARCH BLOCKED` -> display blocker, offer options.

## 4. Spawn Planner

```
Task(
  prompt="First, read ~/.claude/skills/nick-plan-phase/references/agents/nick-planner.md for your role and instructions.

  <planning_context>
  Phase: {phase_number}, Mode: {standard|gap_closure}
  Phase directory: {phase_dir}
  </planning_context>

  <files_to_read>
  - .planning/STATE.md
  - .planning/ROADMAP.md
  - .planning/REQUIREMENTS.md
  - {phase_dir}/*-RESEARCH*.md (if exists -- may be single or multiple domain files)
  - {phase_dir}/*-CONTEXT.md (if exists)
  - {phase_dir}/*-VERIFICATION.md (if --gaps)
  - {phase_dir}/*-UAT.md (if --gaps)
  </files_to_read>

  <downstream_consumer>
  Output consumed by /gsd:execute-phase. Plans need frontmatter, XML tasks, verification, must_haves.
  </downstream_consumer>",
  subagent_type="general-purpose",
  model="{planner_model}"
)
```

Handle return: `## PLANNING COMPLETE` -> continue. `## CHECKPOINT REACHED` -> present to user. `## PLANNING INCONCLUSIVE` -> show attempts, offer options.

## 5. Verify Plans (conditional)

**Skip if:** `--skip-verify` flag or `plan_checker_enabled=false`.

```
Task(
  prompt="First, read ~/.claude/skills/nick-plan-phase/references/agents/nick-plan-checker.md for your role and instructions.

  <verification_context>
  Phase: {phase_number}, Goal: {goal}
  Phase directory: {phase_dir}
  </verification_context>

  <files_to_read>
  - {phase_dir}/*-PLAN.md (all plans)
  - .planning/REQUIREMENTS.md (Phase {N} requirements)
  - {phase_dir}/*-CONTEXT.md (if exists)
  </files_to_read>",
  subagent_type="general-purpose",
  model="{checker_model}"
)
```

Handle return: `## VERIFICATION PASSED` -> continue. `## ISSUES FOUND` -> revision loop.

## 6. Revision Loop (max 3)

If checker returns ISSUES FOUND, re-spawn planner with issue list and plan paths:

```
Task(
  prompt="First, read ~/.claude/skills/nick-plan-phase/references/agents/nick-planner.md for your role and instructions.

  <revision_context>
  Phase: {phase_number}, Mode: revision
  Phase directory: {phase_dir}
  </revision_context>

  <files_to_read>
  - {phase_dir}/*-PLAN.md (existing plans)
  </files_to_read>

  <checker_issues>
  {structured issues from checker}
  </checker_issues>",
  subagent_type="general-purpose",
  model="{planner_model}"
)
```

After planner returns, re-spawn checker (step 5). Max 3 iterations. If max reached, display remaining issues and offer: force proceed, provide guidance, or abandon.

## 7. Present Results

Display plan count, wave breakdown, and offer next steps:

```
NICK > PHASE {X} PLANNED

Phase {X}: {Name} -- {N} plan(s) in {M} wave(s)

| Wave | Plans | What it builds |
|------|-------|----------------|

Research: {Completed | Used existing | Skipped}
Verification: {Passed | Passed with override | Skipped}

Next: /gsd:execute-phase {X}
```
