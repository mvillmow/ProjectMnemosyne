---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY and shared-helper migration plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extending a main() function with new sub-checks, (5) consolidating two functions with the same name but different signatures, (6) replacing a bare subprocess or external CLI call with a canonical helper while preserving failure fallback behavior."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
history: dry-refactoring-plan-assumption-audit.history
tags: [dry, refactoring, module-consolidation, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, subprocess, shared-helper, mock-target, gh-cli, fallback]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture hidden assumptions in DRY/refactor plans before implementation, including module consolidation and single-call migration from a bare subprocess/external CLI probe to a canonical shared helper |
| **Outcome** | Plan produced; NOT executed. v2.2.0 adds ProjectHephaestus issue #1411 planning risks for replacing `subprocess.run(["gh", "api", "rate_limit"])` with `run_subprocess(...)` while preserving fallback behavior. |
| **Verification** | unverified — plan not yet implemented or CI-confirmed |
| **History** | v1.0.0: initial 5-assumption capture. v2.0.0: revised with concrete fix patterns for signature collision and test delegation. v2.1.0: add R2 findings — DOTALL regex crosses TOML sections, wrong test count stated in plan. v2.2.0: add shared subprocess-helper migration assumptions from ProjectHephaestus issue #1411. |

## When to Use

- Planning to merge two modules into one canonical module (DRY consolidation)
- Replacing an existing module with a delegation shim that re-exports from the canonical
- Porting test classes from one file to another during a refactor
- Adding new public functions to an existing module within a package
- Extending a `main()` function with new sub-checks
- Adding a `from packaging.version import Version` (or any ecosystem dependency) to a new function
- Two modules share a function name with different signatures
- Extracting values from structured config files (TOML, YAML) using regex — always verify section-boundary behavior with a cross-section test case
- Replacing a one-off `subprocess.run(...)` or external CLI probe with a shared helper such as `run_subprocess(...)`
- Updating tests after a helper import changes the mock seam from `subprocess.run` to `module_under_test.run_subprocess`
- Preserving "probe failed, fall back quietly" behavior while adding helper-level timeout/error handling
- A plan relies on helper defaults (`capture_output`, `text`, `stdin`, env propagation, exception types) by reading source but has not executed the call path

## Verified Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms.
> The actionable guidance below is a **Proposed Workflow** for `verification: unverified`
> planning artifacts; the top-level heading remains for the repository validator.

### Proposed Workflow (UNVERIFIED)

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

# 7. Shared subprocess-helper migration audit
rg -n "subprocess\.run|run_subprocess|gh api rate_limit" hephaestus/ tests/
rg -n "patch\\(\"subprocess\\.run\"|patch\\(\".*run_subprocess" tests/
rg -n "except .*CalledProcessError|TimeoutExpired|OSError|JSONDecodeError|ValueError" hephaestus/<module>.py
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

6. **For shared subprocess-helper migrations, prove the helper contract and the fallback contract separately.**
   Replacing a bare probe like `subprocess.run(["gh", "api", "rate_limit"], ...)` with a canonical
   helper is only behavior-preserving if both contracts survive:
   - The helper receives the old behavior-bearing kwargs explicitly (`check=True`, the same timeout,
     and `log_on_error=False` for expected probe failures).
   - The caller still catches the helper's failure modes and returns the same fallback value instead
     of surfacing helper-level logging or exceptions.

   Do not rely on "the helper defaults are equivalent" unless the plan either executes the path or
   explicitly marks the source-read facts as unverified. Read the helper implementation for defaults
   such as `capture_output=True`, `text=True`, `stdin=subprocess.DEVNULL`, correlation-id environment
   propagation, and timeout logging, then hand the reviewer the exact assumptions that were not run.

7. **Patch the imported helper seam, not the old global function.**
   Once the module under test imports `run_subprocess`, tests must patch
   `module_under_test.run_subprocess`. A leftover `patch("subprocess.run")` is a false green: it no
   longer intercepts the call and may leave the real CLI probe reachable. Add an assertion on the
   exact helper call, including `timeout=<old value>` and `log_on_error=False`, so the test protects
   the behavior that motivated the migration.

