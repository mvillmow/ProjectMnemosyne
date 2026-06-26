---
name: dry-refactoring-plan-assumption-audit
description: "Checklist of hidden assumptions that bite DRY module-consolidation and helper-centralization plans before implementation starts. Use when: (1) planning to merge two modules into one canonical, (2) replacing a module with a delegation shim that re-exports from the canonical, (3) porting tests from one file to another, (4) extending a main() function with new sub-checks, (5) consolidating two functions with the same name but different signatures, (6) centralizing repeated optional repo-root fallbacks without changing marker-walk semantics, (7) moving mechanics into a shared helper while preserving a public compatibility wrapper."
category: architecture
date: 2026-06-26
version: "2.2.0"
user-invocable: false
verification: unverified
history: dry-refactoring-plan-assumption-audit.history
tags: [dry, refactoring, module-consolidation, helper-centralization, planning, assumptions, shim, __all__, packaging, test-delegation, signature-collision, repo-root-fallback, marker-walk, compatibility-wrapper, git-utils]
---

# DRY Refactoring — Plan Assumption Audit

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture the hidden assumptions that invalidate DRY refactoring plans before implementation, including module consolidation (issue #1189) and helper centralization that must preserve path-resolution and compatibility-wrapper contracts (issue #1413) |
| **Outcome** | Plan produced; issue #1413 additions are reviewer risks only, because the evidence came from repo text searches/file references rather than an implemented diff or CI run |
| **Verification** | unverified — plan not yet implemented or CI-confirmed |
| **History** | [changelog](./dry-refactoring-plan-assumption-audit.history). v2.2.0: add issue #1413 optional repo-root fallback + branch-wrapper planning risks. v2.1.0: add R2 findings — DOTALL regex crosses TOML sections, wrong test count stated in plan. |

## When to Use

- Planning to merge two modules into one canonical module (DRY consolidation)
- Replacing an existing module with a delegation shim that re-exports from the canonical
- Porting test classes from one file to another during a refactor
- Adding new public functions to an existing module within a package
- Extending a `main()` function with new sub-checks
- Adding a `from packaging.version import Version` (or any ecosystem dependency) to a new function
- Two modules share a function name with different signatures
- Extracting values from structured config files (TOML, YAML) using regex — always verify section-boundary behavior with a cross-section test case
- Centralizing repeated optional CLI fallbacks such as `args.repo_root or get_repo_root()` into a small helper
- Adding a wrapper like `resolve_repo_root(repo_root=None)` where explicit-argument behavior must remain distinct from auto-detection
- Moving local mechanics into a shared helper while preserving a public import/patch seam such as `hephaestus.github.pr_merge.local_branch_exists`
- Reviewing a plan whose evidence is `rg` output and file references, not an implemented diff, focused test run, or CI result

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

# 7. For optional repo-root fallback centralization, enumerate ALL fallback spellings.
#    Do not trust only the exact `args.repo_root or get_repo_root()` form.
rg -n "args\\.repo_root|get_repo_root\\(\\)|repo_root.*get_repo_root" hephaestus tests -g "*.py"

# 8. Read the canonical resolver and its tests before adding a wrapper.
#    A helper may centralize caller fallback without changing marker-walk semantics.
rg -n "def get_repo_root|nested|marker|pyproject|\\.git" hephaestus/utils/helpers.py tests/unit/utils/test_general_utils.py

# 9. Before moving branch-existence mechanics, find exported/imported seams.
rg -n "local_branch_exists|from hephaestus\\.github import|from hephaestus\\.github\\.pr_merge import" hephaestus tests scripts -g "*.py"

# 10. Keep PR-specific dry-run helpers out of generic git utilities unless the scope explicitly includes them.
rg -n "def run_git_cmd|dry_run|run_subprocess|METADATA_TIMEOUT" hephaestus/github/pr_merge.py hephaestus/automation/git_utils.py

# 11. After implementation, prove duplicate fallback patterns are gone and compatibility seams remain.
test -z "$(rg -n 'args\\.repo_root\\s+or\\s+get_repo_root\\(\\)|args\\.repo_root\\s+if\\s+args\\.repo_root\\s+is\\s+not\\s+None\\s+else\\s+get_repo_root\\(\\)|if args\\.repo_root is not None' hephaestus -g '*.py')"
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

6. **For optional repo-root fallback centralization, separate explicit-root semantics from auto-detection.**
   A helper such as `resolve_repo_root(repo_root=None)` should make the two code paths visible:
   explicit roots return `Path(repo_root)`; only missing roots call `get_repo_root()`.
   Do not add `.resolve()`, `expanduser()`, marker walks, or existence checks to the explicit path unless existing callers
   already did that. Those are behavior changes, not DRY cleanup.

7. **Do not rewrite the canonical resolver while centralizing caller fallback.**
   If the plan says `get_repo_root()` marker-walk semantics are out of scope, the implementation and tests must prove
   that. Read the resolver and nested-marker tests before planning edits. Reviewer focus: the new helper delegates to
   `get_repo_root()` only for `None`, and no existing first-match-up behavior changes.

8. **Audit fallback call sites by behavior, not by a single regex.**
   Repeated optional-root patterns appear as `args.repo_root or get_repo_root()`, ternaries, explicit
   `if args.repo_root is not None` blocks, local imports, and constructors that may or may not share the same contract.
   The plan may correctly exclude object constructors, but that exclusion must be justified per call site rather than
   hidden behind a broad "16+ occurrences" claim.

9. **When moving mechanics to a shared helper, preserve public compatibility wrappers.**
   If `hephaestus.github.pr_merge.local_branch_exists` is exported from `hephaestus/github/__init__.py` or patched by
   tests, do not delete it just because the implementation moved to `automation.git_utils`. Keep a thin wrapper and
   add delegation tests at the public seam. The shared helper's subprocess options (`cwd`, timeout, `check=False`,
   `log_errors=False`, exception tuple) are part of the behavior under review.

