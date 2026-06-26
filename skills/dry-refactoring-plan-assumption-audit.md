---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY module-consolidation and shared CLI-parser refactor plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing repeated argparse setup with a shared parser factory, (3) replacing a module with a delegation shim that re-exports from the canonical, (4) porting tests from one file to another, (5) extending a main() function with new sub-checks, (6) consolidating two functions with the same name but different signatures."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
history: dry-refactoring-plan-assumption-audit.history
tags: [dry, refactoring, module-consolidation, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, argparse, validation-cli, parser-factory, parse-known-args, repo-root, inventory-scope]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture hidden assumptions that invalidate DRY refactor plans before implementation, including module consolidation and shared validation CLI parser extraction plans. |
| **Outcome** | Plan-risk checklist extended with issue #1409 parser-factory risks: private `ArgumentParser` subclass behavior, `parse_known_args` forwarding, export/barrel checks, inventory drift, and non-running CLI paths. |
| **Verification** | unverified — plan not yet implemented or CI-confirmed |
| **History** | [changelog](./dry-refactoring-plan-assumption-audit.history). v1.0.0: initial 5-assumption capture. v2.0.0: revised with concrete fix patterns for signature collision and test delegation. v2.1.0: add R2 findings — DOTALL regex crosses TOML sections, wrong test count stated in plan. v2.2.0: add ProjectHephaestus issue #1409 shared validation CLI parser planning-risk capture. |

## When to Use

- Planning to merge two modules into one canonical module (DRY consolidation)
- Replacing an existing module with a delegation shim that re-exports from the canonical
- Porting test classes from one file to another during a refactor
- Adding new public functions to an existing module within a package
- Extending a `main()` function with new sub-checks
- Adding a `from packaging.version import Version` (or any ecosystem dependency) to a new function
- Two modules share a function name with different signatures
- Extracting values from structured config files (TOML, YAML) using regex — always verify section-boundary behavior with a cross-section test case
- Centralizing repeated argparse setup into a shared helper, especially when some callers use `parse_args()` and others use `parse_known_args()` to forward unknown flags
- Reviewing a plan whose CLI inventory, line numbers, or issue scope came from previous grep output rather than a fresh current-head verification
- A plan says fallback repo-root resolution must not run for `--help`, `--version`, or parse-error paths

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

# 7. For shared argparse helpers, inventory every caller and parser mode at current HEAD
rg -n "ArgumentParser|parse_args|parse_known_args|--json|--version|--repo-root" hephaestus tests scripts

# 8. Prove non-running paths do not resolve fallback repo roots
python -m <validation_cli> --help
python -m <validation_cli> --version
python -m <validation_cli> --unknown-flag
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

6. **For shared argparse factories, preserve each caller's parser contract explicitly.**
   A helper such as `hephaestus.cli.utils.create_validation_parser()` can remove duplicated
   `--repo-root`, `--json`, and `--version` setup, but it must not force every caller through the same
   parsing API. If one validation CLI intentionally calls `parse_known_args()` to forward unknown mypy
   flags, the plan must preserve that consumer's mode and test it with representative unknown flags.
   Treat a private `argparse.ArgumentParser` subclass that overrides `parse_args()` as an implementation
   choice to verify, not a free KISS win: reviewers should ask whether it affects `parse_known_args()`,
   help/version exits, error formatting, and type-checking expectations.

7. **Re-run the inventory that defines scope at implementation time.**
   A plan that says "all issue-named validation CLIs plus `repo_analyze_skills.py` are in scope" because
   of a prior `rg` is a snapshot, not evidence. Re-run the inventory on current HEAD before editing and
   include edge cases in the checklist: modules with two entry points, scripts outside the validation
   package, and tests that assert exports via a barrel module such as `tests/unit/cli/test_utils.py`.

8. **Exercise non-running CLI paths before approving fallback-resolution changes.**
   When a shared parser resolves a fallback `get_repo_root()`, the highest-risk regression is doing that
   work too early. `--help`, `--version`, and parse-error paths should exit inside argparse before any
   filesystem-dependent fallback runs. Put those paths in tests, not just in review prose.

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
| Issue #1409 — Treat shared parser subclass as obviously safe | Planned a private `argparse.ArgumentParser` subclass overriding `parse_args()` to centralize `--repo-root` fallback behavior | The plan did not execute argparse edge paths in this learn step; subclass behavior around `parse_known_args()`, `--help`, `--version`, and parse errors remains an assumption | Keep the subclass if it stays small, but require tests that prove both parser modes and non-running paths preserve existing behavior |
| Issue #1409 — Trust prior `rg` inventory as final scope | Plan scope came from current/prior grep observations: all issue-named validation CLIs plus `repo_analyze_skills.py`; `markdown.py` has two CLI entrypoints | File contents and issue scope can drift before implementation, and a missed CLI leaves duplicated parser setup or inconsistent flags | Re-run scope inventory on current HEAD and make the reviewer check omitted validation modules, scripts, and dual-entrypoint files |
| Issue #1409 — Structural tests only assert strings | Planned tests may prove `create_validation_parser` or option strings appear in code without proving runtime semantics | String usage tests can pass while `parse_known_args()` stops forwarding flags, duplicate `--json`/`--version` registration raises, or fallback repo-root resolution runs too early | Add runtime tests for parser invocation modes, duplicate flag registration, fallback suppression on exits/errors, and unknown-flag forwarding |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
- [ ] If extracting argparse setup, listed every CLI caller and whether it uses `parse_args()` or `parse_known_args()`
- [ ] Confirmed tests cover runtime parser behavior, not only import/string usage
- [ ] Proved `--help`, `--version`, and parse-error paths do not call fallback repo-root resolution
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1409 Specific Findings

| Assumption | Status | Reviewer Focus |
|------------|--------|----------------|
| A private `argparse.ArgumentParser` subclass with a `parse_args()` override is acceptable KISS | UNVERIFIED | Confirm it does not affect `parse_known_args()` consumers, error formatting, help/version exits, or type-checking expectations |
| `parse_known_args()` for `mypy_per_file` keeps forwarding unknown mypy flags | UNVERIFIED | Add a runtime test that passes unknown mypy flags and asserts they remain in the forwarded remainder |
| `create_validation_parser` must be exported because `tests/unit/cli/test_utils.py` has a barrel export check | UNVERIFIED IN THIS LEARN STEP | Re-read the test at implementation time and update both the defining module and package/barrel export if required |
| All issue-named validation CLIs plus `repo_analyze_skills.py` are in scope based on a current-head `rg` inventory | SNAPSHOT ONLY | Re-run `rg` on current HEAD; look for scope omissions among validation modules, scripts, and duplicate entrypoints |
| `markdown.py` has two CLI entrypoints | SNAPSHOT ONLY | Preserve both entrypoints or prove one is intentionally out of scope before deleting duplicate parser setup |
| Fallback `get_repo_root()` must not run for `--help`, `--version`, or parse-error paths | HIGH RISK | Test those paths by monkeypatching/spy-wrapping fallback resolution and asserting it is not called |
| Structural tests prove the refactor | WEAK | Prefer runtime semantics tests for duplicate/missing `--json`/`--version`, unknown-flag forwarding, and non-running paths |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning phase for issue #1409 (centralize validation CLI parser setup in `hephaestus.cli.utils.create_validation_parser()`) | Plan produced, NOT executed in this learn step; line numbers, inventory, argparse external behavior, exact issue wording, and current file contents remain unverified before implementation. |
