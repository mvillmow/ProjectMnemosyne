---
name: planning-review-risk-capture
description: "Capture the weakest assumptions in an implementation plan before review: issue-title/body mismatches, stale grep evidence, unverified paths/line numbers, unchecked dependency direction, compatibility aliases that may only reflect test inertia, env-var/default contracts, import-time side effects, and external files/APIs relied on without direct verification. Use when: (1) reviewing or authoring a plan produced before implementation, (2) the plan cites current grep/audit output that can drift, (3) the plan chooses a source-of-truth location for constants/config/API behavior, (4) the plan preserves a compatibility shim or public alias based on an asserted test contract, (5) reviewers need an explicit list of high-risk assumptions to re-check before GO."
category: architecture
date: 2026-06-26
version: "1.0.0"
user-invocable: false
verification: unverified
tags:
  - planning
  - plan-review
  - risk-capture
  - unverified-assumptions
  - reviewer-focus
  - source-of-truth
  - grep-drift
  - line-number-drift
  - compatibility-contract
  - env-vars
  - import-time-side-effects
---

# Planning Review Risk Capture

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Turn an implementation plan's weakest assumptions into a concise reviewer checklist before any code is written. |
| **Outcome** | Planning-stage skill captured from ProjectHephaestus issue #1417; no implementation was executed and no CI was observed. |
| **Verification** | unverified |
| **Category** | Architecture / Planning review |

This skill is about the review surface of a plan, not the implementation mechanics.
Use it to separate facts the planner directly verified from claims the implementer or
reviewer must re-check against the current repository, GitHub issue, tests, and APIs.

## When to Use

- Reviewing or authoring a pre-implementation plan that cites exact file paths, line
  numbers, grep output, current tests, or GitHub issue text.
- The issue title and body point in different directions, and the plan chooses one as
  the source of truth.
- The plan centralizes constants, config defaults, env-var behavior, or API contracts
  and must choose where the canonical definition lives.
- The plan proposes compatibility aliases, re-exports, or public symbols because a
  test allegedly asserts them.
- The plan moves logic across package boundaries where dependency direction or
  import-time side effects can become the real risk.
- A reviewer needs a short list of the most uncertain assumptions to re-run before a
  GO verdict.

## Proposed Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms.

### Quick Reference

```text
For every implementation plan, extract three review lists:

1. Most uncertain assumptions
   - Issue title/body mismatches
   - Grep/audit evidence that can drift
   - Architecture/dependency-direction claims not executed
   - Compatibility contracts inferred from tests

2. External/current-state claims not directly verified in the plan
   - Exact paths and line numbers
   - GitHub issue body/title and linked context
   - Existing tests and asserted public surface
   - Env-var names, default values, parser behavior, and logging behavior

3. Reviewer focus risks
   - Naming/default compatibility
   - Import-time vs call-time behavior changes
   - Reload/env-override tests that mutate module constants
   - Grep scope missing live literals outside listed files
   - Non-happy-path input handling
```

### Detailed Steps

1. **Confirm the issue source of truth before accepting the plan's objective.**
   If a title says one thing and the body says another, call that out as a first-order
   review risk. The plan may be right to follow the body, but a reviewer should verify
   the mismatch against the live issue before implementation begins.

2. **Treat grep/audit output as point-in-time evidence, not a settled fact.**
   Any plan that says "current grep shows only these sites" should include the command,
   scope, and a reviewer instruction to re-run it. Line numbers, matches, generated
   files, and newly-merged sibling changes drift quickly.

3. **List every external or current-state claim that the plan did not directly verify.**
   This includes exact paths and line numbers, issue text, existing tests and their
   assertions, public module surface, env-var naming/default contracts, and API behavior
   inferred from docs or memory. Mark them as assumptions, not facts.

4. **Check source-of-truth placement and dependency direction explicitly.**
   When a plan creates shared constants or helpers, ask whether the proposed module can
   be imported by all consumers without reversing package boundaries or triggering
   import-time side effects. Prefer a neutral package layer when child modules would
   otherwise import automation/runtime code just to read a default.

