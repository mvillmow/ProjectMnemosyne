---
name: architecture-agent-timeout-constant-planning-review-risks
description: "Planning-review checklist for centralizing agent/pipeline subprocess timeout literals into named constants while preserving import boundaries, env override semantics, and public test seams. Use when: (1) a plan moves timeout defaults into a canonical constants module, (2) runtime/automation import direction could create cycles, (3) old helper modules re-export constants for compatibility, (4) public aliases or tests rely on existing timeout names, (5) smaller subprocess budgets must not be confused with long-running phase budgets."
category: architecture
date: 2026-06-26
version: "1.0.0"
user-invocable: false
verification: unverified
tags: [planning, timeout-constants, subprocess-timeouts, import-boundaries, env-overrides, public-seams, reviewer-risks, hephaestus]
---

# Agent Timeout Constant Planning Review Risks

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Preserve durable planning review lessons from a ProjectHephaestus issue #1415 implementation plan that proposed centralizing agent/pipeline subprocess timeout literals into named env-overridable constants while preserving import boundaries and existing public test seams. |
| **Outcome** | Planning artifact only. The plan was not executed end-to-end when captured; this skill records assumptions and reviewer risks to verify before approving or implementing a similar timeout-constant sweep. |
| **Verification** | unverified — no code was applied, no tests were run, and CI was not confirmed for this workflow. |

## When to Use

- A plan centralizes timeout literals from agent/runtime or automation subprocess call sites into named constants.
- The proposed canonical module is chosen because lower-level runtime code needs the constants and importing an automation module would invert package boundaries.
- An existing automation-facing timeout helper module will re-export canonical constants for compatibility.
- Existing tests or callers import a public timeout seam that must remain as an alias, even if the canonical name changes.
- A broad grep or regex replacement could accidentally touch unrelated timeout semantics.
- Reviewers need a checklist for unverified assumptions in a planning-only timeout constant refactor.

## Proposed Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat every step as a planning hypothesis until the target repository is re-grepped, imports are smoke-tested, focused tests pass, and CI confirms the change.

### Quick Reference

```bash
# 1. Re-derive every literal site at implementation time; do not trust plan line numbers.
rg -n "timeout=(10|30|60|120|300|600|1800|2400)\\b|TimeoutExpired|_CLAUDE_IMPL_TIMEOUT" hephaestus tests

# 2. Prove the canonical module does not invert import boundaries.
python - <<'PY'
import hephaestus.constants
import hephaestus.agents.runtime
import hephaestus.automation.claude_timeouts
print("import OK")
PY

# 3. Prove compatibility re-exports and public seams still resolve.
python - <<'PY'
from hephaestus.constants import AGENT_IMPL_TIMEOUT
from hephaestus.automation.claude_timeouts import _CLAUDE_IMPL_TIMEOUT
assert _CLAUDE_IMPL_TIMEOUT == AGENT_IMPL_TIMEOUT == 1800
print("alias OK")
PY

# 4. Keep long-running phase budgets distinct from small subprocess defaults.
rg -n "7200|planner_claude_timeout|implementer_claude_timeout|review.*timeout" hephaestus/automation

# 5. Run focused tests that cover public aliases, runtime auth status, and timeout call sites.
pytest -q \
  tests/unit/automation/test_implementer.py \
  tests/unit/automation/test_planner.py \
  tests/unit/agents/test_runtime.py \
  tests/unit/github/test_tidy.py \
  tests/unit/github/test_fleet_sync.py
```

### Detailed Steps

1. **Choose the canonical module by import direction, not by convenience.** If lower-level runtime code such as `hephaestus/agents/runtime.py` needs `AGENT_AUTH_STATUS_TIMEOUT`, placing canonical values in `hephaestus/automation/claude_timeouts.py` can invert the boundary because automation already imports runtime. A lower-level module such as `hephaestus/constants.py` is plausible, but this is an assumption until import smoke tests prove no cycle.

2. **Use re-exports for automation-facing compatibility.** If automation modules already import `hephaestus.automation.claude_timeouts`, keep that module as the compatibility facade and re-export canonical constants from there. The re-export must not become a second source of truth or an independently configurable override path.

3. **Separate small subprocess budgets from long-running phase budgets.** Do not collapse existing 7200-second helper functions into smaller subprocess timeout constants. The helper functions represent long-running agent phase budgets; the literals being centralized are shorter subprocess guardrails such as auth status, git, diff, clone, review, implement, and rebase subprocess calls.

4. **Preserve public test seams as aliases.** If tests import a legacy seam such as `implementer._CLAUDE_IMPL_TIMEOUT` and assert the 1800-second value, keep it as an alias to the canonical `AGENT_IMPL_TIMEOUT`. Do not make the old seam independently configurable, or compatibility turns into drift.

5. **Replace by semantic call site, not broad numeric regex.** The literal values are common numbers. A raw replacement of every `timeout=300` or `600` can rewrite unrelated retry, validation, rate-limit, smoke-test, or CLI behavior. Scope the sweep to the issue-listed agent/pipeline subprocess sites plus any explicitly included invoker module, then inspect exclusions by hand.

