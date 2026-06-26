---
name: python-delegation-refactor-plan-risks
description: "Planning-risk checklist for Python refactors that replace explicit wrapper methods with allowlisted dynamic delegation while preserving legacy patch.object seams. Use when: (1) moving pure pass-through methods behind __getattr__, (2) exposing collaborator objects through read-only properties, (3) preserving unittest.mock.patch.object(instance, \"_method\", ...) compatibility, (4) reviewing plans that rely on exact line numbers and grep snapshots, (5) checking that no blanket proxy or method-identity regression slipped into a delegation refactor."
category: architecture
date: 2026-06-26
version: "1.0.0"
user-invocable: false
verification: unverified
tags: [python, delegation, getattr, refactoring, planning, patch-object, dynamic-attributes, phase-runner, reviewer-risks]
---

# Python Delegation Refactor Plan Risks

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Capture durable reviewer and implementer risks from the unexecuted ProjectHephaestus issue #1389 plan to refactor `IssueImplementer` pass-through wrappers into read-only phase properties plus an allowlisted `__getattr__` delegation path. |
| **Outcome** | Planning artifact only. No implementation, tests, or CI were run in the capture session. The value is the risk checklist and verification commands an implementer should run before trusting the plan. |
| **Verification** | unverified - plan not implemented, no local test run, no CI observed |

## When to Use

- Reviewing or implementing a Python refactor that deletes explicit `self.runner._method(...)` wrapper methods and resolves them dynamically through `__getattr__`.
- Preserving old `patch.object(instance, "_method", ...)` tests while moving low-level behavior to a collaborator object.
- Adding read-only collaborator properties such as `impl.review_phase` or `impl.followup_phase` and expecting future tests to patch those directly.
- Auditing a plan that cites exact line ranges, grep output, or issue-provided skill versions without re-running them in the current checkout.
- Checking that a delegation refactor is allowlisted and strict, not a blanket proxy that silently exposes all collaborator attributes.

## Proposed Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms.

### Quick Reference

```bash
# Re-run every snapshot claim before implementing; do not trust stale plan lines.
nl -ba hephaestus/automation/implementer.py | sed -n '460,790p'
nl -ba hephaestus/automation/implementer_phase_runner.py | sed -n '140,190p'
rg -n 'patch\.object\((impl|implementer), "_' tests/unit/automation
rg -n 'impl\._[a-zA-Z0-9_]+|implementer\._[a-zA-Z0-9_]+' hephaestus/automation tests/unit/automation

# After the refactor, prove removed wrappers are gone but legacy patch paths still work.
rg -n '^    def (_parse_follow_up_items|_collect_diff|_run_codex_code|_run_claude_impl_session)\(' hephaestus/automation/implementer.py
pixi run pytest tests/unit/automation/test_implementer.py -q
pixi run pytest tests/unit/automation/test_implementer_loop.py -q
pixi run pytest tests/unit/automation -q
pixi run ruff check hephaestus/automation/implementer.py tests/unit/automation/test_implementer.py
pixi run mypy hephaestus/automation
```

### Detailed Steps

1. **Re-verify the wrapper inventory from the live checkout.**
   Treat plan line numbers and grep counts as snapshot evidence. Re-read `implementer.py`, `implementer_phase_runner.py`, and the phase modules before editing. Anchor the edit by method names and section comments, not by absolute line numbers from the plan.

2. **Classify each candidate method as a truly mechanical pass-through.**
   A method is safe to move behind `__getattr__` only if it has no local state mutation, local logging, exception translation, type adaptation, metric emission, retry behavior, or semantic branch. Anything with orchestration meaning should stay as an explicit wrapper.

3. **Keep dynamic delegation allowlisted and strict.**
   Define a fixed `_PHASE_RUNNER_DELEGATES` set. `__getattr__` should only delegate names in that set and should raise normal `AttributeError` for every unknown attribute. Do not introduce `__getattribute__`, blanket `getattr(self.phase_runner, name)`, or a proxy that exposes the whole runner surface.

4. **Prove `patch.object` compatibility directly.**
   Legacy tests that do `patch.object(impl, "_collect_diff", ...)` depend on Python attribute lookup returning a patchable bound method. Add or preserve tests that patch at least one dynamically delegated method on the `IssueImplementer` instance and verify the patched object is the one called by cross-phase code.

5. **Review cross-phase callbacks before deleting wrappers.**
   The plan called out callbacks that still pass through `impl._method` from `implementer_phase_runner.py`, `_pr_create_phase.py`, `_review_phase.py`, and `_followup_phase.py`. Re-run a repo-wide `rg` for every delegated name, because those references are the real compatibility contract.

6. **Add collaborator properties as a future-facing seam, not a behavior change.**
   Read-only properties for `plan_phase`, `implement_phase`, `review_phase`, `pr_create_phase`, and `followup_phase` should return the exact objects owned by `phase_runner`. Add identity tests and avoid setters unless a test or production caller already needs mutation.

