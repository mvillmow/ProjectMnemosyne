---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY module-consolidation plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extending a main() function with new sub-checks, (5) consolidating two functions with the same name but different signatures, (6) extracting duplicated git commit-if-dirty / push-branch mechanics into a shared helper while preserving wrapper patch seams."
category: architecture
date: 2026-06-26
version: "2.2.0"
history: dry-refactoring-plan-assumption-audit.history
user-invocable: false
verification: unverified
tags: [dry, refactoring, module-consolidation, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, git-utils, patch-seam, push-branch, commit-if-dirty]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture hidden assumptions that invalidate DRY extraction plans before implementation, including ProjectHephaestus issue #1384 planning risks around extracting duplicated commit-if-dirty and push-branch mechanics into `hephaestus/automation/git_utils.py` |
| **Outcome** | Plan guidance only. The #1384 implementation was not executed end-to-end; exact line numbers, duplicate locations, issue comments, branch state, CI state, and downstream callers were not independently verified during planning. |
| **Verification** | unverified — plan not implemented or CI-confirmed |
| **History** | [changelog](./dry-refactoring-plan-assumption-audit.history) |

## When to Use

- Planning to merge two modules into one canonical module (DRY consolidation)
- Replacing an existing module with a delegation shim that re-exports from the canonical
- Porting test classes from one file to another during a refactor
- Adding new public functions to an existing module within a package
- Extending a `main()` function with new sub-checks
- Adding a `from packaging.version import Version` (or any ecosystem dependency) to a new function
- Two modules share a function name with different signatures
- Extracting values from structured config files (TOML, YAML) using regex — always verify section-boundary behavior with a cross-section test case
- Extracting duplicate git operations into `git_utils.py` while keeping local wrapper methods in place for tests and callers that patch those wrappers
- Planning a behavior-preserving push helper where a plain `git push` must not silently become force, force-with-lease, or a different remote/refspec
- Moving commit helpers across module boundaries where importing the old helper at module import time could create a cycle
- Reviewing a plan that lists exact duplicate line numbers without showing a current grep/diff proof

## Proposed Workflow

<!-- Validator compatibility: ## Verified Workflow -->

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms.

### Quick Reference

```bash
# Before writing the plan, run these audits:

# 1. Check __all__ in affected __init__.py files
grep -n "__all__" hephaestus/<package>/__init__.py

# 2. Find early-exit paths in the target main()
grep -n "return\|sys.exit" hephaestus/<module>.py | head -30

# 3. Count test classes in the file being replaced/shimmed
grep -n "^class Test" tests/unit/<path>/test_<module>.py | wc -l

# 4. Verify dependency is declared
grep "packaging" pyproject.toml

# 5. Find all callers of the function being renamed/consolidated
grep -rn "from hephaestus.scripts_lib import\|from hephaestus.validation import" hephaestus/ tests/ scripts/

# 6. Find same-name functions across both modules
grep -rn "^def <function_name>" hephaestus/<module_a>.py hephaestus/<module_b>.py

# 7. For git-operation extraction, prove duplicate ownership and current semantics
rg -n "commit_if|commit_changes|git push|force-with-lease|_push_branch|run\\(" hephaestus/automation tests

# 8. Before deleting any wrapper, prove it is not a patch seam
rg -n "patch\\.object\\([^\\n]*_(commit|push)|patch\\([^\\n]*_(commit|push)|_commit_if_changes|_push_branch" tests hephaestus/automation
```

### Detailed Steps

1. **Audit `__all__` in every `__init__.py` that re-exports from the canonical module.**
   Before adding functions to `validation/python_version.py`, read `hephaestus/validation/__init__.py` in full.
   If it has an explicit `__all__`, the new symbols MUST be added or `from hephaestus.validation import new_function` will raise `AttributeError`.

2. **Trace every early-return path in `main()` before extending it.**
   Open the target `main()` function and list every `return` / `sys.exit` statement.
   Any new sub-checks added must not be gated behind an early exit that already existed (e.g., `if args.json: ... return 0`).
   Fix: move ALL check calls before the format-branching block, then use results in both JSON and text paths.
   Each output-mode branch (JSON, plain text, quiet) must invoke the same set of checks.

3. **Test delegation shims must import test CLASSES, not just symbols.**
   If replacing a test file with an import-only shim, the shim must import the test **classes** themselves:
   ```python
   # CORRECT: pytest re-discovers imported test classes at module scope
   from tests.unit.validation.test_python_version import TestFoo, TestBar

   # WRONG: pytest collects zero tests from import-only shims of non-test symbols
   from hephaestus.scripts_lib.check_python_version_consistency import extract_pyproject_versions
   ```
   Count `grep "^class Test" | wc -l` in the source test file; verify they all exist in the destination before shimming.

4. **Verify runtime dependencies before adding new imports.**
   For any `import X` inside a new function, confirm `X` appears in `[project.dependencies]` in `pyproject.toml`.
   `packaging` is common in the Python ecosystem but not universal — check before assuming.

