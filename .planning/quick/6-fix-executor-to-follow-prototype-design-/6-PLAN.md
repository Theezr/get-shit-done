---
phase: quick-6
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - skills/nick-execute-phase/references/agents/nick-executor.md
autonomous: true
must_haves:
  truths:
    - "Executor extracts specific design tokens (colors, spacing, fonts, borders, shadows) from prototype before implementing"
    - "Executor treats prototype as a design SPECIFICATION with strict visual fidelity, not a loose guide"
    - "Executor performs a post-implementation prototype fidelity check for frontend tasks"
    - "Executor still builds proper framework components (no HTML copy-paste from prototype)"
  artifacts:
    - path: "skills/nick-execute-phase/references/agents/nick-executor.md"
      provides: "Prescriptive prototype-driven execution instructions"
      contains: "design tokens"
  key_links:
    - from: "load_plan step"
      to: "execute_tasks step"
      via: "extracted design tokens inform implementation"
      pattern: "design.tokens|design.specification"
---

<objective>
Rewrite the prototype handling section of the nick-executor agent to be much more prescriptive about visual fidelity.

Purpose: The current "GUIDE, not a template" framing gives the executor too much latitude to deviate from the prototype's visual design. The executor should extract concrete design tokens and treat the prototype as a strict visual specification.
Output: Updated nick-executor.md in both repo and installed locations.
</objective>

<execution_context>
@/Users/nick/.claude/get-shit-done/workflows/execute-plan.md
@/Users/nick/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/STATE.md
@skills/nick-execute-phase/references/agents/nick-executor.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Rewrite prototype handling in nick-executor.md</name>
  <files>skills/nick-execute-phase/references/agents/nick-executor.md</files>
  <action>
Replace lines 132-145 of the `<step name="load_plan">` section in `skills/nick-execute-phase/references/agents/nick-executor.md`. The current text starts at "**If plan frontmatter has `prototype` field:**" and ends before "In SUMMARY.md, note:".

Replace with a rewritten prototype section that has THREE parts:

**Part 1: Design Token Extraction (new)**
When the plan frontmatter has a `prototype` field, read the prototype file and extract a design token inventory BEFORE writing any code:
- Colors: background colors, text colors, accent colors, border colors (exact hex/rgb/hsl values from the CSS)
- Spacing: margins, paddings, gaps (exact values)
- Typography: font families, font sizes, font weights, line heights
- Borders: border-radius values, border widths, border styles
- Shadows: box-shadow values
- Layout: grid/flex structure, column counts, breakpoints, max-widths
- Component hierarchy: what wraps what, nesting structure

Format this as a mental checklist to reference during implementation. Each value should be extracted from the actual CSS in the prototype file, not approximated.

**Part 2: Implementation Rules (rewritten from current)**
Change the framing from "GUIDE, not a template" to "design SPECIFICATION":
- The prototype is a design SPECIFICATION -- match its visual output precisely
- Build proper framework components (React/Next.js/etc), do NOT copy raw HTML into JSX
- Do NOT modify the prototype file
- Do NOT use the prototype for non-visual concerns (API contracts, state management, data flow)
- Every design token extracted in Part 1 MUST appear in the implementation (use Tailwind classes, CSS variables, or inline styles as appropriate to the project's conventions)
- When the prototype uses specific spacing (e.g., `gap: 24px`), use that exact value -- do not round or approximate
- When the prototype uses specific colors, use those exact colors -- do not substitute similar ones
- Component structure should mirror the prototype's nesting hierarchy

**Part 3: Post-Implementation Fidelity Check (new)**
After completing any task that touches frontend/UI files (JSX, TSX, CSS, HTML):
- Re-read the prototype file
- Compare each extracted design token against what was implemented
- Check: layout structure matches (grid cols, flex direction, gaps)
- Check: colors match (background, text, accent, border)
- Check: spacing matches (padding, margin, gap values)
- Check: typography matches (font size, weight, family)
- Check: border-radius and shadows match
- If any token was missed or approximated, fix it before committing the task
- In the task commit message, note: "Prototype fidelity verified"

Keep the existing SUMMARY.md note line (line 145): `In SUMMARY.md, note: "Implemented from prototype: {prototype-file}"`

Do NOT change anything outside the prototype section (lines 132-145 area). The rest of the file must remain identical.
  </action>
  <verify>
Read the updated file. Confirm:
1. The phrase "GUIDE, not a template" is gone
2. "design SPECIFICATION" or "design specification" appears
3. "design token" appears in the extraction section
4. Post-implementation fidelity check section exists
5. "Do NOT copy" rule is still present (proper components, not HTML copy-paste)
6. The rest of the file outside lines ~132-145 is unchanged
  </verify>
  <done>The prototype section in nick-executor.md is rewritten with prescriptive design token extraction, strict specification framing, and a post-implementation fidelity check</done>
</task>

<task type="auto">
  <name>Task 2: Sync installed copy at ~/.claude/</name>
  <files>~/.claude/skills/nick-execute-phase/references/agents/nick-executor.md</files>
  <action>
Copy the updated `skills/nick-execute-phase/references/agents/nick-executor.md` from the repo to `~/.claude/skills/nick-execute-phase/references/agents/nick-executor.md` so the installed runtime copy matches the repo source of truth.

Use `cp` to overwrite the installed copy with the repo copy.
  </action>
  <verify>
Run `diff` between the two files. Output should be empty (files are identical).
```bash
diff skills/nick-execute-phase/references/agents/nick-executor.md ~/.claude/skills/nick-execute-phase/references/agents/nick-executor.md
```
  </verify>
  <done>Installed copy at ~/.claude/ is byte-identical to the repo source copy</done>
</task>

</tasks>

<verification>
1. `grep -c "design SPECIFICATION" skills/nick-execute-phase/references/agents/nick-executor.md` returns 1+
2. `grep -c "design token" skills/nick-execute-phase/references/agents/nick-executor.md` returns 1+
3. `grep -c "GUIDE, not a template" skills/nick-execute-phase/references/agents/nick-executor.md` returns 0
4. `grep -c "fidelity" skills/nick-execute-phase/references/agents/nick-executor.md` returns 1+ (fidelity check)
5. `diff` between repo and installed copy returns empty
</verification>

<success_criteria>
- The executor agent treats prototypes as strict visual specifications, not loose guides
- Design token extraction is mandatory before implementing frontend tasks
- Post-implementation fidelity verification prevents visual drift
- The "build proper components" rule is preserved (no HTML copy-paste)
- Both repo and installed copies are in sync
</success_criteria>

<output>
After completion, create `.planning/quick/6-fix-executor-to-follow-prototype-design-/6-SUMMARY.md`
</output>
