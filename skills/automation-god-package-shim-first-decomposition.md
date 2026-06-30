---
name: automation-god-package-shim-first-decomposition
description: >-
  Use when: (1) a flat Python sub-package has grown to 40+ co-located .py files
  (god-package) and needs reorganization into domain sub-packages without breaking
  any existing import sites, (2) planning a shim-first migration where the original
  module paths remain as thin re-export shims after symbols are moved to new
  sub-packages, (3) determining migration ordering for a package with an internal
  import graph (adapters first, orchestrators last), (4) verifying that every
  moved module defines __all__ before relying on wildcard re-exports in shims,
  (5) auditing for circular import risk when shims and moved modules both reference
  each other, (6) decomposing a flat package that is gated behind an optional
  install extra (e.g. [automation]) so the library/product boundary is preserved,
  (7) consolidating a small cluster of always-co-imported modules into ONE canonical
  module while keeping the original paths as explicit re-export shims (the inverse
  direction — merge, not split — but the same shim discipline applies),
  (8) decomposing a single oversized MODULE (one `.py` file, not a flat package) where
  you cannot leave a flat-file shim AND create a same-named package — turn the module
  into a `__init__`-as-shim PACKAGE instead, with a `_deps.py` indirection sub-module for
  heavily-patched imported-into names and shared mutable state (UNVERIFIED — plan only, #1455).
category: architecture
date: 2026-06-30
version: "2.2.0"
user-invocable: false
verification: verified-local
history: automation-god-package-shim-first-decomposition.history
tags:
  - python
  - refactoring
  - god-package
  - shim-pattern
  - sub-package
  - migration
  - import-compatibility
  - automation
  - flat-to-hierarchical
  - circular-imports
  - __all__
  - optional-extra
  - re-export
  - ruff-f401
  - module-consolidation
  - module-to-package
  - init-shim
  - deps-indirection
  - patch-seam
---

# Automation God-Package Shim-First Decomposition

## Overview

| Field | Value |
| ------- | ------- |
| **Date** | 2026-06-30 (v2.2.0) |
| **Objective** | (v1.0.0) Decompose a 52-file flat god-package into 8 domain sub-packages via shim files. (v2.0.0) ALSO covers the inverse: consolidate a small cluster of always-co-imported config modules into ONE canonical module, keeping the originals as explicit re-export shims — executed for real in ProjectHephaestus #1441. (v2.1.0) ALSO records a verified-local SPLIT/move execution (#1443) and the **whole-test-tree patch-seam sweep** it surfaced. (v2.2.0) ALSO adds a THIRD shape — the **module-→-package (`__init__`-as-shim)** variant with a **`_deps.py` indirection** for heavily-patched imported-into names (planned for #1455; **UNVERIFIED — plan only**) |
| **Outcome** | v1.0.0 plan for #1177 was never executed. v2.0.0 records a verified-local execution of the shim consolidation for #1441 ("Merge 4 Claude agent modules"): merge confirmed with ruff + mypy clean and a 145-test focused suite green. v2.1.0 records a verified-local SPLIT of 3 `*_state.py` modules into `state/` (#1443): full `tests/unit/automation` suite **2284 passed**, ruff + mypy clean — after a whole-test-tree patch-seam sweep fixed 4 failures the approved plan missed. v2.2.0 records a PLANNING session for #1455 (decompose the 2093-LoC, 46-function `github_api.py`): the module-→-package variant + `_deps`-indirection were designed but **NOT executed** — see the Risks note (R1, the patch-seam equivalence, is an open question) |
| **Trigger** | Three shapes: a flat package with 40+ .py files (SPLIT), a cluster of 3-4 tiny always-co-imported modules (MERGE), OR a single oversized MODULE decomposed into a `__init__`-as-shim PACKAGE (module-→-package, #1455, unverified). All keep original import paths as explicit re-export shims |
| **Verification** | verified-local for the executed MERGE (#1441) and SPLIT (#1443) core. The module-→-package variant + `_deps`-indirection (#1455) are **unverified (plan only)** — R1 (patch-seam equivalence) is unproven |
| **History** | [changelog](./automation-god-package-shim-first-decomposition.history) |

## When to Use

Apply this skill when any of the following is true:

- A Python sub-package has **40+ co-located .py files** with distinct domain clusters
- The package is imported by many external callers whose import paths **must not change**
- An existing **shim / re-export proof-of-concept** already exists in the package (e.g., a `prompts/` sub-directory with a backward-compat `prompts.py` shim)
- The package has a **lazy `__getattr__`** in its `__init__.py` and you want to preserve that boundary
- The package is **gated behind an optional install extra** and you must preserve that gate
- You need a **leaf-to-root migration ordering** to avoid creating circular imports during the migration
- You are planning (not yet executing) a migration and need to enumerate **unverified assumptions** for reviewer focus
- **(Merge direction, v2.0.0)** You have a small cluster of **3-4 tiny modules that are always imported together** (e.g. `claude_models.py` + `claude_timeouts.py` + `session_naming.py`) and want to consolidate them into ONE canonical module while leaving each original path as a thin explicit re-export shim
- You are writing an **explicit `from X import (name as name, ...)` re-export shim** and need to know whether `# ruff: noqa: F401` is required (it is NOT — and adding it triggers RUF100)
- A test reads a **private symbol** (`_KNOWN_MODELS`), calls `importlib.reload`, or pins a **logger name** (`caplog.at_level(..., logger="...")`) of a module you are about to turn into a shim — these tests CANNOT stay on the shim and must be repointed at the canonical module
- **(Split direction, v2.1.0)** A test anywhere in the tree `mock.patch("...<flat_path>.<name>")` or `monkeypatch.setattr`s a name that the module you are moving **imported from elsewhere** (not one of the module's OWN public symbols) — after the move that bound name lives in the canonical sub-package and the flat shim no longer carries it, so the patch target vanishes. These seams hide in OTHER test files, not just the moved module's own test file, and require a whole-test-tree grep sweep (Step 8b below) — the `dir(shim)` parity test cannot catch them
- **(Module-→-package, v2.2.0 — UNVERIFIED, plan only #1455)** The path you must decompose is itself a single oversized **module** (`github_api.py`, ~2000 LoC, 40+ functions), not a flat package. You cannot leave a flat-file `.py` shim AND create a same-named package — so the technique is to turn the module into a same-named **PACKAGE** whose `__init__.py` IS the shim (explicit `name as name` re-exports of every sub-module's public symbol). Under pytest `prepend` import mode the dotted path resolves to the package `__init__` identically to the old module, so import sites and `@patch("...<module>.X")` seams keep resolving
- **(Module-→-package, v2.2.0 — UNVERIFIED, plan only #1455)** The module being decomposed is patched dozens-to-hundreds of times in tests, and most patched names are imported INTO it (from sibling library modules) or are shared MUTABLE module state (caches, circuit breakers). Consider a single internal `_deps.py` sub-module that holds those names; sub-modules reference them as module-ATTRIBUTE access (`_deps.<name>`, NOT a `from _deps import <name>` bound copy) so one rebind is seen by all callers — BUT see Risk R1: the patch-seam equivalence is unproven, and a large fraction of patch sites may still need repointing to `<module>._deps.<name>`

## Verified Workflow

> **Verification level: verified-local.** The SPLIT workflow (Steps 0-9 below) was designed
> during planning for ProjectHephaestus #1177 and remains a proposal for the full 52-file split,
> but a **partial SPLIT was executed end-to-end** for ProjectHephaestus #1443 — moving the three
> `*_state.py` modules into a `state/` sub-package with explicit `name as name` shims — and is
> verified-local: ruff + mypy clean, full `tests/unit/automation` suite **2284 passed, 0 failed**.
> That execution surfaced the **whole-test-tree patch-seam sweep** (Step 8b below): the decisive
> finding that a name patched on the flat path keeps working after the move ONLY if it is one of
> the shim's OWN re-exported symbols — any name imported INTO the moved module must be patched on
> the canonical sub-package module instead, and those broken seams hide in OTHER test files.
> The MERGE workflow (the "Shim Consolidation" section directly below) was **executed end-to-end**
> for ProjectHephaestus #1441 and is verified-local: ruff + mypy clean, 145-test focused suite
> green. Where the two disagree, the executed lessons win — most importantly the
> **explicit-`as`-alias re-export shim needs NO `# ruff: noqa: F401`** (v1.0.0's templates added
> that noqa; that is now corrected — see the Shim Consolidation section and Failed Attempts).

### Shim Consolidation (Merge Direction — verified-local, ProjectHephaestus #1441)

Use this when the goal is to **merge** a few always-co-imported modules into one canonical
module (the inverse of the split below), keeping each original path as an explicit re-export shim.

```text
3-4 tiny modules always imported together?
└─ YES → Shim Consolidation
     1. Create agent_config.py with one labelled section per source module
        (paste each body VERBATIM under a comment banner).
     2. Repoint the ONE real cross-import (claude_invoke.py: from ...session_naming
        → from ...agent_config). Keep subprocess logic (claude_invoke.py) SEPARATE —
        merge config, not behavior.
     3. Replace each original module body with an EXPLICIT re-export shim:
          from hephaestus.automation.agent_config import (name as name, ...)
        Do NOT add `# ruff: noqa: F401`. The `as name` alias IS ruff's recognized
        re-export idiom; F401 never fires, so the noqa is unused → RUF100 failure.
     4. Sort the canonical module's __all__ with `ruff check --fix --unsafe-fixes`
        (RUF022 requires isort order — a flat alphabetical list; domain-grouped
        __all__ with comments FAILS RUF022).
     5. Repoint private-symbol / reload / caplog-logger-name tests at the canonical
        module (they cannot read privates through a shim — shims omit privates by design).
     6. Add a full-surface parity test: parametrize over each shim, assert every public
        name `is` the same object in agent_config (filter module objects out of dir()).
```

#### Step C1: Explicit re-export shim — NO `# ruff: noqa: F401`

```python
# hephaestus/automation/session_naming.py  (shim — replaces the original body)
"""Backward-compatibility shim. Canonical impl: hephaestus.automation.agent_config."""
from hephaestus.automation.agent_config import (
    build_session_name as build_session_name,
    SESSION_PREFIX as SESSION_PREFIX,
)

__all__ = ["build_session_name", "SESSION_PREFIX"]
```

The `name as name` re-binding is exactly what ruff treats as an intentional re-export, so
**F401 (`imported but unused`) is never raised on these lines.** Adding `# ruff: noqa: F401`
to silence a warning that does not fire makes the directive itself unused, and ruff's RUF100
(`unused noqa`) then fails CI. **This reverses v1.0.0's shim templates**, which carried
`# ruff: noqa: F401` — drop that comment from any explicit-`as`-alias shim.

#### Step C2: `__all__` must be isort-sorted (RUF022)

```bash
# Domain-grouped __all__ with comment banners FAILS RUF022. Let ruff sort it:
ruff check --fix --unsafe-fixes hephaestus/automation/agent_config.py
# Result: a single FLAT alphabetical list. Domain grouping is lost — accept it.
```

#### Step C3: Repoint private-symbol / reload / caplog tests

A test that reaches a module **private** symbol (`claude_models._KNOWN_MODELS`), calls
`importlib.reload(claude_models)`, or pins the module logger name in
`caplog.at_level(..., logger="hephaestus.automation.claude_models")` **cannot** keep targeting
the shim — the shim deliberately does not re-export privates, and the merged module's logger is
`hephaestus.automation.agent_config` (it uses `logging.getLogger(__name__)`). Fix with a local
alias plus a logger-name retarget; the alias keeps the rest of the test body unchanged:

```python
# was: from hephaestus.automation import claude_models
from hephaestus.automation import agent_config as claude_models  # local alias

# every caplog logger string retargets to the canonical module:
with caplog.at_level(logging.WARNING, logger="hephaestus.automation.agent_config"):
    ...
```

#### Step C4: Full-surface shim-parity test

A missing re-export is only an `AttributeError` at the call site, so focused tests miss drift.
Parametrize over every shim and assert object identity against the canonical module:

```python
import importlib
import pytest
from hephaestus.automation import agent_config

@pytest.mark.parametrize(
    "shim_name",
    ["claude_models", "claude_timeouts", "session_naming"],
)
def test_shim_reexports_match_canonical(shim_name):
    shim = importlib.import_module(f"hephaestus.automation.{shim_name}")
    public = [n for n in dir(shim) if not n.startswith("_")]
    for name in public:
        obj = getattr(shim, name)
        # filter imported module objects (logging, os) out of dir()
        if getattr(obj, "__class__", None).__name__ == "module":
            continue
        assert getattr(agent_config, name) is obj, f"{shim_name}.{name} drifted from agent_config"
```

### Module-→-Package (`__init__`-as-Shim) — UNVERIFIED (plan only, #1455)

> **Verification: UNVERIFIED. This entire subsection is a PLAN that was never executed.**
> It was designed during a PLANNING session for ProjectHephaestus #1455 ("decompose
> `hephaestus/automation/github_api.py` — a 2093-LoC, 46-function SRP violation"). The
> module-→-package shape and the `_deps`-indirection pattern below are a **proposed extension**
> of the verified workflow, not a verified-local result. **Risk R1 (the patch-seam equivalence)
> is an open design question** — see "Risks / Unverified Assumptions (#1455)" below. Do NOT treat
> any claim in this subsection as proven. The skill's overall `verification: verified-local`
> field reflects only the executed #1441 (MERGE) and #1443 (SPLIT) core.

Use this when the path to decompose is a single oversized **module** (one `.py` file), not a
flat package. The MERGE and SPLIT variants both assume the unit you keep as a shim can stay a
flat `.py` file. But when `github_api.py` itself must be broken up, you cannot keep a flat-file
shim *and* introduce a same-named `github_api/` package — the two collide on the same dotted path.

**The technique (proposed, unverified):**

1. Turn `github_api.py` into a `github_api/` **package**. Move the 46 functions into
   responsibility-clustered sub-modules (`_diff.py`, `_reviews.py`, `_issues.py`, `_labels.py`,
   …) in leaf-to-root order (a single grep pass suggested a 9-way clustering — re-verify, see R3).
2. The package `__init__.py` IS the shim: explicit `name as name` re-exports of every
   sub-module's public symbol. Under pytest `prepend` import mode (`pyproject.toml`:
   `pythonpath=["."]`, no `importmode=importlib`), the dotted path
   `hephaestus.automation.github_api` resolves to the package `__init__` exactly as the old
   module did, so import sites and `@patch("...github_api.X")` seams keep resolving — *for names
   defined/re-exported directly on the package namespace* (but see R1 for imported-into names).
3. **`_deps.py` indirection for heavily-patched imported-into names + shared mutable state.**
   `github_api` is patched ~200× in tests (`github_api._gh_call` alone 88×, `get_repo_info` 34×,
   `io_write_secure` 11×, `run` 6×) and holds shared MUTABLE state (`_label_cache`,
   `_issue_state_cache`, `_GH_BREAKER`, `_GH_THROTTLE`). Most of these names are imported INTO
   `github_api` from sibling library modules (`hephaestus.github.client`, `.git_utils`,
   `hephaestus.io.utils`) — exactly the Step-8b hazard. The PROPOSED pattern: put all shared
   mutable state + imported-through deps in ONE internal `_deps.py`; every sub-module references
   them as **module-attribute access** `_deps.<name>` (NOT `from _deps import <name>`, which
   would bind a copy) so a single rebind of `_deps.<name>` is seen by all callers; the package
   `__init__` re-exports `<name> = _deps.<name>` as the public patch seam.

   ```python
   # github_api/_deps.py  (internal — single source of imported-into deps + mutable state)
   from hephaestus.github.client import _gh_call, get_repo_info
   from hephaestus.io.utils import write_secure as io_write_secure
   _label_cache: set[str] | None = None
   _issue_state_cache: dict = {}

   # github_api/_diff.py  (a sub-module — attribute access, NOT `from _deps import _gh_call`)
   from . import _deps
   def get_pr_diff(pr): return _deps._gh_call(...)   # one rebind of _deps._gh_call reaches here

   # github_api/__init__.py  (the shim — public patch seam)
   from . import _deps
   _gh_call = _deps._gh_call            # public re-export (BUT see R1 — may not be the live seam)
   _label_cache = _deps._label_cache    # binds the INITIAL object (BUT see R2 — aliasing breaks on rebind)
   ```

**`__all__` is NOT a reliable public-surface inventory here.** `github_api.py`'s `__all__`
(`:64-70`, approximate — re-confirm, R4) lists only 5 re-exported exception/helper names, NOT its
46 own functions. So the wildcard-`import *` shim variant is UNSAFE for this module, and the
explicit-per-symbol shim is mandatory. This also means the Step-0 pre-flight `grep -L '__all__'`
check is **necessary-but-insufficient**: a module CAN define `__all__` and still have it cover
only a fraction of the real public surface. After confirming a module defines `__all__`, also
confirm `__all__` actually enumerates every public symbol you intend the shim to carry — diff
`set(__all__)` against the module's public `def`/`class`/assignment names.

#### Risks / Unverified Assumptions (#1455) — for reviewer focus

| # | Risk (unverified, plan only) | Why it may break | What the reviewer must demand |
| --- | ---------------------------- | ---------------- | ----------------------------- |
| **R1** *(highest uncertainty — central design risk)* | The `_deps` patch-seam equivalence is **ASSUMED, not proven**. The plan claims re-exporting `_gh_call = _deps._gh_call` in `__init__` makes `@patch("github_api._gh_call")` reach sub-modules that call `_deps._gh_call(...)` | `mock.patch("github_api._gh_call")` rebinds the attribute on the `github_api` PACKAGE namespace, but sub-modules read `_deps._gh_call` — a DIFFERENT namespace. The patch is NOT seen unless tests repoint to `@patch("github_api._deps._gh_call")`. The plan waves at "both targets work" with no mechanism | A concrete proof-of-mechanism: one worked example showing a `github_api._gh_call` patch ACTUALLY intercepting a sub-module call. Realistic outcome: a large fraction of the ~200 patch sites must repoint to `github_api._deps.<name>` (or sub-modules use late `from . import github_api` binding) — contradicting the "zero churn / stay unchanged" claim |
| **R2** | Mutable-state aliasing breaks on rebind | `_label_cache` / `_issue_state_cache` are REASSIGNED at runtime (`_deps._label_cache = parsed`). `__init__`'s `_label_cache = _deps._label_cache` binds the INITIAL object; after a sub-module reassigns `_deps._label_cache`, `github_api._label_cache` still points at the stale initial value | Repoint the 2 tests reading/patching `github_api._label_cache` / `_issue_state_cache` to `github_api._deps.<name>` — the plan only half-commits to this |
| **R3** | Leaf/ordering and counts are unverified | "46 functions", the 9-way clustering, the `diff`→`reviews` leaf ordering, and the "~600 LoC per sub-module" target are from a SINGLE grep pass; counts/clusters go stale (cf. the dry-refactoring count-staleness lesson) | Re-grep the live file; confirm no sub-module retains a hidden cross-dependency (reviews→threads, issues→labels) beyond the assumed `_deps`-only edges |
| **R4** | Line citations are approximate | `github_api.py:22-36` (import block) and `:64-70` (`__all__`) were read once | Treat as approximate; re-confirm before editing |

### Quick Reference

```text
Decision tree for flat god-package decomposition:

  40+ flat .py files, identifiable domains?
  └─ YES → Shim-First Migration (this skill)
       ├─ Step 0: Pre-flight checks (run BEFORE any file moves)
       │    ├─ grep -L '__all__' hephaestus/automation/*.py  → must be EMPTY
       │    ├─ grep -rn 'importmode' pyproject.toml           → note actual mode
       │    └─ audit internal relative imports in files to move
       ├─ Step 1: Create domain sub-package directories + __init__.py stubs
       ├─ Step 2: Move modules in leaf-to-root order (adapters first, orchestrators last)
       ├─ Step 3: Convert each original .py to a shim (from .sub.module import * + __all__)
       ├─ Step 4: Run full test suite after EACH move (not batch)
       └─ Step 5: Validate with scripts/validate_plugins.py equivalent (project-specific)

Leaf-to-root ordering for packages with internal import graph:
  adapters/ → claude/ → state/ → reviewers/ → phases/ → planner/ → implementer/
  (imports flow toward right; move left-most first to avoid dangling references)
```

### Step 0: Pre-flight Checks (MUST Run Before Any File Moves)

These checks validate assumptions the shim pattern relies on. Skipping them causes
silent breakage that is hard to diagnose after the fact.

**Check 1: Every module to be moved must define `__all__`**

```bash
# This must produce ZERO output before proceeding
grep -L '__all__' hephaestus/automation/*.py
```

Why: The shim pattern uses `from hephaestus.automation.adapters.github_api import *`.
Without `__all__`, wildcard import exports all names, including private `_` symbols —
worse than the pre-shim state. For any file without `__all__`, either:
- Add an explicit `__all__` before moving, OR
- Use explicit per-symbol re-exports in the shim instead of `*`

**Check 2: Verify the actual pytest `--import-mode` setting**

```bash
grep -A5 '\[tool.pytest.ini_options\]' pyproject.toml | grep -E 'importmode|addopts'
```

Why: The plan assumes `importmode = importlib`. If the project uses the default
`prepend` mode, test file locations within the package matter differently. Confirm
before assuming test files must be moved alongside modules.

**Check 3: Audit internal relative imports for circular-import risk**

For every file X being turned into a shim, check whether any file in the new
sub-package imports from X (which would create a shim→sub-package→shim cycle):

```bash
# For each moved module, check for imports back to the original flat path
module="github_api"
grep -rn "from hephaestus.automation.${module}" hephaestus/automation/adapters/
grep -rn "from \. import ${module}" hephaestus/automation/adapters/
```

**Check 4: Verify shared-ownership claims for borderline files**

Before placing a file at the top level (shared ownership claim), verify by reading it:

```bash
# Verify _stage_context.py is actually imported by both phases/ and implementer/
grep -rn 'stage_context\|_stage_context' hephaestus/automation/
```

**Check 5: Verify re-exported function names exist in moved modules**

If verification criteria reference specific function names (e.g. `call_graphql`,
`run_git`), confirm those names exist in `__all__` before writing the criteria:

```bash
python3 -c "
import ast, pathlib
for f in pathlib.Path('hephaestus/automation').glob('*.py'):
    src = f.read_text()
    tree = ast.parse(src)
    for node in ast.walk(tree):
        if isinstance(node, ast.Assign):
            for t in node.targets:
                if isinstance(t, ast.Name) and t.id == '__all__':
                    print(f'{f.name}: {ast.literal_eval(node.value)}')
"
```

### Step 1: Map Domain Clusters

Enumerate files by domain cluster before any structural changes:

```bash
# Count and list files by inferred domain prefix
ls hephaestus/automation/*.py | xargs -n1 basename | sort | head -60
wc -l hephaestus/automation/*.py | sort -rn | head -20
```

Proposed sub-packages for a typical automation pipeline package:

| Sub-package | Content | Moved-from patterns |
| ----------- | ------- | ------------------- |
| `adapters/` | External API wrappers (GitHub, git) | `github_api.py`, `git_utils.py`, `graphql_*.py` |
| `claude/` | LLM invocation wrappers | `claude_*.py`, `invoke_*.py` |
| `state/` | Shared state, context, stores | `*_state.py`, `*_context.py`, `*_store.py` |
| `reviewers/` | Code review automation | `reviewer*.py`, `review_*.py` |
| `phases/` | Stage/phase runners | `*_phase*.py`, `stages.py` |
| `planner/` | Planning pipeline | `planner*.py`, `plan_*.py` |
| `implementer/` | Implementation pipeline | `implementer*.py` |
| `ui/` | Terminal/curses UI | `curses*.py`, `*_ui.py` |

**Files that may need shared placement** (borderline): Any file imported by 3+
domain clusters belongs at the top level, not in any single sub-package. Verify
with `grep -rn 'import <filename>' hephaestus/automation/` before deciding.

### Step 2: Create Sub-Package Structure

```bash
# Create each sub-package directory with an empty __init__.py
for subpkg in adapters claude state reviewers phases planner implementer ui; do
    mkdir -p "hephaestus/automation/${subpkg}"
    touch "hephaestus/automation/${subpkg}/__init__.py"
done
```

The `__init__.py` files start empty; they are populated during Step 4.

### Step 3: Move Modules in Leaf-to-Root Order

Move one module at a time. Never move a module before all modules it imports have
already been moved (or are staying at the top level).

```bash
# Example: move github_api.py to adapters/
git mv hephaestus/automation/github_api.py hephaestus/automation/adapters/github_api.py
```

After moving, update the module's own `from . import X` / `from hephaestus.automation.X`
imports to use the new location — but do NOT update callers yet (that's the shim's job).

### Step 4: Create Shim Files

For each moved module, replace the original path with a thin shim:

**Option A: Wildcard shim** (requires `__all__` in moved module — VERIFY FIRST):

```python
# hephaestus/automation/github_api.py  (shim — replaces original)
"""Backward-compatibility shim. Real implementation in adapters/github_api.py."""
# ruff: noqa: F401, F403
from hephaestus.automation.adapters.github_api import *  # noqa: F401, F403
from hephaestus.automation.adapters.github_api import __all__
```

**Option B: Explicit per-symbol shim** (safer; use when `__all__` is absent or opaque):

```python
# hephaestus/automation/github_api.py  (shim — replaces original)
"""Backward-compatibility shim. Real implementation in adapters/github_api.py."""
from hephaestus.automation.adapters.github_api import (
    call_graphql as call_graphql,
    get_pr_diff as get_pr_diff,
    # ... enumerate all public symbols
)

__all__ = [
    "call_graphql",
    "get_pr_diff",
    # ...
]
```

> **v2.0.0 correction (verified-local, #1441):** do **NOT** add `# noqa: F401` /
> `# ruff: noqa: F401` to an explicit `name as name` re-export shim. The `as` re-binding is
> ruff's recognized re-export idiom, so F401 never fires — the directive is unused and RUF100
> then fails CI. (v1.0.0 carried `# noqa: F401` here; it is removed.) The noqa is only needed for
> the wildcard `import *` variant below, which genuinely triggers F403/F401.

Option B is preferred because it:
1. Does not depend on the moved module defining `__all__`
2. Makes the shim's public surface explicit (visible to static analysis)
3. Does not accidentally export private symbols
4. Needs **no** `# noqa` — keeping it lint-clean without suppression (RUF100-safe)

**The `prompts/` sub-directory** in `hephaestus/automation/` is the existing proof-of-concept
for this pattern (28 symbols, verified working). Examine it before writing the first shim.

### Step 5: Update Sub-Package `__init__.py`

For each sub-package, decide whether to aggregate exports in `__init__.py`:

```python
# hephaestus/automation/adapters/__init__.py
# Option 1: Transparent (no aggregation) — callers must use full path
#   from hephaestus.automation.adapters.github_api import call_graphql

# Option 2: Aggregated — all sub-module symbols accessible at adapters level
from hephaestus.automation.adapters.github_api import (  # noqa: F401
    call_graphql as call_graphql,
)
```

Recommendation: Start with Option 1 (transparent) to minimize the blast radius of
circular import risk. Aggregation in `__init__.py` can be added later.

### Step 6: Run Tests After Each Move

```bash
# After each individual module move + shim creation:
pixi run pytest tests/unit/automation/ -q -x   # stop at first failure
pixi run pytest tests/integration/ -q -x
```

Never batch multiple moves before testing. If a circular import is introduced,
the error will point to the specific shim, not to a confusing multi-file chain.

### Step 7: Verify Backward Compatibility

For each shim, verify the original import path still works:

```bash
python3 -c "
# Verify each original import path still resolves
from hephaestus.automation.github_api import call_graphql
from hephaestus.automation.git_utils import run_git
# etc.
print('All shims resolve correctly')
"
```

### Step 8: Update `test_automation_boundary.py`

Verify the boundary test is not affected by the reorganization:

```bash
# Read the test to confirm LIB_ROOT excludes hephaestus/automation/
grep -n 'LIB_ROOT\|automation' tests/unit/test_automation_boundary.py | head -20
pixi run pytest tests/unit/test_automation_boundary.py -v
```

### Step 8b: Patch-Seam Sweep Across the WHOLE Test Tree (verified-local, #1443)

> **This is a separate, mandatory check — NOT subsumed by the full-surface parity test.**
> A shim re-exports only the moved module's OWN public symbols. Any name the module
> *imported from elsewhere* (e.g. `prefetch_issue_states` imported into `planner_state`
> from `..github_api`) is intentionally absent from the shim, so a `dir(shim)` parity
> test will never flag it. But `mock.patch("...<flat_path>.<imported_name>")` rebinds the
> attribute on the *named* module — and after the move that name no longer lives on the
> flat shim, so the patch target vanishes. These broken seams hide in **OTHER** test files,
> not just the moved module's own test file.

Before declaring a shim-first move done, grep the WHOLE test tree for every patch /
monkeypatch / setattr against each moved module's flat namespace and repoint each hit:

```bash
# For each moved module, find EVERY patch/monkeypatch against its flat namespace across ALL tests:
for m in planner_state implementer_state review_state; do
  grep -rn "hephaestus\.automation\.${m}\." tests/ | grep -iE "patch|monkeypatch|setattr"
done
# Repoint imported-into names to the canonical module (state.<module>); only the shim's OWN
# re-exported symbols keep working on the flat path.
```

**Concrete miss (#1443):** the approved plan listed patch-string repoints only for the three
moved test files (`test_planner.py` / `test_implementer.py` / `test_review.py`) plus the known
`test_implementer_loop.py` coupling. It MISSED `tests/unit/automation/test_planner_loop.py` and
`tests/unit/automation/test_planner_main.py`, which patched
`hephaestus.automation.planner_state.prefetch_issue_states` and
`hephaestus.automation.planner_state.fetch_all_issue_labels_graphql` — names imported INTO
`planner_state` from `..github_api` and `.review`. After the move those bound names live in
`hephaestus.automation.state.planner`, and the flat `planner_state` shim re-exports only
`PlannerStateManager` / `_comments_contain_plan`, so the patch targets vanished → **4 test
failures that surfaced only on the first FULL-suite run.** Fix: repoint to
`hephaestus.automation.state.planner.*`. This is the split-direction analogue of Step C3
(which covers private-symbol / reload / caplog repoints in the MERGE direction).

### Step 9: Final Structural Verification

```bash
# New sub-package directories exist
ls hephaestus/automation/*/

# Original flat .py shims still present
ls hephaestus/automation/*.py | wc -l   # count unchanged or increased

# No wildcard imports (unless __all__ verified)
grep -rn 'from hephaestus.automation.*import \*' hephaestus/automation/*.py

# Full test suite passes
pixi run pytest tests/ -q
pixi run ruff check hephaestus/ tests/
pixi run mypy
```

## Unverified Assumptions (Risks for Reviewer Focus)

These seven assumptions were NOT directly verified during planning for issue #1177.
A reviewer or implementer should check each one before executing the migration.

| # | Assumption | Risk if Wrong | Verification Command |
| --- | ---------- | ------------- | -------------------- |
| 1 | Every moved module defines `__all__` | Wildcard shim exports private `_` symbols | `grep -L '__all__' hephaestus/automation/*.py` — must be empty |
| 2 | `pytest --import-mode=importlib` is the actual mode | `prepend` mode may not require moving test files alongside modules | `grep -A5 '\[tool.pytest.ini_options\]' pyproject.toml` |
| 3 | `_stage_context.py` is imported by both `phases/` and `implementer/` (shared ownership) | It belongs in `implementer/` if only used there | `grep -rn '_stage_context\|stage_context' hephaestus/automation/` |
| 4 | `call_graphql` and `run_git` are exported names in their modules | Verification criterion tests a false green | `python3 -c "from hephaestus.automation.github_api import call_graphql"` (pre-move) |
| 5 | `test_automation_boundary.py` scans only `LIB_ROOT` (excludes automation/) | Reorganization could break boundary test | `grep -n 'LIB_ROOT' tests/unit/test_automation_boundary.py` |
| 6 | No existing internal relative imports within moved files point back at the flat package | Shim circular imports result | Per-file audit: `grep -n 'from hephaestus.automation import\|from \. import' <file>` |
| 7 | Creating `hephaestus/automation/claude/` does not shadow the `claude` top-level CLI | Unlikely, but unverified | `python3 -c "import claude"` before and after (confirm it's a CLI binary, not a Python package) |

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
| --------- | ---------------- | --------------- | ---------------- |
| Add `# ruff: noqa: F401` to an explicit `as`-alias re-export shim | Followed v1.0.0's shim template, which carried `# ruff: noqa: F401` | The `name as name` re-binding is ruff's recognized re-export idiom, so F401 never fires; the unused directive then trips RUF100 (`unused noqa`) and fails CI | NEVER add `# ruff: noqa: F401` to an explicit-`as`-alias shim. Only the wildcard `import *` variant needs F403/F401 suppression |
| Group `__all__` by domain with comment banners in the merged module | Wrote a readable, domain-clustered `__all__` for agent_config.py | ruff RUF022 requires `__all__` to be isort-sorted (flat alphabetical); domain grouping with comments fails the check | Run `ruff check --fix --unsafe-fixes` to sort `__all__`; accept the flat alphabetical list — domain grouping is lost |
| Leave a private-symbol / reload / caplog-logger test pointed at the shim | Kept `test_claude_models.py` importing the original (now-shim) module path | The shim deliberately omits privates (`_KNOWN_MODELS`), and the merged module's logger name changed to `hephaestus.automation.agent_config` (uses `getLogger(__name__)`), so `caplog.at_level(..., logger="...claude_models")` captured nothing | Repoint such tests via a local alias `from ... import agent_config as claude_models` AND retarget every caplog logger-name string to the canonical module's logger |
| Rely only on focused per-module tests to catch shim drift | Assumed the focused suites covered every re-exported symbol | A missing re-export surfaces only as an `AttributeError` at a future call site — focused tests that don't touch that symbol stay green | Add a parametrized full-surface parity test asserting every public name `is` the same object in the canonical module (filter module objects out of `dir()`) |
| Repoint patch seams only in the moved module's own test file | Followed a plan that listed patch-string repoints for the three moved test files + one known coupling | `test_planner_loop.py` / `test_planner_main.py` patched `planner_state.prefetch_issue_states` / `.fetch_all_issue_labels_graphql` (names imported INTO the module, NOT re-exported by the shim); 4 failures surfaced only on the first full-suite run | Grep the WHOLE test tree for `patch("<flat_path>.<anyname>")`; repoint every imported-into name to the canonical module — the shim carries only the module's OWN symbols |
| Assume `@patch("github_api._gh_call")` reaches sub-modules calling `_deps._gh_call` *(unverified — plan only, #1455)* | The #1455 plan re-exported `_gh_call = _deps._gh_call` in the package `__init__` and assumed both patch targets are equivalent | `mock.patch` rebinds the attribute on the `github_api` PACKAGE namespace; sub-modules read `_deps._gh_call` on a DIFFERENT namespace, so the patch is not seen there (UNVERIFIED — this is the predicted failure, R1, not an observed one) | Demand a proof-of-mechanism worked example FIRST; expect to repoint a large fraction of the ~200 patch sites to `github_api._deps.<name>` rather than claiming "function patches stay unchanged" |
| Re-export reassigned mutable state via `_label_cache = _deps._label_cache` in `__init__` *(unverified — plan only, #1455)* | The #1455 plan bound the package-level `_label_cache` to `_deps._label_cache` once at import | After a sub-module REASSIGNS `_deps._label_cache = parsed`, the `__init__`-level alias still points at the stale initial object (UNVERIFIED — predicted, R2) | Repoint tests that read/patch `github_api._label_cache` / `_issue_state_cache` to `github_api._deps.<name>`; do not rely on the `__init__` alias for reassigned state |
| Trust `__all__` as the public-surface inventory before a module-→-package shim *(unverified — plan only, #1455)* | `github_api.py` defines `__all__`, so the pre-flight `grep -L '__all__'` check passed | `__all__` listed only 5 re-exported exception/helper names, NOT the module's 46 own functions — a wildcard `import *` shim would have dropped them all (R-context) | The `grep -L '__all__'` check is necessary-but-insufficient; also diff `set(__all__)` against the module's actual public def/class names, and prefer the explicit-per-symbol shim |

## Results & Parameters

### Scale Parameters (ProjectHephaestus issue #1177)

| Parameter | Value |
| ---------- | ----- |
| Source directory | `hephaestus/automation/` |
| Total LOC | 25,403 |
| Total files | 52 flat .py files |
| % of source tree | 54.4% |
| Proposed sub-packages | 8 |
| Existing shim proof-of-concept | `prompts/` sub-directory (28 symbols) |
| Optional-extra gate | `HomericIntelligence-Hephaestus[automation]` |
| Lazy `__getattr__` pattern | `automation/__init__.py` (established pattern) |

### Merge-Direction Parameters (ProjectHephaestus issue #1441 — verified-local)

| Parameter | Value |
| ---------- | ----- |
| Canonical module created | `hephaestus/automation/agent_config.py` |
| Modules merged into it | `claude_models.py`, `claude_timeouts.py`, `session_naming.py` (3 always-co-imported config modules) |
| Module kept SEPARATE | `claude_invoke.py` (subprocess logic — merge config, not behavior) |
| Original paths retained as | thin explicit `from agent_config import (name as name, ...)` re-export shims |
| Real cross-import repointed | `claude_invoke.py`: `from ...session_naming` → `from ...agent_config` (the ONE genuine internal import) |
| `__all__` discipline | flat isort-sorted via `ruff check --fix --unsafe-fixes` (RUF022); domain grouping fails |
| noqa discipline | NO `# ruff: noqa: F401` on explicit-`as` shims (would trip RUF100) |
| Tests repointed | `test_claude_models.py` (private `_KNOWN_MODELS` + `importlib.reload` + caplog logger name → `hephaestus.automation.agent_config`) |
| New test added | full-surface shim-parity test (parametrized `is`-identity over each shim) |
| Verification | ruff clean (hephaestus/ + tests/); mypy clean (450 source files); 145-test focused suite green; full unit suite re-running at capture (verified-local, NOT verified-ci) |

### Split-Direction Parameters (ProjectHephaestus issue #1443 — verified-local)

| Parameter | Value |
| ---------- | ----- |
| Sub-package created | `hephaestus/automation/state/` |
| Modules moved into it | `planner_state.py` → `state/planner.py`, `implementer_state.py` → `state/implementer.py`, `review_state.py` → `state/review.py` |
| Original paths retained as | thin explicit `from ...state.<module> import (name as name, ...)` re-export shims at the old flat paths |
| Cross-import repointed | `_review_phase.py`: `from .state import review as review_state` (consumes the moved module via canonical path) |
| Deliberately left on shim paths | `implementer_phase_runner.py:42,77` — kept on the flat shim paths to preserve the #714 patch surface and the no-cycle guard |
| **Decisive fix** | **whole-test-tree patch-seam sweep** — `test_planner_loop.py` / `test_planner_main.py` patched `planner_state.prefetch_issue_states` / `.fetch_all_issue_labels_graphql` (imported-into names, NOT shim symbols); repointed to `state.planner.*` |
| Failures surfaced | 4 — only on the first FULL `tests/unit/automation` run (the approved plan's per-moved-file seam list missed the two cross-file patches) |
| Verification | ruff clean; mypy clean; full `tests/unit/automation` suite **2284 passed, 0 failed** (verified-local) |

### Leaf-to-Root Migration Ordering (Proposed)

```text
Round 1 (no internal deps):   adapters/     — github_api, git_utils, graphql helpers
Round 2 (deps on adapters):   claude/       — LLM invocation wrappers
Round 3 (deps on adapters):   state/        — shared state, stores, context
Round 4 (deps on state):      reviewers/    — review automation
Round 5 (deps on reviewers):  phases/       — stage/phase runners
Round 6 (deps on phases):     planner/      — planning pipeline
Round 7 (deps on planner):    implementer/  — implementation pipeline (largest)
Shared (top-level stays):     top-level     — _stage_context.py, any file with 3+ importers
```

### Shim Template (Explicit Variant — Preferred)

```python
"""Backward-compatibility shim for <module_name>.

Real implementation: hephaestus.automation.<subpkg>.<module_name>
This file is retained so existing import sites need no changes.
"""
# NOTE: no `# ruff: noqa: F401` here — the explicit `as` re-export idiom does not
# trigger F401, so a noqa would be unused and fail RUF100 (verified-local, #1441).
from hephaestus.automation.<subpkg>.<module_name> import (
    PublicClass as PublicClass,
    public_function as public_function,
    PUBLIC_CONSTANT as PUBLIC_CONSTANT,
)

__all__ = [
    "PublicClass",
    "public_function",
    "PUBLIC_CONSTANT",
]
```

### Shim Template (Wildcard Variant — Only When `__all__` Verified)

```python
"""Backward-compatibility shim for <module_name>.

Real implementation: hephaestus.automation.<subpkg>.<module_name>
REQUIRES: hephaestus.automation.<subpkg>.<module_name> defines __all__
"""
# ruff: noqa: F401, F403
from hephaestus.automation.<subpkg>.<module_name> import *  # noqa: F401, F403
from hephaestus.automation.<subpkg>.<module_name> import __all__
```

### Pre-Flight Checklist (Copy-Paste)

```bash
# 1. All modules define __all__ (must produce ZERO output)
grep -L '__all__' hephaestus/automation/*.py

# 2. pytest import mode
grep -A5 '\[tool.pytest.ini_options\]' pyproject.toml | grep -E 'importmode|addopts'

# 3. _stage_context.py importers
grep -rn '_stage_context\|stage_context' hephaestus/automation/ | grep -v '_stage_context.py'

# 4. call_graphql export name
python3 -c "from hephaestus.automation.github_api import call_graphql; print('OK')"

# 5. boundary test scope
grep -n 'LIB_ROOT\|automation' tests/unit/test_automation_boundary.py | head -20

# 6. claude/ sub-package shadow risk
python3 -c "import claude; print(type(claude), getattr(claude, '__file__', 'no __file__'))"
```

## Verified On

| Project | Context | Details |
| --------- | --------- | --------- |
| ProjectHephaestus | issue #1441 "Merge 4 Claude agent modules" (executed) | **verified-local** — Shim Consolidation (merge direction) run end-to-end: agent_config.py created, 3 modules turned into explicit re-export shims, claude_invoke.py kept separate; ruff + mypy clean, 145-test focused suite green (full suite re-running at capture) |
| ProjectHephaestus | issue #1443 "Move *_state.py into state/" (executed, SPLIT direction) | **verified-local** — 3 modules moved into `state/` with explicit `name as name` shims; `_review_phase.py` repointed to `from .state import review as review_state`; `implementer_phase_runner.py:42,77` left on shim paths (preserves #714 patch surface + no-cycle guard). The whole-test-tree patch-seam sweep was the decisive fix (4 failures from imported-into names patched on the flat path). ruff + mypy clean; full `tests/unit/automation` suite 2284 passed, 0 failed |
| ProjectHephaestus | Plan written for issue #1177 (not yet executed) | Verification level: unverified; the full 52-file SPLIT workflow remains a proposal |
| ProjectHephaestus | Plan written for issue #1455 "decompose `github_api.py` (2093 LoC, 46 fns)" (NOT executed) | **unverified (plan only)** — module-→-package (`__init__`-as-shim) variant + `_deps`-indirection for the ~200 patch sites + shared mutable state. R1 (the `_deps` patch-seam equivalence) is an open question; see the "Risks / Unverified Assumptions (#1455)" table |

## References

- [ProjectHephaestus issue #1441](https://github.com/HomericIntelligence/ProjectHephaestus/issues/1441) — merge direction (Shim Consolidation), verified-local execution
- [ProjectHephaestus issue #1443](https://github.com/HomericIntelligence/ProjectHephaestus/issues/1443) — split direction (move `*_state.py` into `state/`), verified-local; surfaced the whole-test-tree patch-seam sweep
- [ProjectHephaestus issue #1177](https://github.com/HomericIntelligence/ProjectHephaestus/issues/1177) — source issue for the full 52-file split workflow
- [ProjectHephaestus issue #1455](https://github.com/HomericIntelligence/ProjectHephaestus/issues/1455) — decompose `github_api.py` (2093 LoC, 46 fns); source of the **unverified** module-→-package (`__init__`-as-shim) variant + `_deps`-indirection (plan only, R1 unproven)
- [python-module-decomposition-and-refactor-patterns.md](python-module-decomposition-and-refactor-patterns.md) — single-module decomposition (Phase 11/12 for CLI extraction and sibling-cycle fixes)
- [python-circular-import-symbol-extraction.md](python-circular-import-symbol-extraction.md) — leaf-module extraction for circular import errors