5. **When two modules share a function name with incompatible signatures, add a new name — do not alias.**
   If the canonical module has `extract_pyproject_versions(path: Path)` and the source module has
   `extract_pyproject_versions(content: str)`, a shim alias re-exporting the path version under the
   string name causes silent behavioral regression: `"".is_file()` returns `False`, so every
   string-content call returns `{}`.

   Fix pattern:
   - Keep `extract_pyproject_versions(path: Path)` unchanged for existing callers in the canonical module.
   - Extract a private helper `_extract_versions_from_text(content: str)` that both implementations delegate to.
   - Add a new public function `extract_pyproject_versions_str(content: str)` in the canonical module that
     calls `_extract_versions_from_text`.
   - In the shim, alias `extract_pyproject_versions_str as extract_pyproject_versions` so the source module's
     callers get the string-based version without renaming their calls.
   - Add both names to `__all__` in `validation/__init__.py`.

   ```python
   # canonical module (validation/python_version.py)
   def _extract_versions_from_text(content: str) -> dict[str, ...]:
       """Shared implementation — used by both public entry points."""
       ...

   def extract_pyproject_versions(path: Path) -> dict[str, ...]:
       """Existing callers use this; unchanged."""
       content = path.read_text()
       return _extract_versions_from_text(content)

   def extract_pyproject_versions_str(content: str) -> dict[str, ...]:
       """New entry point for callers that already have the file content."""
       return _extract_versions_from_text(content)

   # shim (scripts_lib/check_python_version_consistency.py)
   from hephaestus.validation.python_version import (
       extract_pyproject_versions_str as extract_pyproject_versions,  # preserve caller contract
       ...
   )
   ```

6. **For git-operation extraction, verify current duplicate bodies before scoping.**
   Exact line numbers and duplicate locations in an issue or plan are stale by default.
   Re-run `rg` against current HEAD, open every claimed duplicate body, and classify whether the
   repeated code is identical mechanics or a local wrapper seam that callers/tests still own.
   The extraction is only complete when a duplicate-source grep proves the mechanics live in
   `git_utils.py` only, while intentional wrappers remain as thin delegates.

7. **Preserve wrapper patch seams when moving mechanics.**
   If tests patch `_review_phase.py`, `address_review.py`, or
   `implementer_phase_runner.py` wrapper methods, do not delete those methods just because the
   implementation moved. Keep one-line wrappers that delegate to `git_utils.py`, and update only
   tests whose assertion is explicitly about the new shared helper. Deleting a wrapper turns a DRY
   cleanup into a test API break.

8. **Treat push semantics as behavior, not plumbing.**
   A helper named "push branch" must preserve the existing command shape unless the issue explicitly
   asks for a policy change. A plain `git push` must not become `--force`, `--force-with-lease`, a
   different remote, or a changed refspec as an incidental DRY cleanup. Write the expected subprocess
   call shape into tests so mocks cannot hide a semantic drift.

9. **Avoid import cycles by importing legacy commit helpers lazily.**
   If the shared helper must call `pr_manager.commit_changes`, import it inside the function instead
   of at module import time until the dependency graph is proven acyclic. The plan should call this
   out as a temporary cycle-avoidance boundary, not leave the import location to implementation
   guesswork.

10. **Lock return-shape compatibility before normalizing helper APIs.**
    If `AddressReviewer._commit_if_changes` currently returns `None`, assume that shape is
    intentional until source and downstream callers prove otherwise. A shared helper can return a
    richer value internally, but wrapper methods must preserve their historical public return shape.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Plan claimed "no signature change needed" | Shim re-exported `extract_pyproject_versions(path)` under the same name as the source module's `extract_pyproject_versions(content)` | `"".is_file()` returns `False`; every string-content caller received `{}` silently | When same-name collision exists, a shim alias at the wrong layer is a silent regression — add a new name instead |
