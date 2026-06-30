---
name: planning-audit-score-parser-assumptions
description: "Planning discipline for audit-score parser fixes: treat CVSS/vector parsing behavior, fallback field schemas, grep-only usage claims, and line-number citations as unverified until checked live. Use when: (1) reviewing a plan that removes a no-op parser branch or regex special case, (2) a plan relies on pip-audit or vulnerability-report JSON fields without sample/docs verification, (3) a fallback expression uses `or` across numeric fields, (4) regression tests cover only the happy fallback path."
category: testing
date: 2026-06-30
version: "1.0.0"
user-invocable: false
verification: unverified
tags:
  - planning
  - testing
  - parser
  - audit
  - cvss
  - pip-audit
  - vulnerability-report
  - line-number-drift
  - schema-assumptions
  - reviewer-risks
  - numeric-zero
---

# Planning: Audit Score Parser Assumptions

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-30 |
| **Objective** | Capture the planning lessons from a ProjectHephaestus issue #1466 implementation plan that proposed removing an ambiguous bare `pass` from `extract_cvss_score()` while preserving CVSS-vector-only behavior and adding a numeric fallback regression test. |
| **Outcome** | Plan produced only. The claims below are reviewer tasks, not verified facts. |
| **Verification** | unverified - planning artifact only; no implementation, tests, pip-audit docs, or sample JSON were verified end-to-end. |

This skill is about the planning/review discipline around a small parser cleanup: a no-op branch
can look obviously dead, but the surrounding fallback behavior depends on current code shape,
external report schema, and numeric edge cases. The durable lesson is to convert those assumptions
into explicit verification tasks before implementation or review approval.

## When to Use

- Reviewing or authoring a plan that removes a parser regex or no-op special case because it
  "does nothing."
- A plan cites exact file lines for parser behavior or test expectations, but does not include
  the live command output used to prove those lines still exist.
- A plan says a repo-wide `rg` proves a symbol is used only in one no-op branch, but the result is
  summarized rather than pasted.
- A plan relies on pip-audit, vulnerability-report, SARIF, or other security-tool JSON fields
  (`score`, `base_score`, `cvss_score`, etc.) without checking current docs or a sample artifact.
- A fallback expression uses truthiness (`a or b`) across numeric fields where `0`, `0.0`, or
  string `"0"` may be meaningful.
- The proposed regression test covers only the happy path and does not prove behavior for pure
  vectors, arbitrary non-numeric strings, missing fallback fields, or zero-valued scores.

## Verified Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat it as a proposed
> implementation-plan review checklist until CI and live source/schema checks confirm it.

### Quick Reference

```bash
# Verify cited lines against the current tree before implementation/review.
sed -n '70,90p' hephaestus/validation/audit.py
sed -n '50,70p' tests/unit/validation/test_audit.py

# Prove the symbol/behavior scope with output, not a prose summary.
rg -n "CVSS_PATTERN|extract_cvss_score|CVSS:" hephaestus/validation tests

# Find fallback-field consumers and tests before changing truthiness behavior.
rg -n "base_score|cvss_score|score" hephaestus/validation tests

# Verify external report schema from a live artifact or docs before relying on field names.
python3 -m json.tool /tmp/pip-audit-report/pip-audit.json | head -80
```

### Detailed Steps

1. **Treat line-number citations as pointers, not evidence.** The #1466 plan cited
   `hephaestus/validation/audit.py:77-78` as still containing
   `elif isinstance(score_str, str) and CVSS_PATTERN.match(score_str): pass`, and
   `tests/unit/validation/test_audit.py:59-62` as asserting pure CVSS vector strings return
   `None`. Those were plan-time claims. Re-open the current files or grep stable markers before
   editing; line numbers drift quickly and can create false confidence.

2. **Paste or rerun the grep that proves dead-code scope.** The plan assumed
   `rg -n "CVSS_PATTERN|extract_cvss_score|CVSS:" hephaestus/validation tests` proves
   `CVSS_PATTERN` is used only for the no-op branch. That is the right command shape, but the
   output was not included. A reviewer should require the actual result, because a second use in
   docs, tests, fixtures, or a helper changes the safe removal scope.

3. **Verify external schema behavior, not just local fallback intent.** The plan assumed
   pip-audit severity entries may provide numeric `base_score` or `cvss_score` when `score` is a
   CVSS vector. It also assumed `entry.get("base_score") or entry.get("cvss_score")` preserves
   intended fallback behavior. Both are schema/source assumptions unless checked against pip-audit
   docs or a representative JSON artifact.

