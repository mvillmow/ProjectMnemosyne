---
name: planning-deferred-stdlib-import-regression-guard
description: "Plan narrowly-scoped Python deferred-stdlib-import cleanup when an issue names specific helpers. Use when: (1) an issue asks to move local stdlib imports to module scope, (2) cited import evidence may be stale, (3) the right guard is a focused AST regression test rather than a repo-wide lazy-import ban."
category: testing
date: 2026-06-30
version: "1.0.0"
user-invocable: false
verification: unverified
tags:
  - planning
  - python
  - stdlib-imports
  - deferred-imports
  - ast-tests
  - regression-guard
  - projecthephaestus
---

# Planning: Deferred Stdlib Import Regression Guard

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-30 |
| **Objective** | Capture planning guidance for ProjectHephaestus issue #1465: move issue-named deferred stdlib imports to module scope and guard against regression without overclaiming a repo-wide import policy. |
| **Outcome** | Planning artifact only. The live-tree audit suggested the issue's `ci_driver.py` evidence was stale and the only remaining actionable deferred stdlib import was `loop_runner.py::_make_work_report_path` importing `tempfile` locally. |
| **Verification** | unverified — the implementation, RED test, GREEN test run, Ruff/isort behavior, and CI were not executed end-to-end before this capture. |

## When to Use

- A Python issue names one or more helpers that perform deferred local stdlib imports and asks to move those imports to module scope.
- A plan cites line numbers or files where the import may already have moved or disappeared.
- You need a regression guard for specific issue-named helpers, not a broad policy banning all function-local imports.
- Reviewers need risks called out around stale issue evidence, AST inspection shape, unused imports, and import-cost assumptions.

## Verified Workflow

> **Warning (Proposed Workflow):** This workflow has not been validated end-to-end. Treat it as a hypothesis until the focused test fails before the code change, passes after the code change, and CI confirms the result.
>
> This section uses the validator-required `Verified Workflow` heading, but the actual workflow is proposed and the skill's verification level is `unverified`.

### Quick Reference

```bash
# 1. Re-audit the cited files on the live tree before editing.
rg -n "import (re|tempfile)|from .* import" \
  hephaestus/automation/ci_driver.py \
  hephaestus/automation/_review_utils.py \
  hephaestus/automation/loop_runner.py

# 2. Confirm whether the issue's cited ci_driver.py evidence is stale.
rg -n "parse_json_block|\\bre\\." hephaestus/automation/ci_driver.py hephaestus/automation/_review_utils.py

# 3. Add the narrow RED guard for issue-named helpers only.
pixi run python -m pytest tests/unit/automation/test_loop_runner.py -q --no-cov

# 4. Move only the remaining actionable stdlib import to module scope.
# Expected target from planning: hephaestus/automation/loop_runner.py imports tempfile at top level.

# 5. Re-run focused tests and formatter/lint checks before broad verification.
pixi run python -m pytest tests/unit/automation/test_loop_runner.py tests/unit/automation/test_ci_driver.py -q --no-cov
pixi run ruff check hephaestus/automation/loop_runner.py tests/unit/automation/test_loop_runner.py
pixi run ruff format --check hephaestus/automation/loop_runner.py tests/unit/automation/test_loop_runner.py
```

### Detailed Steps

1. **Verify the issue premise against the live tree.** Open each cited file and search by symbol, not line number. For issue #1465 planning, `ci_driver.py` appeared to have already moved the cited `re` usage into `_review_utils.parse_json_block`; treat that as stale evidence unless current code proves otherwise.
2. **Identify the actionable local stdlib import.** The plan found `loop_runner.py::_make_work_report_path` importing `tempfile` inside the function. That is the narrow production edit candidate.
3. **Prefer a targeted AST guard over a repo-wide lazy-import ban.** Broad "no local stdlib imports" checks can create noisy failures for legitimate lazy imports, optional dependencies, circular-import avoidance, platform guards, and import-cost-sensitive paths. Guard only the issue-named helpers unless the issue explicitly establishes a wider policy.
4. **Write the test so it proves the specific regression.** The test should parse the source for the named function and assert there are no local `ast.Import` or `ast.ImportFrom` nodes in that function's direct body. If using `ast.walk(function_node)`, account for nested functions/classes so a nested local import is not accidentally attributed to the parent helper.
5. **Run RED before touching production code.** The new guard should fail while `tempfile` is still local inside `_make_work_report_path`. If it does not fail, fix the test before moving the import.
6. **Move `tempfile` to module scope and do not reintroduce `re` into `ci_driver.py`.** Let Ruff/isort decide placement in the existing import block. Check that the moved import is used and that `ci_driver.py` has no unused `re` import.
7. **Run focused verification, then broader repo checks as appropriate.** At minimum run the targeted loop-runner test and the adjacent ci-driver test file, then Ruff check/format. Treat full-suite/CI as pending until actually observed.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Trusted the issue's cited `ci_driver.py` import evidence | Planned as if `ci_driver.py` still had the deferred `re` import named by the issue | Live-tree planning found the `re` usage had apparently moved into `_review_utils.parse_json_block`, so the cited evidence may be stale | Re-audit current files by symbol before editing; stale issue evidence should narrow scope, not cause a no-op or unused import |
| Treated every deferred stdlib import as policy debt | Considered a broad guard banning local stdlib imports across the repository | Broad bans flag legitimate lazy imports and create noisy failures unrelated to the issue | Use a narrow AST regression test for the issue-named helpers unless a repo-wide policy is explicitly accepted |
| Assumed the AST helper was obviously correct | Proposed walking a function node to find nested import nodes | `ast.walk(function_node)` can traverse nested function/class bodies, which may misattribute an intentionally nested local import to the outer helper | Inspect direct statements or explicitly skip nested scopes when asserting "no local imports in this helper" |
| Assumed moving `tempfile` is behavior-free | Planned to move `tempfile` to module scope without measuring import cost or startup/circular effects | The assumption is likely safe for stdlib `tempfile`, but it was not proven during planning | State the assumption as unverified and let implementation verification catch import-cost, startup, and circular-dependency surprises |

## Results & Parameters

| Item | Value |
|------|-------|
| Originating context | ProjectHephaestus issue #1465 planning |
| Verification level | `unverified` — plan only; no RED/GREEN test run or CI result observed |
| Files relied on during planning | `hephaestus/automation/ci_driver.py`, `hephaestus/automation/_review_utils.py`, `hephaestus/automation/loop_runner.py`, adjacent loop-runner and ci-driver unit tests |
| Likely production edit | Move `tempfile` from `loop_runner.py::_make_work_report_path` to module scope |
| Likely stale evidence | The issue's cited `ci_driver.py` deferred `re` import appeared already resolved by `_review_utils.parse_json_block` |
| Preferred guard | Focused AST test covering issue-named helpers only |
| Reviewer risks | Prove RED before production edit; avoid unused `re`; avoid AST false positives from nested scopes; do not overclaim a general deferred-import policy |