10. **Keep PR-specific helpers local unless the plan explicitly broadens scope.**
    A `run_git_cmd(..., dry_run=True)` wrapper that logs PR-merge commands is not automatically the same abstraction as a
    generic `automation.git_utils.run()`. Moving it without dry-run parity is scope creep; adding dry-run support to the
    generic helper may be a separate design change.

11. **Label grep-and-file-reference plans as unverified until the implementation and tests run.**
    Repo text searches are useful planning evidence, but they do not prove import compatibility, subprocess behavior,
    duplicate-pattern removal, or CLI semantics. The skill's reviewer checklist should use `UNVERIFIED` status for every
    claim that came from `rg`/file references rather than from a focused test or CI result.

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
| Issue #1413 — Treat `Path(repo_root)` as obviously behavior-preserving | Plan proposed `resolve_repo_root()` returning `Path(repo_root)` for explicit values, but did not execute migrated callers | Relative paths, already-`Path` inputs, symlink resolution, and existence-check assumptions can differ across callers | Reviewer must confirm explicit-root callers only expected `Path(...)`, while auto-detect remains the only path that calls `get_repo_root()` |
| Issue #1413 — Exact-pattern-only fallback audit | Plan cited `rg` evidence for duplicated `args.repo_root` fallbacks but no implemented duplicate-pattern audit had run | Ternaries, explicit `if args.repo_root is not None`, local imports, and constructor-level fallbacks can evade a narrow regex | Use a broad discovery grep before editing and a strict zero-hit audit after editing; inspect excluded constructor call sites deliberately |
| Issue #1413 — Move branch-existence mechanics but drop the public wrapper | Plan moves implementation to `automation.git_utils` while keeping `pr_merge.local_branch_exists`, but this was not proven by a diff | `hephaestus.github.__init__` exports and tests may depend on the old seam; deleting it is a public API/test-patch regression | Keep the `pr_merge` wrapper and add delegation tests that patch the shared helper through the wrapper module |
| Issue #1413 — Generalize `run_git_cmd` with branch-existence cleanup | The plan correctly kept `run_git_cmd` local, but a broad refactor could still fold it into generic git utilities | PR-merge dry-run logging is not present in `automation.git_utils.run`; moving it would change behavior or broaden scope | Reviewer should block any `run_git_cmd` move unless dry-run semantics are intentionally designed and tested |
| Issue #1413 — Treat grep/file references as validation | Plan evidence was based on repo text searches and cited file lines, not code execution | No focused tests, lint, type check, or unit suite proved the proposed helper migration or subprocess behavior | Mark the workflow unverified and require the listed duplicate-pattern audit plus focused tests before accepting implementation claims |

## Results & Parameters

### Assumption Audit Checklist (copy-paste into plan PR description)

