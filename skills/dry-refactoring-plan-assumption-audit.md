---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY/refactoring plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extending a main() function with new sub-checks, (5) consolidating two functions with the same name but different signatures, (6) migrating CLI logging/basicConfig behavior while preserving stdout/stderr contracts, (7) removing C901 suppressions from current code rather than stale issue descriptions."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
history: dry-refactoring-plan-assumption-audit.history
tags: [dry, refactoring, module-consolidation, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, logging, basicconfig, stdout-stderr, c901]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture hidden assumptions that invalidate refactoring plans before implementation, including module consolidation, CLI logging/basicConfig migration, and C901 cleanup |
| **Outcome** | Planning-phase checklist amended with ProjectHephaestus issue #1404 risks; not executed end-to-end |
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
- Planning a CLI logging migration from `logging.basicConfig(...)` to shared logging helpers while preserving JSON stdout cleanliness and existing file-handler behavior
- Removing `# noqa: C901` suppressions by extracting helpers from current code, especially when an issue body cites stale counts, locations, or subcommands
- Writing a plan whose correctness depends on reviewer attention to anti-drift tests, stdout/stderr assertions, repeated handler deduplication, or `ruff --select C901 --ignore-noqa`

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

# 7. For logging/basicConfig migrations, count production callers from disk
rg -n 'logging\.basicConfig\(' hephaestus/

# 8. Protect stdout/stderr contracts before moving CLI logs
rg -n 'emit_json_status|setup_logging|get_logger\(__name__\)' hephaestus/

# 9. For C901 cleanup, measure current violations even under noqa
pixi run ruff check hephaestus/ --select C901 --ignore-noqa
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

6. **For CLI logging migrations, preserve the stream contract before replacing `basicConfig`.**
   Count current production callers with a disk scan, not the issue body. Then trace every JSON-emitting CLI:
   `emit_json_status()` writes JSON to stdout, while historical `setup_logging()` defaults may also write
   logs to stdout. Any CLI that emits machine-readable stdout needs explicit stderr routing for logs before
   shared logging setup is introduced.

7. **Audit module-level logger helpers separately from root logging.**
   Replacing `logging.basicConfig(...)` with root configuration does not migrate modules that call
   `get_logger(__name__)` and install their own stdout `StreamHandler`. In ProjectHephaestus issue #1404,
   `tidy.py` and `fleet_sync.py` had this shape. Plans must test both root logger behavior and module
   helper behavior, including repeated-call handler deduplication.

8. **For C901 cleanup, measure current suppressions and extract from the code on disk.**
   Run `ruff --select C901 --ignore-noqa`, read the current functions, and extract helpers from the
   real workflow. Do not follow stale issue-body descriptions of subcommands, branches, or helper names
   that no longer exist.

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
| Issue #1404 — Trusted stale logging count | Treated the issue body's "11+" `logging.basicConfig(...)` callers as current scope | Disk scan found 9 current production callers under `hephaestus/`; comments in `hephaestus/logging/utils.py` and `scripts/example_usage.py` were non-production or stale relative to the issue scope | Always count production callers from the checked-out tree and classify comments/examples separately from migration scope |
| Issue #1404 — Root-only logging migration | Planned to configure root logging and assume all CLI streams moved with it | `get_logger(__name__)` in `tidy.py` and `fleet_sync.py` installs module stdout handlers; root logging does not fully migrate those stream behaviors | Audit helper-created handlers separately from root logging and add repeated-call dedup tests |
| Issue #1404 — JSON stdout contamination risk | Migrated CLIs toward shared logging without making stream routing explicit | `emit_json_status()` writes JSON to stdout, and `setup_logging()` historically defaulted to stdout; logs on stdout corrupt machine-readable JSON output | For every JSON-emitting CLI, assert JSON remains on stdout and logs go to stderr |
| Issue #1404 — Stale C901 subcommand descriptions | Planned C901 cleanup from issue-body descriptions of `tidy.py` and `pr_merge.py` behavior | The current disk workflow differed; suppressions should be removed by extracting helpers from the code that exists now, not from stale subcommand narratives | Run `ruff --select C901 --ignore-noqa`, read the current functions, and derive helpers from live control flow |
| Issue #1404 — Under-specified compatibility tests | Relied on reviewer attention for date format parity, `implementer_cli` file handler behavior, AST anti-drift tests, stdout/stderr assertions, and handler deduplication | These are subtle regressions that can pass superficial logging or C901 tests | Put each reviewer risk into an explicit test/checklist item before implementation starts |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
- [ ] For logging migrations, counted current production `logging.basicConfig(...)` callers from disk and excluded comments/examples from production scope
- [ ] For JSON CLIs, asserted machine-readable JSON stays on stdout and logs route to stderr
- [ ] Checked both root logging and module-level `get_logger(__name__)` handlers for stream behavior and repeated-call deduplication
- [ ] For C901 cleanup, ran `ruff --select C901 --ignore-noqa` and derived extractions from current disk control flow
- [ ] Listed every unverified external dependency: issue text, real CLI invocation behavior, GitHub API merge/review behavior, and CI/pr-policy behavior
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1404 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| Issue-body `logging.basicConfig(...)` count is current | WRONG | Disk scan found 9 production callers under `hephaestus/`, not 11+ |
| Comment/example references are production scope | WRONG | `hephaestus/logging/utils.py` comments and `scripts/example_usage.py` were non-production or stale for the issue scope |
| `setup_logging()` can keep default stdout routing everywhere | RISKY | JSON-emitting CLIs need explicit stderr routing because `emit_json_status()` writes JSON to stdout |
| Root logging migration covers all handlers | WRONG | `tidy.py` and `fleet_sync.py` use `get_logger(__name__)` module stdout handlers |
| C901 cleanup can follow stale issue subcommand descriptions | WRONG | Extract helpers from current `tidy.py` and `pr_merge.py` workflows on disk |
| Date format parity and `implementer_cli` file handler behavior are obvious | UNVERIFIED | Treat both as compatibility risks requiring direct tests |
| GitHub issue text, real CLI invocation behavior, GitHub API merge/review behavior, and CI/pr-policy behavior are verified | UNVERIFIED | These external dependencies were not directly re-verified during planning |

### Issue #1404 Reviewer Checklist

```text
- AST anti-drift test proves all production basicConfig callers are covered.
- stdout/stderr tests prove JSON stdout remains clean after migration.
- Repeated setup/get_logger calls do not duplicate handlers.
- Date formatting matches prior logging output where compatibility matters.
- implementer_cli file handler behavior is preserved.
- ruff --select C901 --ignore-noqa is run and stale noqa suppressions are removed by extraction, not by threshold changes.
```

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning phase for issue #1404 (logging/basicConfig migration and C901 cleanup) | v2.2.0 captures planning risks only; implementation and CI verification pending |
