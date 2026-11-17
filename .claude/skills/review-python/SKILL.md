---
name: review-python
description: Read-only Python code review protocol for `.py` scripts. Checks code quality, reproducibility, pandas/numpy correctness, figure generation, and professional standards; produces a report without editing. Use when user says "review this Python script", "check the Python code", "audit the analysis code", "code review on the Python", or when a .py file is touched as part of a paper submission. NOT for running the code — pair with `/audit-reproducibility` for numeric verification.
argument-hint: "[filename or 'all' or 'LectureN']"
allowed-tools: ["Read", "Grep", "Glob", "Write", "Agent", "Task"]
disallowed-tools: ["Edit", "MultiEdit"]
---

# Review Python Scripts

Run the comprehensive Python code review protocol.

## Steps

1. **Identify scripts to review:**
   - If `$ARGUMENTS` is a specific `.py` filename: review that file only
   - If `$ARGUMENTS` is `LectureN`: review all Python scripts matching that lecture
   - If `$ARGUMENTS` is `all`: review all Python scripts in `scripts/py/` and `Figures/*/`

2. **For each script, launch the `python-reviewer` agent** with instructions to:
   - Follow the full protocol in the agent instructions
   - Read `.claude/rules/python-code-conventions.md` for current standards
   - Return its report as its final response (ending with the findings `json` block); this skill saves it to `quality_reports/[script_name]_py_review.md` — the agent is read-only

3. **After all reviews complete**, present a summary:
   - Total issues found per script
   - Breakdown by severity (Critical / High / Medium / Low)
   - Top 3 most critical issues

4. **Leave the Python source files unchanged** — the report goes to the user, who decides what to fix.
   Only produce reports. Fixes are applied after user review.
