---
name: update-workflow
description: Updates an existing workflow test document based on code changes.
  Preserves manual tweaks (quirks, test data, timing notes) while updating
  fields, validation, routes, and data flow that changed in code.
allowed-tools: Read, Bash, Grep, Glob, LS, Task, Write
argument-hint: [workflow-file-path] [optional: git ref to diff against, defaults to HEAD~1]
---

You are updating an EXISTING workflow test document based on recent code changes.
Workflow file: $ARGUMENTS

CRITICAL RULE: Preserve all manual additions the user made — known quirks, adjusted
test data, timing notes, field type reference entries. Only change what the CODE
changed. The user's manual tweaks are hard-won knowledge from actual test runs.

## Phase 1: Read the existing workflow

Read the workflow file specified in $ARGUMENTS (e.g., workflows/partij-aanmaken.md).
Parse and understand:
- Which routes/pages it covers
- All form fields listed (labels, types, constraints)
- Test data values
- Data flow verification steps
- Field type reference entries
- Known quirks section

Identify the feature area from the workflow's routes and component references.

## Phase 2: Find what changed

Run a git diff to find changed files in the feature area. Use the second argument
as the diff base if provided, otherwise default to HEAD~1.

```bash
git diff HEAD~1 --name-only -- src/
```

Filter the changed files to only those relevant to this workflow's feature area.
If no relevant files changed, tell the user: "No code changes detected that affect
this workflow. The workflow is up to date."

If changes ARE found, categorize them:

- **Schema changes**: validation files, zod/yup schemas → fields added/removed/modified
- **Component changes**: form components, input components → UI labels, field types changed
- **API changes**: route.ts, server actions → endpoints, request/response shapes
- **Page changes**: page.tsx files → routes, data display, redirects
- **Dropdown changes**: combobox/select components → data sources, options, search behavior

## Phase 3: Targeted exploration

ONLY spawn explorers for the categories that have changes. Don't re-analyze
unchanged code — it wastes tokens and time.

For each category with changes, spawn a focused explorer:

### If schema/form fields changed:
"These files changed: [list files]
Read each changed file. Compare against what the existing workflow documents:
[paste the Form fields section from the existing workflow]
Report ONLY the differences:
- New fields added (with full details: label, type, constraints, default)
- Fields removed
- Fields with changed constraints (old → new)
- Fields with changed types
- Changed labels or placeholder text"

### If dropdown/combobox components changed:
"These files changed: [list files]
Read each changed file. Compare against what the existing workflow documents:
[paste the Dropdown/Field type reference sections]
Report ONLY the differences:
- New dropdowns added
- Removed dropdowns
- Changed data sources (different API endpoint, static → async, etc.)
- Changed debounce timing
- Changed interaction patterns"

### If API routes changed:
"These files changed: [list files]
Read each changed file. Compare against what the existing workflow documents:
[paste the API/data flow sections]
Report ONLY the differences:
- New endpoints
- Removed endpoints
- Changed request/response shapes
- Changed side effects (different redirect, different toast message)"

### If page components changed:
"These files changed: [list files]
Read each changed file. Compare against what the existing workflow documents:
[paste the route/verification sections]
Report ONLY the differences:
- New pages/routes
- Changed data display (different columns, different field formatting)
- Changed navigation flow
- New or removed UI elements (buttons, modals)"

Wait for all spawned explorers to return.

## Phase 4: Merge changes

Pass the results to the workflow-composer agent with EXPLICIT merge instructions:

"You are UPDATING an existing workflow, NOT writing from scratch.

Here is the EXISTING workflow (preserve everything not affected by changes):
[paste full existing workflow]

Here are the CODE CHANGES detected:
[paste all explorer outputs]

Rules for merging:
1. KEEP all existing content that is NOT affected by the code changes
2. KEEP the Known quirks section ENTIRELY — these are manual additions
3. KEEP existing test data values unless the field itself changed
4. KEEP Field type reference entries unless the component changed
5. ADD new fields/steps discovered in the changes
6. REMOVE fields/steps for code that was deleted
7. UPDATE constraints, types, labels only where the code actually changed
8. UPDATE data flow steps only if redirects/toasts/API calls changed
9. If a field's constraints changed, keep the existing test data IF it still
   satisfies the new constraints. Only change test data if it would now be invalid.
10. ADD a note at the top: '<!-- Updated: [today's date] based on git changes -->'

Mark every change you make with an inline comment so the user can review:
<!-- UPDATED: constraint changed from max 255 to max 500 -->
<!-- ADDED: new field Gewicht -->
<!-- REMOVED: field Opmerking no longer in schema -->

Output the complete updated workflow."

## Phase 5: Save and report

Save the updated workflow back to the SAME file path.

Report:
1. What changed in the code (list changed files)
2. What was updated in the workflow:
   - Fields added/removed/modified
   - Constraints changed
   - Data flow changes
   - API changes
3. What was PRESERVED (manual quirks, test data, etc.)
4. Inline <!-- comments --> mark every change for easy review
5. Suggest: review the changes, then run /test-flow [workflow-path]