| Import-only test shim | Replaced test file body with `from hephaestus.scripts_lib import extract_pyproject_versions` | pytest collects zero tests; 400+ lines of test classes do not teleport via non-test symbol imports | Test delegation shims must import test classes: `from tests.unit.validation.test_python_version import TestFoo, TestBar` |
| Added new sub-checks after `if args.json:` | Placed new `check_*` calls after an existing `if args.json: ...; return 0` block | JSON callers exit before reaching the new checks; CI never sees the new coverage | List all early exits in `main()` first; move check calls above the format-branching block |
| Did not update `validation/__init__.py __all__` | Added 8+ new functions to `python_version.py` without updating the package's `__init__.py` | `from hephaestus.validation import new_function` raises `AttributeError` even though the function exists | Grep all `__init__.py` files that import from the modified module; add every new symbol to `__all__` |
| Assumed `packaging` is a declared dependency | Used `from packaging.version import Version` in a new function | `packaging` may not be in `[project.dependencies]`; runtime `ImportError` on CI | `grep packaging pyproject.toml` before adding the import |
| R2 — DOTALL regex crosses TOML sections | Ported `_extract_versions_from_text` inherited `re.DOTALL` from `_extract_via_regex:113`. With DOTALL, `\[tool\.mypy\].*?python_version` lazy-matches past blank lines and `[tool.other]` headers — `test_mypy_version_not_crossed_from_other_section` fails. | Fix: use scripts_lib's section-bounded negative-lookahead `\[tool\.mypy\]\n(?:(?!\[).+\n)*?python_version` instead. `_extract_via_regex` now delegates to `_extract_versions_from_text`, eliminating the DOTALL regex entirely. | Always use section-bounded negative-lookahead `(?:(?!\[).+\n)*?` for TOML section extraction — never `re.DOTALL` across sections. |
| R0/R1 — Wrong test count stated as "44" | Plans stated "44 test functions" but actual scripts_lib test file has 35 functions / 9 classes. | Count test functions by direct grep before writing the plan (`grep -c "def test_" file`). | Verify counts by reading the actual file before stating them in a plan. |
| #1384 — Treat stale duplicate coordinates as proof | Plan cited exact duplicate locations for commit-if-dirty and push-branch mechanics without independently re-running grep during planning | Current file contents and line numbers can drift; duplicate counts may already have changed | Before implementation, re-run `rg` and open each body; use current grep output as the scope, not stale issue coordinates |
| #1384 — Delete wrapper methods after extraction | Planned to move mechanics into `git_utils.py` without proving wrapper methods are not patched in tests | `_review_phase.py`, `address_review.py`, and `implementer_phase_runner.py` wrappers may be test patch seams and local API boundaries | Keep wrapper delegates unless a current grep proves no caller/test patches them; patch seams are part of behavior |
| #1384 — Normalize push mechanics too aggressively | A shared push helper could accidentally replace plain `git push` with force or force-with-lease semantics | Push policy is user-visible and branch-protection-sensitive; changing it during DRY cleanup is a behavioral change | Preserve the exact current command shape and assert it in focused tests |
| #1384 — Import `commit_changes` at module load | Shared `git_utils.py` helper could import `pr_manager.commit_changes` at top level | `git_utils` and `pr_manager` may form an import cycle once helpers move across automation modules | Use a function-local lazy import until the dependency graph is proven acyclic |
| #1384 — Change `AddressReviewer._commit_if_changes` return shape | Shared helper APIs can tempt a return-value cleanup | Existing `AddressReviewer` compatibility may intentionally be `None`; callers/tests may rely on that | Preserve wrapper return shape even if the shared helper returns richer internal state |
| #1384 — Trust mocks as subprocess proof | Tests that mock `run` or `commit_changes` can go green while real Git failure behavior is never exercised | Mocks validate call wiring, not Git's runtime failure modes or stderr/exit-code behavior | Pair mock assertions with at least one subprocess-shaped failure-path test or explicitly mark the gap unverified |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
- [ ] Re-ran duplicate-source grep on current HEAD; every claimed duplicate body opened and classified
- [ ] Preserved wrapper patch seams; `rg "patch.*_commit|patch.*_push|_commit_if_changes|_push_branch"` checked in tests and source
- [ ] Push helper preserves exact prior command shape; no incidental `--force`, `--force-with-lease`, remote, or refspec change
- [ ] `pr_manager.commit_changes` imported lazily if needed to avoid a `git_utils` import cycle
- [ ] Wrapper return shapes preserved, especially `AddressReviewer._commit_if_changes -> None`
- [ ] Duplicate-source grep after refactor proves mechanics moved to `git_utils.py` only, with wrappers as intentional delegates
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1384 Planning Risks

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| Exact duplicate line numbers and locations are current | UNVERIFIED | Re-run `rg` on current HEAD before scoping the extraction |
| GitHub issue #1384 wording is the source of truth | UNVERIFIED | Fetch issue body/comments and compare against current code; stale wording should lose to live code |
| Push semantics can be generalized while extracting | RISK | Preserve plain `git push` semantics unless a separate issue changes push policy |
| Wrapper methods are disposable after shared helper extraction | RISK | Preserve wrappers in `_review_phase.py`, `address_review.py`, and `implementer_phase_runner.py` when tests/callers patch them |
| `pr_manager.commit_changes` can be imported at module import time | RISK | Prefer function-local lazy import until no `git_utils` import cycle exists |
| `AddressReviewer._commit_if_changes` should return the shared helper value | RISK | Preserve historical `None` return shape unless all downstream callers prove otherwise |
| Mocked `run` / `commit_changes` tests prove Git behavior | RISK | They prove wiring only; subprocess failure behavior remains unverified without a real or subprocess-shaped failure test |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Issue #1384 planning for extracting duplicated commit-if-dirty and push-branch mechanics into `hephaestus/automation/git_utils.py` | unverified — implementation not executed; exact duplicate locations, issue comments, branch/CI state, and downstream callers not checked live during planning |