4. **Review truthiness fallbacks for numeric zero.** If `base_score` can be `0` or `0.0`, the
   expression `entry.get("base_score") or entry.get("cvss_score")` skips it and falls through to
   `cvss_score`. That may be correct or wrong, but it must be deliberate. Use `is not None`
   semantics if zero is a valid score that must be preserved.

5. **Test behavior boundaries, not only the happy fallback.** A regression where `score` is a CVSS
   vector and `base_score` is numeric is necessary, but not sufficient. Reviewers should look for
   tests covering pure vector-only input, arbitrary non-numeric strings, missing fallback fields,
   `base_score=0`, and precedence when both `base_score` and `cvss_score` are present.

6. **Confirm removal of the regex does not change non-target semantics.** If the branch was truly
   no-op, removing `CVSS_PATTERN` and the unused `re` import should leave vector-only strings and
   arbitrary non-numeric strings as non-numeric results. Prove that with tests or a small direct
   call, not by inspection alone.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Trust plan-time file:line citations | Accepted `audit.py:77-78` and `test_audit.py:59-62` as current evidence | The plan did not include live output, and line numbers can drift after any nearby edit | Re-open the current files or grep stable markers before implementing or approving the plan |
| Treat a summarized `rg` as proof | Plan said the `CVSS_PATTERN|extract_cvss_score|CVSS:` search proves the regex is used only in the no-op branch | The command output was not included, so reviewers cannot see missed tests, fixtures, docs, or alternate uses | Paste or rerun the exact grep result; dead-code removals need observable scope evidence |
| Assume pip-audit fallback fields without a sample | Planned around `base_score` / `cvss_score` being available when `score` is a vector | The schema behavior was not verified against docs or sample JSON | Check a real artifact or current docs before relying on field names and precedence |
| Use truthiness fallback for numeric fields | `entry.get("base_score") or entry.get("cvss_score")` was treated as preserving intended fallback behavior | `0` or `0.0` is skipped by `or`, even though zero may be a meaningful numeric score | Decide whether zero is valid; if yes, use explicit `is not None` fallback semantics |
| Add only the happy fallback regression | Planned a test for CVSS vector in `score` plus numeric `base_score` | It does not prove pure vectors remain non-numeric, arbitrary non-numeric strings behave unchanged, missing fields return the expected result, or zero scores are preserved | Pair the happy regression with boundary tests around vector-only, non-numeric, missing, zero, and precedence cases |

## Results & Parameters

### Plan-Time Claims To Re-Verify

- `hephaestus/validation/audit.py:77-78` still contains the bare `pass` branch under
  `CVSS_PATTERN.match(score_str)`.
- `tests/unit/validation/test_audit.py:59-62` still asserts pure CVSS vector strings return
  `None`.
- `rg -n "CVSS_PATTERN|extract_cvss_score|CVSS:" hephaestus/validation tests` shows no use of
  `CVSS_PATTERN` outside the no-op branch and related tests.
- pip-audit report entries can contain a CVSS vector in `score` while numeric fallback fields live
  in `base_score` or `cvss_score`.
- `base_score` / `cvss_score` fallback precedence is correct for zero and for both-fields-present
  cases.

### Reviewer-Risk Checklist

```text
- [ ] Are file:line citations current, or were they re-derived by marker/grep?
- [ ] Is the dead-code search output included and scoped to production code, tests, and fixtures?
- [ ] Was the external pip-audit/report schema verified from a live artifact or current docs?
- [ ] Does the fallback preserve 0 / 0.0 if those are valid scores?
- [ ] Do tests cover vector-only, arbitrary non-numeric, missing fallback, zero, and precedence cases?
- [ ] Does removing the regex/import leave unrelated non-numeric strings unchanged?
```

### ProjectHephaestus #1466 Context

The specific unexecuted plan proposed:

- Remove `CVSS_PATTERN` and the unused `re` import from `extract_cvss_score()`.
- Preserve vector-only CVSS score behavior as non-numeric.
- Add a regression test where `score` is a CVSS vector and numeric `base_score` returns the
  numeric value.

Useful review framing: this is probably a small change, but the risk is not the deleted `pass`
itself. The risk is hidden in unverified evidence, external schema assumptions, and numeric
fallback edge cases.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #1466 implementation plan for `extract_cvss_score()` | unverified; planning lesson only. No implementation, tests, pip-audit docs, or sample JSON were checked end-to-end during this learn capture. |