8. **Treat external CLI response shape as unverified unless the API was exercised.**
   For a `gh api rate_limit` probe, reading code/tests is not the same as proving the live CLI/API
   still returns the expected JSON path (for example `resources.graphql.reset`). If the planning
   session did not run `gh api rate_limit`, mark the JSON-shape and authentication/environment
   assumptions as reviewer risks and keep the fallback test explicit.

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
| Treat helper defaults as verified behavior | Plan relied on `run_subprocess` source-read defaults (`capture_output=True`, `text=True`, `stdin=subprocess.DEVNULL`, env propagation) without executing the `gh api rate_limit` path | Reading the helper confirms intent but not runtime equivalence; a wrapper can still differ in exception timing, logging, environment, or return object behavior | Mark helper-default facts as read-not-run, assert only the behavior-bearing kwargs in tests, and run the targeted probe tests before claiming preservation |
| Leave tests patched at `subprocess.run` after importing a helper | Existing tests patch the old global function while implementation now calls `rate_limit.run_subprocess` | The patch no longer intercepts the call, so tests can false-green or accidentally reach the real CLI | Patch the name as imported by the consumer module: `patch("hephaestus.github.rate_limit.run_subprocess")` |
| Assume external `gh api rate_limit` shape from tests | Plan used a JSON fixture with `resources.graphql.reset` but did not run the real `gh` command/API | The live CLI/auth environment and JSON shape remain unverified; a shape drift should fall back quietly rather than break callers | Keep JSON/ValueError fallback coverage and make the reviewer check whether live `gh api rate_limit` still matches the fixture before relying on it |
| Preserve fallback behavior only by happy-path assertion | Added an assert for the helper happy-path call but did not re-check failure cases | The migration's highest-risk behavior is the expected-failure path: helper exceptions must still be swallowed locally with debug-level fallback | Exercise `CalledProcessError` and invalid JSON/value cases through the new helper seam, and keep `log_on_error=False` for expected handled failures |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
- [ ] Shared helper migration: old bare call removed, imported helper seam patched in tests, behavior-bearing kwargs asserted
- [ ] Fallback path preserved: helper-raised failure, invalid JSON, and missing value still return the old fallback result
- [ ] External CLI/API assumptions separated: live command shape/auth not claimed unless actually run
- [ ] Grep guard scoped correctly: proves the targeted bare call is gone without claiming unrelated subprocess policy
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1411 Specific Findings

| Assumption | Status | Reviewer focus |
|------------|--------|----------------|
| `run_subprocess(["gh", "api", "rate_limit"], check=True, timeout=10, log_on_error=False)` is behavior-equivalent to the old bare probe | UNVERIFIED | Read helper defaults were cited, but the path was not executed. Confirm timeout, captured stdout, exception type, and quiet expected-failure semantics before approving. |
| Patching `hephaestus.github.rate_limit.run_subprocess` is the right test seam | LIKELY, SOURCE-READ ONLY | Once imported into `rate_limit.py`, patch the consumer binding, not `subprocess.run`; verify no old patch target remains in `TestGhRateLimitResetEpoch`. |
| `gh api rate_limit` live JSON has `resources.graphql.reset` | UNVERIFIED EXTERNAL | The plan used a fixture and code reading, not a live GitHub CLI/API call. Treat API shape/auth as a reviewer check, and ensure malformed/missing values still return `None`. |
| Keeping `timeout=10` preserves bounded probe behavior | SOURCE-READ ONLY | Assert the helper call includes `timeout=10`; do not rely on helper defaults for timing. |
| `log_on_error=False` preserves expected-probe-failure noise level | SOURCE-READ ONLY | Confirm failure is still locally handled at debug level and helper logging does not add unexpected operator noise. |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning phase for issue #1411 (replace `github/rate_limit.py` bare `subprocess.run` GitHub CLI probe with `run_subprocess`) | v2.2.0 planning capture only; implementation not executed, no pytest/ruff/grep guard run in the planning session |
