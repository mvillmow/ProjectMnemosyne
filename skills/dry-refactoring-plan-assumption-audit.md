---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY refactoring plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extending a main() function with new sub-checks, (5) consolidating two functions with the same name but different signatures, (6) extracting shared argparse/CLI parser helpers while preserving flags, defaults, JSON output, repo-root behavior, and barrel re-exports."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
history: dry-refactoring-plan-assumption-audit.history
tags: [dry, refactoring, module-consolidation, cli-parser, argparse, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, repo-root, json-output, version-flag]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture hidden assumptions that invalidate DRY refactoring plans, including module consolidation (`scripts_lib` -> `validation`, issue #1189) and shared validation/version CLI parser helper extraction (`hephaestus.cli.utils`, issue #1419) |
| **Outcome** | Plan-risk checklist maintained; issue #1419 additions are planning-stage only and must be verified against the current checkout before implementation approval |
| **Verification** | unverified — plan not yet implemented or CI-confirmed |
| **History** | [changelog](./dry-refactoring-plan-assumption-audit.history). v2.2.0 adds validation/version CLI parser helper planning risks from ProjectHephaestus issue #1419. |

## When to Use

- Planning to merge two modules into one canonical module (DRY consolidation)
- Replacing an existing module with a delegation shim that re-exports from the canonical
- Porting test classes from one file to another during a refactor
- Adding new public functions to an existing module within a package
- Extending a `main()` function with new sub-checks
- Adding a `from packaging.version import Version` (or any ecosystem dependency) to a new function
- Two modules share a function name with different signatures
- Extracting values from structured config files (TOML, YAML) using regex — always verify section-boundary behavior with a cross-section test case
- Centralizing duplicated argparse/parser setup across validation or version consistency entry points
- Adding a helper that composes existing parser utilities such as `create_parser()` and `add_version_arg()`
- Migrating many parser sites found by `rg` while some sites intentionally preserve different defaults, root-resolution behavior, or `parse_known_args()` forwarding
- Reviewing a plan that relies on issue title/body/affected-file details or line-number evidence that was not freshly verified against GitHub/current checkout

## Verified Workflow

### Proposed Workflow (UNVERIFIED — planning artifact only)

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

# 7. For CLI parser helper extraction, map every parser construction and behavior-sensitive flag.
rg -n "create_parser|add_version_arg|parse_known_args|ArgumentParser|--repo-root|--json|-V|--version" \
  hephaestus/ tests/

# 8. Verify the live issue scope before trusting title/body/affected-file summaries.
gh issue view 1419 --repo HomericIntelligence/ProjectHephaestus --json title,body,labels

# 9. Prove behavior with tests before claiming a parser refactor preserved CLI contracts.
python -m pytest tests/unit/cli tests/unit/validation -k "parser or cli or json or version or repo_root"
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

6. **Map the parser helper contract before extracting shared CLI boilerplate.**
   If `create_parser()` already registers `-V/--version`, the new helper must not call
   `create_parser()` and then call `add_version_arg()` again. Argparse duplicate option registration
   is a runtime failure, and the bug can be missed if tests only import the module instead of
   constructing the parser or invoking `--help`/`--version`.

7. **Preserve intentional parser variants; do not migrate by blind search/replace.**
   A validation/version parser refactor should first classify every call site into "standard helper"
   versus "intentional variant." Examples from the issue #1419 plan:
   - `mypy_per_file.py` must keep `parse_known_args()` forwarding for mypy flags.
   - `markdown.py` README validation must keep the `Path.cwd()` default scan behavior and omit `--repo-root`.
   - `repo_analyze_skills.py` uses repository-relative constants; use a shared helper only for JSON/version
     boilerplate, not root resolution.
   - `audit.py` should derive the default ignore file from `resolve_repo_root(args)` only when
     `args.ignore_file` is absent.
   - `coverage.py` should switch to `repo_root / "coverage.toml"` only when `args.repo_root` is
     explicitly supplied, preserving the previous fallback otherwise.

8. **Treat issue summaries and line numbers as stale until verified.**
   If the issue title mentions one script but the body/proposed solution/affected files point at a
   parser refactor, verify `gh issue view` before approving scope. Likewise, replace cited line
   numbers with fresh `rg`/`sed` evidence from the checkout that will be edited.

9. **Test behavior, not just importability.**
   For parser helper refactors, tests must assert the observable CLI contract: preserved arguments and
   defaults, JSON output shape, `-V/--version` behavior, `__all__`/barrel re-export coverage, and any
   `parse_known_args()` pass-through behavior.

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
| CLI parser helper double-registers version flags | A new helper would call `create_parser()` and then also call `add_version_arg()` because both looked like reusable boilerplate | If `create_parser()` already owns `-V/--version`, argparse rejects duplicate option strings or the behavior diverges by parser site | Before extracting the helper, read the parser utility contract and add a parser-construction or `--version` test that would fail on duplicate registration |
| Blindly standardize every parser site found by `rg` | Search/replace migrated adjacent validation/version scripts without classifying intentional variants | `parse_known_args()` forwarding, README default scan behavior, repository-relative constants, and conditional config defaults can all regress even when the new helper compiles | Build a site-by-site migration table and mark intentional variants before editing |
| Normalize repo-root resolution globally | Applied `resolve_repo_root(args)` wherever `--repo-root` or config paths appeared | Some tools intentionally default to `Path.cwd()`, some only use repo root when a flag is explicit, and some should not resolve root at all | Preserve caller-specific default semantics; add tests for absent-flag behavior as well as explicit-flag behavior |
| Trust unverified issue title/body and stale line numbers | Planned scope from a contextual issue summary and cited line numbers without re-opening GitHub issue #1419 or the current files | The issue title reportedly mentioned `ensure_state_labels.py` while the body/proposed solution pointed to parser boilerplate; line numbers also drift | Verify live issue scope with `gh issue view` and re-derive file evidence from the checkout before approving implementation scope |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
- [ ] For parser helper extraction, read `create_parser()` and `add_version_arg()` before composing them — no duplicate `-V/--version` registration?
- [ ] Re-export contract checked: new helpers appear in module `__all__` and package-level barrel imports expected by tests?
- [ ] Every parser site found by `rg` classified as standard-helper or intentional-variant before editing?
- [ ] Absent-flag defaults covered by tests, especially repo-root/config defaults that differ by entry point?
- [ ] Live issue scope and current line evidence verified, not copied from an old plan or issue snapshot?
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1419 CLI Parser Consolidation Planning Risks

These findings are **unverified** and come from a planning checklist for centralizing validation/version
CLI parser boilerplate in `hephaestus.cli.utils`.

| Assumption or dependency | Status | Reviewer focus |
|--------------------------|--------|----------------|
| `create_parser()` already registers `-V/--version` | UNVERIFIED until read in checkout | Ensure `create_validation_parser()` does not call both `create_parser()` and `add_version_arg()` |
| `resolve_repo_root(args)` should delegate to `hephaestus.utils.helpers.get_repo_root` | UNVERIFIED until source/test checked | Verify helper location/import cycle risk and preserve caller behavior |
| `tests/unit/cli/test_utils.py` enforces every `utils.__all__` symbol is re-exported from `hephaestus.cli` | UNVERIFIED until current test read | Add new helpers to `__all__` and package barrel exports together |
| All issue-listed plus adjacent parser sites should migrate | CONDITIONAL | Use `rg` to find candidates, but preserve intentional variants instead of bulk replacing |
| `mypy_per_file.py` uses `parse_known_args()` for mypy flag forwarding | UNVERIFIED until current parser read | Keep pass-through behavior; add a regression test if helper extraction touches it |
| `markdown.py` README validation defaults to `Path.cwd()` and omits `--repo-root` | UNVERIFIED until current parser read | Do not standardize away the current default scan behavior |
| `repo_analyze_skills.py` uses repository-relative constants | UNVERIFIED until current code read | Use helper only for JSON/version boilerplate, not root resolution |
| `audit.py` default ignore file depends on repo root only when absent | UNVERIFIED until behavior test added | Preserve explicit `--ignore-file`; only derive default when missing |
| `coverage.py` config default falls back unless `--repo-root` is explicit | UNVERIFIED until tests cover both paths | Test absent and explicit `--repo-root` separately |
| Issue title/body mismatch around `ensure_state_labels.py` | UNVERIFIED external source | Open GitHub issue #1419 before approving scope |
| Plan line-number evidence | STALE BY DEFAULT | Re-derive line numbers with `rg`/`sed` from the implementation checkout |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning phase for issue #1419 (centralize validation/version CLI parser boilerplate via `create_validation_parser()` and `resolve_repo_root()`) | Unverified planning checklist only: implementation not executed, tests not run, GitHub issue scope not freshly verified |
