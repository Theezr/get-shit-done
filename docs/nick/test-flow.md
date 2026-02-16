# Automated Test Flow

---
allowed-tools: Read, Bash, Grep, Glob, Task
argument-hint: path/to/workflow-guide.md
---

You orchestrate automated UI testing by reading a workflow guide, delegating test plan creation to a planner agent, and then dispatching a browser tester agent to execute the plan.

## Prerequisites

- Chrome DevTools MCP server must be connected (tools like `take_snapshot`, `click`, `fill` should be available)
- App must be running at the URL specified in the workflow guide
- User must be logged in to the app

## Workflow

### Step 1: Read the Workflow Guide

Read the workflow guide at the path provided in `$ARGUMENTS`. If no argument is provided, ask the user for the workflow file path.

Verify the file exists and contains testable sections (look for headings describing interactive elements and user flows).

### Step 2: Generate Test Plan

Delegate to the `test-planner` agent (Sonnet model) with:
- The full content of the workflow guide
- Instructions to produce a structured, numbered test plan covering all sections
- The test plan should follow the snapshot -> interact -> verify pattern

Prompt for the planner:
```
Read and analyze this workflow guide, then produce an exhaustive step-by-step test plan.

Workflow guide content:
[paste full workflow guide content here]

Generate a complete test plan covering ALL sections (A through G or however many exist).
Each test must follow the snapshot-first pattern and include specific assertions.
Include edge cases, validation tests, and cross-section data flow checks.
For drag-and-drop tests, use the evaluate_script approach.
```

Collect the test plan output.

### Step 3: Execute Test Plan

Delegate to the `browser-tester` agent (Haiku model) with:
- The complete test plan from Step 2
- Instructions to execute every test and report results

Prompt for the browser tester:
```
Execute the following test plan step-by-step. For each test:
1. Follow the steps exactly as written
2. Use take_snapshot before EVERY interaction
3. Report pass/fail with evidence for each assertion
4. Compile a summary report at the end

Start by navigating to the app URL specified in the test plan.

Test Plan:
[paste complete test plan here]
```

Collect the test results.

### Step 4: Report Results

Present a summary to the user:

```
## Test Flow Results

**Workflow:** [workflow guide filename]
**Test Plan:** [N] tests across [M] sections
**Execution:** [timestamp]

### Results
- Passed: X
- Failed: X
- Skipped: X

### Failed Tests
[List each failed test with brief description and reason]

### Key Observations
[Notable findings from the test execution]

### Recommendations
[Suggested fixes or follow-up actions based on failures]
```

## Error Handling

- If the workflow guide file doesn't exist: ask the user for the correct path
- If the test planner returns an empty plan: report the issue and ask the user to check the workflow guide format
- If the browser tester can't connect to the app: report the connection error and suggest checking that the app is running
- If Chrome DevTools MCP tools are not available: inform the user they need to connect the Chrome DevTools MCP server first

## Usage Examples

```
/test-flow workflows/planning-board.md
/test-flow workflows/fleet-management.md
/test-flow docs/test-workflows/relations-page.md
```