7. **Check typing and documentation claims against dynamic behavior.**
   `__getattr__` can hide problems from static analyzers and readers. Keep docstrings precise: explicit wrappers are orchestration seams; `_PHASE_RUNNER_DELEGATES` is a compatibility bridge for mechanical leaf methods. Add `TYPE_CHECKING` imports only where they improve property return types without causing runtime cycles.

## Verified Workflow

This skill is intentionally unverified. The actual workflow is titled `## Proposed Workflow` above.
This placeholder exists because the current ProjectMnemosyne validator requires a literal
`## Verified Workflow` section for every skill file.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Trust plan line numbers as current facts | Plan cited wrapper block `implementer.py:496-765`, runner collaborators at `implementer_phase_runner.py:163-167`, and phase callback locations | These are snapshot coordinates from the planning session; any nearby edit shifts them before implementation | Re-run `nl -ba` or `rg` in the current checkout and anchor edits by method names, not by stale line numbers |
| Treat listed methods as safe purely because they look like leaf delegations | Plan proposed moving methods such as `_collect_diff`, `_run_codex_code`, and `_run_claude_impl_session` behind `__getattr__` without executing the implementation | A wrapper can carry hidden semantics through local state, logging, exception shape, method identity, or test patching assumptions | Read each wrapper body and all call sites before deletion; only pure pass-through methods belong in the delegate allowlist |
| Use a blanket proxy to reduce boilerplate faster | Implement `__getattr__` as `return getattr(self.phase_runner, name)` for any missing attribute | Unknown attributes become silently accepted when the runner happens to have them, expanding the public surface and hiding typos | Use a closed `_PHASE_RUNNER_DELEGATES` set and keep unknown attribute rejection strict |
| Assume `patch.object(impl, "_method", ...)` works with dynamic attributes | Rely on Python intuition instead of adding a regression test | `patch.object` has specific lookup and assignment behavior on instances; bound-method identity and cross-phase callback lookup can regress silently | Add an explicit test that patches a dynamically delegated method on the instance and proves the patched callable is used |
| Trust issue-provided skill provenance | Plan relied on `dry-refactoring-workflow` v1.9.0 from the issue context | The source/version was not independently verified in the capture session | Treat the cited skill as context until re-read from the current ProjectMnemosyne checkout |

## Results & Parameters

### Most uncertain assumptions

- The methods moved to `__getattr__` are pure mechanical pass-throughs with no local behavior, local state use, or behavior-bearing docstring/type contract.
- No production or test code depends on method identity, direct class attribute presence, `dir()`, `hasattr()` side effects, autospeccing, or static type discovery for the removed wrappers.
- Cross-phase callbacks that still call `impl._method` will resolve through `__getattr__` at the right time and remain patchable on the `IssueImplementer` instance.
- Read-only phase properties expose exactly the existing `ImplementationPhaseRunner` collaborators and do not invite mutable replacement that the runner never expected.
- The issue-provided `dry-refactoring-workflow` v1.9.0 guidance is accurate, but it was not independently verified during the planning-capture session.

### External sources and snapshot evidence to re-verify

| Source or claim | Why it matters | Re-verify with |
|-----------------|----------------|----------------|
| `hephaestus/automation/implementer.py:496-765` wrapper block | Defines the methods being deleted or kept | `nl -ba hephaestus/automation/implementer.py \| sed -n '460,790p'` |
| `implementer_phase_runner.py:163-167` owns phase collaborators | Supports adding read-only properties on `IssueImplementer` | `nl -ba hephaestus/automation/implementer_phase_runner.py \| sed -n '140,190p'` |
| `rg 'patch\.object\(impl\|implementer, "_' tests/unit/automation` output | Defines legacy patch seams that must keep working | Re-run the grep in the live checkout before and after the refactor |
| Cross-phase callbacks in phase modules | These calls prove `impl._method` remains a runtime contract | `rg -n 'impl\._|implementer\._' hephaestus/automation tests/unit/automation` |
| Issue-provided `dry-refactoring-workflow` v1.9.0 | Planning guidance was cited but not re-read in this capture session | Re-open `skills/dry-refactoring-workflow.md` in ProjectMnemosyne before implementation |

### Reviewer focus

```text
- Is `_PHASE_RUNNER_DELEGATES` a closed allowlist, with strict AttributeError for unknown names?
- Did each delegated method lose only a pure pass-through wrapper and no local behavior?
- Do legacy `patch.object(impl, "_method", ...)` tests still patch the path actually called?
- Do read-only phase properties return identity-equal runner collaborators?
- Are docstrings and type hints honest about dynamic delegation and static typing limitations?
- Did the implementer re-run line-number and grep evidence in the current checkout?
```

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning capture for GitHub issue #1389, `IssueImplementer` delegation-surface refactor | Unverified planning artifact only; no code implementation, no local tests, and no CI were observed in this session. |