6. **Reload-aware tests are required if env parsing happens at import time.** Env-overridable constants evaluated during import need tests that patch the environment and reload the defining module. If the plan relies on import-time parsing but tests only import once, env override behavior can be false-green.

## Verified Workflow

_Not applicable._ This skill is `unverified`: it captures plan-review heuristics and must not be treated as an executed workflow. The actionable hypothesis-level checklist is under **Proposed Workflow** above.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Put canonical timeout constants in the automation timeout helper because most current callers are automation modules | This is tempting when `claude_timeouts.py` already exists, but runtime code also needs an auth-status timeout and automation imports runtime in many places | Importing automation from runtime can invert the dependency direction or create an import cycle | Choose the canonical location by dependency direction first; use import smoke tests to prove the boundary. |
| Replace all matching timeout literals by number | A plan can map `10, 30, 60, 120, 300, 600, 1800, 2400` to constants and then perform a broad regex sweep | Those numbers can appear in unrelated resilience, validation, GitHub rate-limit, CLI smoke-test, or retry semantics | Replace by reviewed call-site intent, not just numeric equality. Keep an explicit out-of-scope list and grep it after editing. |
| Collapse 7200-second helper functions into the new subprocess timeout constants | "Centralize timeouts" can sound like one namespace for every timeout-like value | The 7200-second helpers are long-running phase budgets, not the smaller subprocess guardrails being named | Preserve phase-budget helpers separately unless the issue explicitly asks to redesign phase timeout semantics. |
| Delete the legacy `_CLAUDE_IMPL_TIMEOUT` seam after adding `AGENT_IMPL_TIMEOUT` | The new canonical constant appears to make the old private name redundant | Existing tests and possibly callers import the old seam and assert the 1800-second value | Keep the old name as an alias to the canonical constant; prove equality in tests so it cannot drift. |
| Trust line numbers and test lists captured during planning | The plan cited exact current line numbers and named focused tests without executing them | The repository can change between planning and implementation, and the test list may omit hidden call sites or wrapped keyword arguments | Re-grep current `origin/main` before editing, re-locate by symbol/argument, and treat the planning test list as a starting point only. |

## Results & Parameters

### Constants Mapping From the Captured Plan

| Timeout | Proposed Name | Intended Scope |
|---------|---------------|----------------|
| 10 seconds | `AGENT_AUTH_STATUS_TIMEOUT` | Auth/status subprocess checks in agent runtime. |
| 30 seconds | `AGENT_GIT_TIMEOUT` | Lightweight git subprocess calls. |
| 60 seconds | `AGENT_DIFF_TIMEOUT` | Diff-producing subprocess calls. |
| 120 seconds | `AGENT_CLONE_TIMEOUT` | Clone/setup subprocess calls. |
| 300 seconds | `AGENT_DEFAULT_TIMEOUT`, `AGENT_PLAN_TIMEOUT`, `AGENT_LEARN_TIMEOUT` | Default, plan, and learn subprocess budgets where the plan's scoped sites require this value. |
| 600 seconds | `AGENT_REVIEW_TIMEOUT`, `AGENT_PRE_PR_TEST_TIMEOUT` | Review and pre-PR-test subprocess budgets. |
| 1800 seconds | `AGENT_IMPL_TIMEOUT` | Implementation subprocess budget; legacy `_CLAUDE_IMPL_TIMEOUT` must alias this value. |
| 2400 seconds | `AGENT_REBASE_TIMEOUT` | Rebase subprocess budget. |

### Unverified External Inputs From the Captured Plan

- Issue #1415 wording and acceptance criteria.
- Exact line numbers from planning-time `rg` output.
- Import-boundary reasoning inferred from repository structure rather than proven by executed import smoke tests.
- Existing focused tests in `tests/unit/automation/test_implementer.py`, `tests/unit/automation/test_planner.py`, `tests/unit/agents/test_runtime.py`, `tests/unit/github/test_tidy.py`, and `tests/unit/github/test_fleet_sync.py`; these were referenced but not run during planning.

### Reviewer Focus Checklist

- Verify `hephaestus/constants.py` is actually the right canonical boundary and does not create an import cycle.
- Verify `hephaestus/automation/claude_timeouts.py` only re-exports canonical values and does not duplicate env parsing.
- Verify the 7200-second long-running phase helpers remain separate from smaller subprocess constants.
- Verify `_CLAUDE_IMPL_TIMEOUT` remains an alias to `AGENT_IMPL_TIMEOUT == 1800` for compatibility.
- Verify replacements are limited to issue-listed agent/pipeline subprocess sites plus any explicitly scoped invoker module.
- Verify unrelated timeouts in resilience, validation, GitHub rate limiting, and CLI smoke tests were not changed.
- Verify env override tests reload the defining module if constants parse env at import time.
- Verify grep-based anti-drift tests do not overmatch formatting or miss wrapped keyword arguments.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning for GitHub issue #1415 | Planning-only capture. No implementation, focused tests, full suite, or CI was executed at the time of learning. |