```
## Pre-implementation assumption audit

- [ ] Read `hephaestus/<package>/__init__.py` — does `__all__` exist? New symbols listed?
- [ ] Traced all `return`/`sys.exit` in target `main()` — do all output modes reach new checks?
- [ ] Counted test classes in source test file (`grep "^class Test" | wc -l`) — shim imports test classes (not symbols)?
- [ ] Verified `packaging` in `pyproject.toml [project.dependencies]`
- [ ] Same-name collision resolved: path-vs-string identified, `_extract_versions_from_text` helper added, shim aliases new name?
- [ ] Optional-root helper preserves explicit semantics: non-`None` returns `Path(repo_root)` only; `None` calls canonical `get_repo_root()`
- [ ] `get_repo_root()` marker-walk contract was not rewritten; nested-marker tests still cover first-match-up behavior
- [ ] Fallback-site audit covered exact `or`, ternary, explicit `if`, local import, and constructor-level patterns
- [ ] Public compatibility wrapper remains in the old module when callers/tests import or patch that seam
- [ ] Shared helper subprocess behavior matches the moved local behavior: cwd, timeout, `check=False`, `log_errors=False`, and handled exceptions
```

### Issue #1189 Specific Findings

| Assumption | Status | Correct Answer |
|------------|--------|----------------|
| `validation/__init__.py __all__` doesn't need updating | WRONG | Has explicit `__all__`; 9 new symbols must be added |
| `scripts_lib` test shim satisfies coverage | WRONG | Shim must import test classes; zero tests collected from symbol-only imports |
| JSON mode runs all checks | WRONG | `if args.json: ... return 0` exits before new CI sub-checks; move checks before branch |
| `packaging` is a declared dependency | UNVERIFIED | Not checked against `pyproject.toml` before plan was written |
| Same-name `extract_pyproject_versions` collision is safe to shim | WRONG | `path: Path` vs `content: str` — shim at wrong layer returns `{}` silently; fix = `extract_pyproject_versions_str` + `_extract_versions_from_text` helper |

### Issue #1413 Specific Findings

| Assumption | Status | Reviewer Focus |
|------------|--------|----------------|
| `resolve_repo_root(repo_root)` preserves explicit `--repo-root` behavior by returning `Path(repo_root)` | UNVERIFIED | Check migrated callers did not previously resolve, expand, validate existence, or rely on raw strings |
| `resolve_repo_root(None)` can delegate to `get_repo_root()` without changing marker-walk behavior | UNVERIFIED | Confirm `get_repo_root()` itself is unchanged and nested-marker tests still run |
| All repeated CLI fallback sites were found | UNVERIFIED | Run broad `rg` discovery before editing and strict zero-hit duplicate-pattern audit after editing |
| Object constructors such as `VersionManager(repo_root or get_repo_root())` are safe to exclude | UNVERIFIED | Review each exclusion by signature and caller contract; do not exclude because it is inconvenient |
| `pr_merge.local_branch_exists` can become a wrapper over `automation.git_utils.local_branch_exists` | UNVERIFIED | Preserve the exported old seam and patch target; add delegation tests at `hephaestus.github.pr_merge` |
| Shared branch helper behavior matches the old local command | UNVERIFIED | Check `git branch --list`, cwd, timeout constant, `check=False`, `log_errors=False`, stdout handling, and timeout/called-process exceptions |
| `pr_merge.run_subprocess` is already the canonical helper | TEXT-CHECKED ONLY | Add an identity regression test; grep absence of local `def run_subprocess` is not enough |
| `pr_merge.run_git_cmd` should stay local | REASONED, NOT RUN | Reviewer should reject scope creep unless dry-run logging is intentionally added to the shared runner and tested |

### Issue #1413 Plan Reviewer Checklist

```
## Optional repo-root + branch-helper refactor review

- [ ] Does `resolve_repo_root()` leave explicit values as `Path(repo_root)` and call `get_repo_root()` only for `None`?
- [ ] Did the diff avoid changing `get_repo_root()` marker-walk semantics and keep nested-marker tests passing?
- [ ] Did the before/after audit cover all fallback spellings, not just `args.repo_root or get_repo_root()`?
- [ ] Are excluded constructor call sites explained by their own contracts?
- [ ] Does `hephaestus.github.pr_merge.local_branch_exists` remain importable and tested as a wrapper?
- [ ] Does the shared branch helper preserve subprocess cwd, timeout, failure handling, and stdout truthiness?
- [ ] Is `run_git_cmd` still local unless dry-run semantics were deliberately designed for generic git utilities?
- [ ] Are validation claims based on focused tests/lint/type checks, not just `rg` output?
```

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning phase for issue #1189 (python-version-consistency consolidation) | v1.0.0 plan NOGO'd; v2.0.0 revised plan addresses all 5 failure modes; implementation pending |
| ProjectHephaestus | Planning phase for issue #1413 (centralize optional repo-root fallback and move branch-existence mechanics behind a compatibility wrapper) | Plan produced, NOT executed; v2.2.0 records unverified assumptions and reviewer risks from repo text searches/file references only |
