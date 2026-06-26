---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY module-consolidation plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extending a main() function with new sub-checks, (5) consolidating two functions with the same name but different signatures, (6) extracting a shared CLI parser helper from repeated validation entrypoints while preserving repo-root fallback, --json/--version flags, and parse_known_args pass-through behavior."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
tags: [dry, refactoring, module-consolidation, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, cli, argparse, validation-cli, repo-root, parse-known-args]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture hidden assumptions in DRY validation refactors before implementation starts: module consolidation (#1189) and shared validation CLI parser extraction (#1409). |
| **Outcome** | Plan-stage checklist expanded. v2.2.0 adds issue #1409 risks around stale issue counts, parser subclass behavior, pass-through parsing, and target-population classification. |
| **Verification** | unverified — plan not yet implemented or CI-confirmed |
| **History** | v1.0.0: initial 5-assumption capture. v2.0.0: revised with concrete fix patterns for signature collision and test delegation. v2.1.0: add R2 findings — DOTALL regex crosses TOML sections, wrong test count stated in plan. v2.2.0: add shared validation CLI parser planning risks from issue #1409. |

## When to Use

- Planning to merge two modules into one canonical module (DRY consolidation)
- Replacing an existing module with a delegation shim that re-exports from the canonical
- Porting test classes from one file to another during a refactor
- Adding new public functions to an existing module within a package
- Extending a `main()` function with new sub-checks
- Adding a `from packaging.version import Version` (or any ecosystem dependency) to a new function
- Two modules share a function name with different signatures
- Extracting values from structured config files (TOML, YAML) using regex — always verify section-boundary behavior with a cross-section test case
- Extracting repeated CLI parser setup into a shared helper, especially when the duplicated entrypoints all accept `--repo-root`, `--json`, and `--version`
- The issue names a count of affected validation scripts, but a current `rg` inventory returns a different count
- The planned helper subclasses `argparse.ArgumentParser` or overrides `parse_known_args()` to apply defaults after parsing
- Some candidate entrypoints have pass-through parsing or intentionally different parser contracts

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

# 7. Shared validation CLI parser extraction: enumerate the CURRENT target set.
rg -l '"--repo-root"' hephaestus/validation -g '*.py' | sort

# 8. Classify parser contract differences before migrating.
rg -n 'parse_known_args|parse_args\(argv|argparse\.ArgumentParser' hephaestus/validation hephaestus/cli

# 9. Verify every helper the plan relies on exists and is exported where expected.
rg -n 'def add_json_arg|def add_version_arg|def create_parser|__all__|def get_repo_root' \
  hephaestus/cli hephaestus/utils tests/unit/cli

# 10. Prove the parser subclass behavior with focused tests, not by argparse intuition.
pixi run pytest tests/unit/cli/test_utils.py -q
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

6. **For shared CLI parser extraction, verify the target population and every parser contract.**
   Do not trust the issue's affected-file count. Run the current `rg -l '"--repo-root"'` inventory
   and list the exact migrated files in the plan. If the issue claims 16 validation files but the
   current tree has 12 exact `--repo-root` matches, call that out explicitly and classify the
   exclusions instead of forcing a mechanical migration.

   The shared helper must preserve all parser behaviors, not just the visible flags:
   - explicit `--repo-root` remains a `Path`
   - missing `--repo-root` resolves through `get_repo_root()`
   - `--json` and `--version` stay available through the existing helper functions
   - `parse_known_args()` also resolves `repo_root` for pass-through entrypoints
   - entrypoints with deliberately different contracts, such as mypy-style pass-through flags, are left alone unless their behavior is directly matched

   For an `argparse.ArgumentParser` subclass, test both `parse_args()` and `parse_known_args()`.
   Also have the reviewer check special argparse exits (`--help`, `--version`, parse errors) because
   a post-parse defaulting hook can accidentally execute fallback work on paths that should exit
   before normal namespace use.

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
| Issue #1409 — Trust the issue's validation-file count | Plan inherited the issue premise that 16 validation files needed parser migration, while current discovery found 12 exact `--repo-root` matches | Scope would drift into files whose parser contracts do not match the duplicated pattern, or miss the need to explain why the counts differ | Treat issue counts as stale until `rg` proves the current population; list exact target files and explicitly classify exclusions |
| Issue #1409 — Assume `parse_args()` coverage proves parser subclass safety | Proposed testing explicit repo-root/default/json behavior through normal `parse_args()` only | Some validation CLIs use `parse_known_args()` or `argv`-accepting entrypoints; a helper that works for `parse_args()` can still break pass-through flags or caller-supplied argv | Add focused tests for both `parse_args()` and `parse_known_args()`, including unknown extras preservation |
| Issue #1409 — Treat all validation entrypoints as mechanically identical | Planned to migrate every file with similar visible flags without proving matching behavior for special cases | Files such as mypy pass-through wrappers can rely on unknown-argument preservation; files without `--repo-root` may intentionally use a different root model | Migrate exact matching parser contracts first; leave deliberate variants out unless their contract is separately verified |
| Issue #1409 — Rely on line numbers and helper names without re-reading exports | Plan cited helper/export locations and test barrel behavior from memory and search snippets | Line numbers and export lists drift, and missing barrel exports fail package-level import tests even if module tests pass | Re-open `hephaestus/cli/utils.py`, `hephaestus/cli/__init__.py`, and `tests/unit/cli/test_utils.py`; update `__all__` and run the export parity test |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
```

### Shared Validation CLI Parser Checklist (issue #1409)

```
## Pre-implementation parser extraction audit

- [ ] Ran `rg -l '"--repo-root"' hephaestus/validation -g '*.py'` and copied the exact current target list
- [ ] Explained any mismatch between issue-stated count and current discovery count
- [ ] Classified every non-migrated validation file by parser contract, not by hand-wave
- [ ] Re-read `add_json_arg()`, `add_version_arg()`, `create_parser()`, `get_repo_root()`, and package `__all__`
- [ ] Added tests for explicit `--repo-root`, default `get_repo_root()`, `--json`, `--version`, `parse_args()`, and `parse_known_args()` extras preservation
- [ ] Ran structural grep proving duplicated parser setup disappeared only from the intended migrated set
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
| Issue count says 16 validation files need migration | UNVERIFIED / likely stale | Re-run the current `rg -l '"--repo-root"' hephaestus/validation -g '*.py'` inventory; plan cited 12 exact matches and exclusions must stay explicit |
| `_ValidationArgumentParser.parse_known_args()` fallback is behavior-preserving | UNVERIFIED until tests run | Verify `parse_args()` and `parse_known_args()` both fill `repo_root`, unknown extras survive, and special exits such as `--version` do not trigger unwanted fallback work |
| All migrated CLIs share the same repo-root contract | PARTIAL | Review each target's previous `default=None` + lazy `get_repo_root()` behavior; do not migrate mypy-style pass-through or no-repo-root files without separate proof |
| `create_validation_parser()` can be exported from both utils and package barrel without fallout | UNVERIFIED until export parity test runs | Run `tests/unit/cli/test_utils.py`, especially the export parity assertion, after updating both `__all__` lists |
| Structural grep proves the duplicated setup is gone | NECESSARY BUT NOT SUFFICIENT | Pair grep with focused validation-entrypoint tests; a clean grep can still hide changed `argv`, `prog`, formatter, or epilog behavior |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning phase for issue #1409 (extract shared validation CLI parser helper) | v2.2.0 captures plan-stage risks: issue count drift (16 claimed vs 12 exact current `--repo-root` matches in discovery), parser subclass fallback behavior, pass-through parsing, package barrel exports, and structural grep limitations. Implementation pending. |
