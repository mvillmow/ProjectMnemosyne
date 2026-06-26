---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY module-consolidation plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extending a main() function with new sub-checks, (5) consolidating two functions with the same name but different signatures, (6) consolidating repeated argparse/CLI parser boilerplate across validation modules while preserving repo-root resolution, help/version behavior, explicit argv callers, and unknown-flag passthrough."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
history: dry-refactoring-plan-assumption-audit.history
tags: [dry, refactoring, module-consolidation, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, validation-cli, argparse, repo-root]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture the hidden assumptions that invalidate DRY refactor plans before implementation, including module consolidation (#1189) and validation CLI parser consolidation (#1410). |
| **Outcome** | Plan artifacts only; issue #1189 found concrete NOGO risks, and issue #1410 produced a scoped unverified plan with reviewer risks around argparse semantics, repo-root resolution, and issue title/body mismatch. |
| **Verification** | unverified — plan not yet implemented or CI-confirmed |
| **History** | [changelog](./dry-refactoring-plan-assumption-audit.history) — v1.0.0: initial 5-assumption capture. v2.0.0: revised with concrete fix patterns for signature collision and test delegation. v2.1.0: add R2 findings — DOTALL regex crosses TOML sections, wrong test count stated in plan. v2.2.0: add validation CLI parser consolidation planning risks from ProjectHephaestus issue #1410. |

## When to Use

- Planning to merge two modules into one canonical module (DRY consolidation)
- Replacing an existing module with a delegation shim that re-exports from the canonical
- Porting test classes from one file to another during a refactor
- Adding new public functions to an existing module within a package
- Extending a `main()` function with new sub-checks
- Adding a `from packaging.version import Version` (or any ecosystem dependency) to a new function
- Two modules share a function name with different signatures
- Extracting values from structured config files (TOML, YAML) using regex — always verify section-boundary behavior with a cross-section test case
- Consolidating repeated `argparse.ArgumentParser` setup across many CLI modules
- Planning a validation CLI helper that resolves `args.repo_root` centrally
- Reviewing a plan where the issue title and concrete body acceptance criteria disagree
- Leaving an issue-listed-adjacent module out of a refactor because it was not in the accepted scope
- Preserving `parse_args()`, `parse_known_args()`, explicit `argv`, `--help`, `--version`, JSON output, exit-code, and unknown-flag passthrough behavior while deleting parser boilerplate

## Proposed Workflow

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

# 7. For validation CLI parser consolidation, inventory parser construction and parse styles
rg -n "ArgumentParser|parse_args|parse_known_args|--version|repo_root|argv" hephaestus/validation tests/unit

# 8. Verify the canonical repo-root resolver before planning to centralize it
rg -n "def get_repo_root|get_repo_root\\(" hephaestus tests scripts

# 9. Compare the issue body scope to actual modules before adding or excluding files
find hephaestus/validation -maxdepth 1 -name "*.py" -print | sort

# 10. Protect unknown flag passthrough for mypy/per-file style CLIs
rg -n "parse_known_args|mypy" hephaestus/validation tests/unit
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

6. **Scope to the issue body when title and body disagree, but flag the mismatch as a reviewer risk.**
   For ProjectHephaestus issue #1410, the title mentioned atomic writes, while the concrete acceptance
   criteria described validation CLI parser consolidation. The plan deliberately scoped to the body.
   That is reasonable only if the plan explicitly calls out the mismatch so the reviewer can confirm
   the issue was not misread.

7. **Mark inherited line references as stale unless re-read in the current planning turn.**
   If the plan relies on prior observations such as `hephaestus/cli/utils.py`,
   `hephaestus/cli/__init__.py`, `tests/unit/cli/test_utils.py`, and many
   `hephaestus/validation/*.py` modules, state whether those references were re-verified.
   For #1410 they were not re-verified during the learn capture, so implementation must re-open them.

8. **Treat an `ArgumentParser` subclass that mutates/resolves parsed args as the highest-risk design assumption.**
   A subclass that resolves `args.repo_root` inside `parse_known_args()` may look DRY but can subtly
   change semantics. Reviewers must check `parse_args()`, `parse_known_args()`, `--help`, `--version`,
   explicit `argv` callers, and unknown-flag passthrough.

9. **Verify the repo-root resolver is canonical before centralizing on it.**
   The plan assumed `hephaestus.utils.helpers.get_repo_root()` is the single source of truth.
   Before implementation, re-check current imports and call sites rather than relying on the name.

10. **Ratify the module boundary from the issue list, not from a directory-wide sweep.**
    The #1410 plan assumed all 16 issue-listed `hephaestus/validation/*.py` modules should use the
    shared helper, while `repo_analyze_skills.py` should remain out because it was not in the issue
    list and has different repo-root behavior. Verify this boundary against the current issue before editing.

11. **Review special parser flows one by one before mechanical replacement.**
    Parser consolidation is not a pure search/replace when modules use different parse entrypoints.
    Focused review targets: `mypy_per_file.py` uses `parse_known_args()`, `skill_catalog.py` and
    `cli_tier_docs.py` parse explicit `argv`, `tier_labels.py` has exit-2 root error handling, and
    `markdown.py` has two parser constructions.

12. **Behavior-preservation tests are part of the plan, not an afterthought.**
    Plan direct helper tests, a regression test that prevents parser-boilerplate reintroduction in
    the issue-listed modules, focused validation unit tests for special flows, CLI entry-point
    help/json/version tests, and Ruff check/format checks. The structural grep is necessary but not
    sufficient; the behavior tests are the acceptance gate.

## Verified Workflow

_Not applicable._ This skill revision captures planning-only learnings at `unverified` level. The actionable methodology is in **Proposed Workflow** above and must be treated as unvalidated until the target implementation and CI confirm it. This placeholder exists because `scripts/validate_plugins.py` requires the literal `## Verified Workflow` heading; it intentionally makes no verification claim.

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
| Issue title/body mismatch | Planned #1410 against the issue body's validation CLI parser consolidation criteria even though the title mentioned atomic writes. | The title could indicate stale or split intent; a reviewer might reject the plan as solving the wrong problem. | Scope to the concrete acceptance criteria, but explicitly flag the title/body mismatch and ask reviewers to ratify the scope. |
| Inherited file observations not re-read | Relied on prior planning context for `hephaestus/cli/utils.py`, `hephaestus/cli/__init__.py`, `tests/unit/cli/test_utils.py`, and 16 validation modules. | Line references can drift between planning and implementation. | Mark those facts as unverified in the plan and make re-opening current files a pre-implementation step. |
| `ArgumentParser` subclass treated as behavior-neutral | Proposed centralizing repo-root resolution by subclassing `argparse.ArgumentParser` and resolving `args.repo_root` in `parse_known_args()`. | `parse_args()`, `parse_known_args()`, `--help`, `--version`, explicit `argv`, and unknown passthrough may differ from the old per-module parser code. | Treat parser subclassing as the highest-risk assumption; write behavior tests before replacing boilerplate. |
| Directory-wide validation sweep | Considered all nearby validation modules equally eligible for helper migration. | `repo_analyze_skills.py` was not in the issue list and reportedly has different repo-root behavior. | Ratify the issue-listed module set and document intentionally excluded adjacent files. |
| Special parser flows hidden in the majority case | Planned one helper for most validation CLIs. | Modules using `parse_known_args()`, explicit `argv`, custom exit-2 root errors, or multiple parser constructions can regress under a uniform helper. | Audit and test each special flow before mechanical replacement. |

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

### Validation CLI Parser Consolidation Planning Checklist

```
## Pre-implementation validation CLI parser audit

- [ ] Issue title/body mismatch acknowledged; scope ratified against the body acceptance criteria.
- [ ] Current files re-opened before implementation: hephaestus/cli/utils.py, hephaestus/cli/__init__.py, tests/unit/cli/test_utils.py, and the issue-listed validation modules.
- [ ] Canonical resolver verified: hephaestus.utils.helpers.get_repo_root() is still the intended single source of truth.
- [ ] Parser helper behavior tested for parse_args(), parse_known_args(), explicit argv, --help, --version, JSON mode, root errors, and unknown mypy flag passthrough.
- [ ] The 16 issue-listed validation modules migrate to the shared helper; repo_analyze_skills.py remains excluded unless the issue/reviewer ratifies inclusion.
- [ ] Special flows reviewed: mypy_per_file.py, skill_catalog.py, cli_tier_docs.py, tier_labels.py, markdown.py.
- [ ] Structural regression prevents reintroducing parser boilerplate in the issue-listed modules.
- [ ] Verification plan includes focused unit tests, CLI entry-point help/json/version tests, ruff check, and ruff format.
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1410 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| Issue title and body described the same work | RISK | Title mentioned atomic writes; body acceptance criteria appeared to be validation CLI parser consolidation. Scope to body, but flag mismatch. |
| Prior line references were current | UNVERIFIED | The learn capture did not re-open the ProjectHephaestus files; implementation must re-verify current lines. |
| `ArgumentParser` subclass preserves all behavior | HIGHEST RISK | Must prove `parse_args()`, `parse_known_args()`, `--help`, `--version`, explicit `argv`, and unknown mypy flag passthrough. |
| `get_repo_root()` is canonical | UNVERIFIED | Re-check `hephaestus.utils.helpers.get_repo_root()` and current call sites before centralizing. |
| All validation modules in the directory should migrate | PARTIAL | Migrate the 16 issue-listed modules; leave `repo_analyze_skills.py` out unless scope is ratified. |
| Special parser flows can use the common helper unchanged | RISK | Review `mypy_per_file.py`, `skill_catalog.py`, `cli_tier_docs.py`, `tier_labels.py`, and `markdown.py` individually. |

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning phase for issue #1410 (validation CLI parser consolidation) | v2.2.0 planning-only capture; no implementation or tests executed during learn invocation; prior file observations were not re-verified in this capture. |
