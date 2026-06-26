---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY refactor plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extracting duplicated git commit/push mechanics into a shared helper while preserving patchable wrapper seams, (5) consolidating two functions with the same name but different signatures."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
history: dry-refactoring-plan-assumption-audit.history
tags: [dry, refactoring, module-consolidation, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, git-utils, patch-seams, import-cycle, commit-push]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture hidden assumptions that invalidate narrow DRY refactor plans before implementation, from module consolidation (#1189) through shared git commit/push helper extraction (#1384) |
| **Outcome** | Plan review guidance captured; #1384 content remains unverified and must be checked against current code before implementation |
| **Verification** | unverified — plan not yet implemented or CI-confirmed |
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
- Extracting duplicated git commit/push mechanics into a shared helper while existing tests patch private wrapper methods
- Reviewing a plan that changes helper return values (`None` wrapper to bool helper) or moves commit logic into `git_utils`
- A plan says "force-push" in prose but the intended behavior should remain a plain `git push origin <branch>`

## Verified Workflow

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

# 7. For git commit/push helper extraction, prove wrapper patch seams and duplicate bodies.
rg -n "patch\\.object\\([^)]*_(commit|push|commit_changes|push_changes)|patch\\(\".*_(commit|push|commit_changes|push_changes)" tests hephaestus
rg -n "git commit|git push|commit_changes\\(|push_changes\\(" hephaestus/automation tests

# 8. Prove import direction before moving commit logic into git_utils.
python -c "import hephaestus.automation.git_utils; import hephaestus.automation.pr_manager; print('import OK')"
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

6. **Preserve private wrapper seams when extracting a shared helper.**
   If tests patch wrapper methods such as `_commit_changes`, `_push_changes`, or a runner method that delegates
   to `git_utils`, keep the wrapper method as a thin delegation seam. Moving all call sites directly to the new
   helper may be the cleaner shape in isolation, but it breaks compatibility with older tests and any deliberate
   cycle-break patch surface. The plan must grep the full test tree for private-method patch targets before it
   deletes or bypasses wrappers.

7. **Track bool-return semantics at every caller, not just the new helper.**
   A shared helper returning `bool` can be correct for a phase whose progress gate depends on whether a commit or
   push happened. That same bool can be intentionally ignored by a wrapper whose contract is "try to address review
   threads and continue." The plan should state, caller by caller, whether the return value gates progress or is
   deliberately discarded. Treat a `None`-returning wrapper becoming bool-visible as a semantic change unless tests
   and call sites prove it is harmless.

8. **Keep push behavior literal unless current code proves otherwise.**
   If the issue text says "force-push" but existing code uses `git push origin <branch>`, the refactor should preserve
   the existing plain push unless the task explicitly changes semantics and adds tests for that behavior. A DRY
   extraction should not silently convert push mode, add force flags, or swap the command shape while moving code.

9. **Move shared commit logic into `git_utils` without importing `pr_manager` at module load.**
   A helper in `git_utils` that needs `pr_manager.commit_changes` must avoid creating a `git_utils -> pr_manager ->
   git_utils` cycle. Prefer a lazy import inside the helper or invert the dependency another way. Do not approve a
   plan that states "move it into git_utils" without an import-cycle smoke test and an explicit import-direction note.

10. **Compatibility seams count as requirements.**
    `ImplementationPhaseRunner` direct delegation to `git_utils` may look redundant after helper extraction, but older
    tests may patch that delegation point. A plan for a narrow DRY refactor should preserve the seam first, then let a
    later cleanup remove it after test patch paths and the historical cycle-break workaround are deliberately migrated.

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
| R0/R1 — Wrapper patch seams treated as disposable | Plan extracted duplicated git commit/push mechanics into a shared helper and risked replacing private wrapper bodies with direct helper calls everywhere | Tests may patch private methods on reviewers/runners, and the `ImplementationPhaseRunner -> git_utils` seam may be part of an older cycle-break patch surface | Grep all tests for private wrapper patch targets and keep thin wrappers unless the plan explicitly migrates each patch path |
| Bool helper return value generalized across incompatible callers | Shared helper returned `bool` for ReviewPhase progress gating, then a wrapper that previously returned `None` risked exposing or acting on that bool | ReviewPhase may need bool gating while AddressReviewer intentionally ignores commit/push success as part of its loop contract | Write a caller-by-caller return semantics table: gate, ignore, or preserve wrapper `None` behavior |
| Issue wording changed push semantics by accident | Issue prose said "force-push", so the plan risked adding force-push behavior while extracting shared push code | Existing behavior may be plain `git push origin <branch>`; a DRY refactor should not change push mode without explicit acceptance criteria and tests | Preserve the current command shape by default; verify the actual subprocess argv before implementing |
| Shared helper imported `pr_manager` at module load | Moved commit logic into `git_utils` by adding a top-level import from `pr_manager` | `pr_manager` already depends on git utilities in some flows, so module-load imports can reintroduce an import cycle | Use lazy import inside the helper or another dependency inversion, then run an import smoke test for both modules |
| Over-broad test patch rewrite removed compatibility coverage | Updated tests to patch only the new helper and deleted old wrapper patches | The refactor was supposed to preserve wrapper seams; deleting tests for those seams makes the compatibility promise untested | Keep regression tests for old patch paths plus new helper tests until a separate cleanup intentionally removes the seam |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
- [ ] For git helper extraction: grepped duplicate command bodies and all private wrapper patch targets
- [ ] Return semantics table written: which callers use helper bool for progress gating, which wrappers ignore it
- [ ] Push argv preserved as plain `git push origin <branch>` unless the issue explicitly requires a semantic change
- [ ] `git_utils` import direction checked with an import smoke test; no module-load import cycle introduced
- [ ] Older wrapper seams and direct `ImplementationPhaseRunner` delegation tests still patch the compatibility layer
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1384 Specific Findings (Unverified Planning Guidance)

| Assumption | Status | Reviewer check |
|------------|--------|----------------|
| Wrapper patch seams can be removed once a shared helper exists | HIGH RISK | Grep tests for private-method patch targets; keep thin wrappers unless every compatibility patch path is intentionally migrated |
| Helper `bool` return can flow through all callers | HIGH RISK | Preserve ReviewPhase progress gating while confirming AddressReviewer intentionally ignores the bool or retains a `None` wrapper contract |
| "Force-push" wording means push behavior should change | UNVERIFIED | Read current code and preserve `git push origin <branch>` unless the issue has explicit semantic acceptance criteria |
| `git_utils` can import `pr_manager` normally | HIGH RISK | Avoid module-load imports that reintroduce cycles; lazy import and run `python -c` import smoke tests |
| ImplementationPhaseRunner can bypass its old delegation seam | HIGH RISK | Confirm older tests and the #714 cycle-break patch surface still work with the proposed delegation |

### External Inputs to Re-verify Before Approval

- GitHub issue #1384 text, especially whether "force-push" is requirement text or loose wording.
- Exact ProjectHephaestus file/test line numbers cited in the plan; treat them as stale until reopened.
- GitHub branch protection and required-check behavior if the plan claims a push/PR gate outcome.
- Current `commit_changes` behavior and the exact subprocess failure shape from `subprocess.CalledProcessError`.
- Full duplicate-body inventory for `git commit`, `git push`, `commit_changes`, and wrapper methods.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning review for issue #1384 (extract duplicated git commit/push mechanics into a shared helper while preserving patchable wrapper seams) | v2.2.0 guidance is unverified; use it as a reviewer checklist for stale references, wrapper seams, bool semantics, push argv preservation, and import-cycle risk |