5. **Separate compatibility contract from test inertia.**
   A plan may preserve a legacy alias or re-export because a test allegedly asserts the
   symbol exists. Reviewers should verify whether this is a real user-facing contract
   or only a brittle test pin that should be updated.

6. **Look for import-time behavior changes in env/config centralization plans.**
   Moving a call-time timeout helper into a module-level constant can freeze environment
   values at import, break reload tests, change invalid-env logging timing, or remove
   per-call override behavior. The plan must distinguish constants that are safe to
   import once from helpers that must stay lazy.

7. **End the review with a focused risk checklist.**
   The goal is not a long caveat dump. Produce a short set of checks a reviewer can run
   or inspect before approving: re-run the grep, verify public aliases, inspect import
   graph direction, exercise env override/reload behavior, and confirm invalid input
   handling.

## Verified Workflow

Not applicable. This skill was captured from a planning artifact and is
`unverified`: no implementation ran, no tests ran, and no CI result was observed. The
actionable process lives under **Proposed Workflow** above and must be treated as
unvalidated until a reviewer applies it and verifies the resulting plan/code.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Follow the issue title without reconciling the body | A plan could optimize for the title's wording even when the body describes a different task | The implementation may solve the wrong problem while appearing aligned to the issue header | Treat title/body mismatch as a reviewer risk; verify the live issue and explicitly state which source controls |
| Treat a grep audit as permanent | The plan used current grep output to decide which hardcoded values are live and which issue evidence is stale | New commits, generated files, line-number shifts, or too-narrow scope can invalidate the conclusion | Include the exact grep command and scope; reviewer re-runs it before GO |
| Move defaults into a convenient module without checking import direction | Shared constants were placed where consumers could import them, but dependency direction and import-time side effects were only reasoned about | A low-level module can end up importing automation/runtime code, or env parsing can happen earlier than intended | Put constants in a neutral layer and verify imports execute cleanly |
| Preserve an alias because a test allegedly pins it | A backward-compatible alias was kept based on a claimed test assertion | The alias may be real public surface, or it may be stale test inertia that should not shape the API | Verify the test and any documented/public consumers before treating the alias as a compatibility contract |
| Convert lazy timeout helpers into import-time constants indiscriminately | A plan centralizes hardcoded defaults but accidentally changes call-time env override behavior | Reload/env-override tests, invalid integer handling, and logging timing can change silently | Classify each helper: safe constant vs must remain lazy; test both normal defaults and env override/reload behavior |

## Results & Parameters

### ProjectHephaestus issue #1417 planning capture

The plan objective was to centralize live hardcoded subprocess timeout defaults into
named env-tunable constants while preserving existing long-running automation-loop
timeout helpers.

### Most uncertain assumptions reviewers should verify

- The issue title mentions patch fixtures, but the body is about timeout
  centralization. Verify the mismatch against the live issue before implementation.
- The plan relies on a current grep/audit command to decide which evidence is stale
  and which hardcoded values are live. Re-run it because line numbers and matches can
  drift.
- Shared constants were proposed for `hephaestus/constants.py`, with re-exports via
  `hephaestus/automation/claude_timeouts.py` to avoid GitHub modules importing
  automation. Check dependency direction and import-time side effects.
- `_CLAUDE_IMPL_TIMEOUT` was kept as a backward-compatible alias because a test
  allegedly asserts that public module surface. Verify whether that compatibility
  contract is real or just test inertia.

### External/current-state claims relied on without direct verification

- Exact file paths and line numbers.
- GitHub issue #1417 body/title.
- Existing tests and the assertions they make about public module surface.
- The proposed env-var naming/default contract.
- Non-integer env handling and logging behavior in the proposed constants module.

### Reviewer risk-flags for this plan shape

- Env-var naming compatibility and default value parity.
- Tests that mutate import-time constants with reload/env overrides.
- Accidental conversion of call-time timeout helpers to import-time constants.
- Stale grep scope missing timeout literals outside the listed files.
- Non-integer env handling/logging behavior in the constants module.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #1417 implementation-plan review capture | unverified; plan only, no code executed and no CI observed |
