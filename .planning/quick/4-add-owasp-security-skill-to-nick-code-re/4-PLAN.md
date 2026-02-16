---
phase: quick-4
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - skills/nick-review/references/agents/nick-code-reviewer.md
autonomous: true
must_haves:
  truths:
    - "When owasp-security skill is loaded, reviewer produces a dedicated Security Findings section in REVIEW.md separate from general severity findings"
    - "Each security finding in the Security Findings section maps to a specific OWASP Top 10 category"
    - "REVIEW.md frontmatter includes owasp_findings count when security skill is loaded"
    - "Security findings appear in BOTH the severity-graded sections AND the dedicated Security Findings section"
  artifacts:
    - path: "skills/nick-review/references/agents/nick-code-reviewer.md"
      provides: "Updated code reviewer agent with OWASP integration"
      contains: "Security Findings"
  key_links:
    - from: "review_protocol Step 3"
      to: "owasp-security skill"
      via: "OWASP Top 10 category mapping instructions"
      pattern: "OWASP Top 10"
    - from: "severity_levels"
      to: "Security Findings section"
      via: "cross-reference guidance"
      pattern: "Security Findings section"
    - from: "report_output frontmatter"
      to: "owasp_findings field"
      via: "conditional field when owasp-security loaded"
      pattern: "owasp_findings"
---

<objective>
Enhance the nick-code-reviewer agent with dedicated OWASP security reporting.

Purpose: When the owasp-security skill is loaded, the reviewer currently treats security findings the same as any other finding. This change adds a dedicated Security Findings section to REVIEW.md, maps findings to OWASP Top 10 categories, and adds explicit security review instructions -- making security audit results immediately visible and actionable.

Output: Updated `skills/nick-review/references/agents/nick-code-reviewer.md` with four enhancements.
</objective>

<execution_context>
@/Users/nick/.claude/get-shit-done/workflows/execute-plan.md
@/Users/nick/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@skills/nick-review/references/agents/nick-code-reviewer.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Add security review instructions and severity cross-reference guidance</name>
  <files>skills/nick-review/references/agents/nick-code-reviewer.md</files>
  <action>
Make three changes to the nick-code-reviewer.md agent definition:

**Change 1: Enhance Step 3 in `<review_protocol>` (after line 68, before "Record findings with:")**

After the existing bullet list in Step 3 (the "Anti-patterns" bullet), add a new subsection:

```markdown
**Security Review (when owasp-security skill is loaded):**

When the owasp-security skill was loaded in Step 2, perform additional security-focused review:

- For each file handling auth, validation, user input, API endpoints, secrets, or env vars:
  - Check against OWASP Top 10 2021 categories (A01:Broken Access Control through A10:SSRF)
  - For each security finding, record which OWASP category it maps to (e.g., "A03:Injection -- unsanitized user input in SQL query")
  - Check for issues the general review might miss: insecure defaults, missing security headers, overly permissive CORS, hardcoded credentials, timing attacks in auth comparisons
- Security findings get their normal severity classification (Critical/High/Medium/Low) AND are collected separately for the Security Findings section in REVIEW.md
```

**Change 2: Add cross-reference note to `<severity_levels>` section**

At the end of the `<severity_levels>` section (after the Low subsection, before `</severity_levels>`), add:

```markdown
### Security Findings Cross-Reference

When the owasp-security skill is loaded, security-related findings are classified under their normal severity level above AND duplicated in the dedicated "Security Findings" section of REVIEW.md. This ensures security issues are both (a) counted toward PASS/FAIL via severity and (b) visible as a consolidated security audit trail with OWASP category mappings.
```

**Change 3: Expand the owasp-security row in the Step 2 skill loading table (line 43)**

Update the loading strategy bullets (lines 46-51) by adding one more bullet after "Multiple skills can apply..." (line 49):

```markdown
- When owasp-security is loaded: also read OWASP Top 10 2021 category descriptions from the skill's references to accurately map findings to categories (A01-A10)
```
  </action>
  <verify>Read the modified file. Confirm: (1) Step 3 has "Security Review (when owasp-security skill is loaded)" subsection with OWASP Top 10 instructions, (2) severity_levels has "Security Findings Cross-Reference" subsection, (3) loading strategy has owasp-specific bullet.</verify>
  <done>Step 3 contains explicit OWASP Top 10 review checklist, severity_levels documents the cross-reference pattern, and skill loading has owasp-specific guidance.</done>
</task>

<task type="auto">
  <name>Task 2: Add Security Findings section and owasp_findings field to REVIEW.md template</name>
  <files>skills/nick-review/references/agents/nick-code-reviewer.md</files>
  <action>
Make two changes to the `<report_output>` section of nick-code-reviewer.md:

**Change 1: Add `owasp_findings` to frontmatter template**

In the frontmatter YAML block (lines 192-210), add after the `findings:` block (after `low: N` on line 209):

```yaml
security:
  owasp_loaded: true | false
  owasp_findings: N        # 0 if owasp loaded but no findings; omit entire security block if not loaded
```

**Change 2: Add Security Findings section to the REVIEW.md body template**

In the body markdown template, add a new section AFTER "## Best Practices Review" (after line 250, "- [what was done well]") and BEFORE "## Test Results":

```markdown
## Security Findings

[Only include this section when owasp-security skill was loaded in Step 2]

| # | OWASP Category | Severity | File | Finding | Recommendation |
|---|----------------|----------|------|---------|----------------|
| 1 | A01:Broken Access Control | Critical | `file.ts:42` | [description] | [fix] |
| 2 | A03:Injection | High | `file.ts:88` | [description] | [fix] |

[If no security findings: "No security issues identified. OWASP Top 10 categories reviewed against all applicable files."]

**Categories Reviewed:** [List which A01-A10 categories were applicable based on the codebase]
```
  </action>
  <verify>Read the modified file. Confirm: (1) frontmatter template has `security:` block with `owasp_loaded` and `owasp_findings` fields, (2) body template has "## Security Findings" section with OWASP category table between Best Practices Review and Test Results, (3) section has conditional inclusion note and empty-state text.</verify>
  <done>REVIEW.md template includes owasp_findings in frontmatter and a dedicated Security Findings section with OWASP category mapping table, conditional on owasp-security skill being loaded.</done>
</task>

</tasks>

<verification>
- Read the full modified `skills/nick-review/references/agents/nick-code-reviewer.md`
- Verify all four enhancement areas are present:
  1. Step 3 has OWASP-specific security review instructions
  2. Severity levels has cross-reference guidance
  3. Skill loading has owasp-specific bullet
  4. REVIEW.md template has Security Findings section + frontmatter field
- Verify the file still has valid XML structure (all tags properly opened/closed)
- Verify no existing content was accidentally removed or altered
</verification>

<success_criteria>
- nick-code-reviewer.md contains all four OWASP enhancements
- Existing review protocol, severity definitions, and report structure are preserved
- Security Findings section appears in REVIEW.md template with OWASP Top 10 category table
- Frontmatter includes owasp_findings count field
- File parses correctly (no broken XML tags)
</success_criteria>

<output>
After completion, create `.planning/quick/4-add-owasp-security-skill-to-nick-code-re/4-SUMMARY.md`
</output>
