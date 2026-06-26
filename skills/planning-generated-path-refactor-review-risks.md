---
name: planning-generated-path-refactor-review-risks
description: "Capture planning and review risks before refactoring repeated generated filename/path construction patterns into a shared helper. Use when: (1) a plan consolidates repeated issue-scoped log/artifact paths, (2) the call-site inventory is grep-derived and may miss split-line or prebuilt-variable forms, (3) filename preservation depends on dynamic prefixes/suffixes, (4) reviewers need a checklist for inventory drift, import-cycle risk, and behavior-preserving migration tests."
category: architecture
date: 2026-06-26
version: "1.0.0"
user-invocable: false
verification: unverified
tags: [planning, refactoring, generated-paths, log-paths, call-site-inventory, grep, ast-audit, reviewer-risks, automation]
---

# Generated Path Refactor Planning Review Risks

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Preserve the planning-review checklist for consolidating repeated generated path construction patterns, especially issue-scoped automation log/artifact filenames, into one shared helper without silently changing filenames or missing bypasses. |
| **Outcome** | Planning capture only. The implementation was not executed end-to-end, so every workflow item below is a reviewer/implementer hypothesis until verified against the current repository. |
| **Verification** | unverified — plan artifact only; no migration, tests, lint, CI, or AST audit was run end-to-end |
| **History** | v1.0.0: initial reusable capture from a ProjectHephaestus plan-review session. |

## When to Use

- A plan proposes a shared helper for repeated generated `.log`, artifact, cache, or report path construction.
- The proposed migration is driven by grep output rather than a full AST-backed inventory.
- The repeated filenames encode issue IDs, iteration numbers, dynamic agent names, parse-error suffixes, or other public-ish operational names that must remain byte-for-byte compatible.
- The plan cites issue text, affected-file lists, or exact line numbers that were not directly re-read during the current planning turn.
- Reviewers need to know what to inspect before implementation, rather than only whether the helper function shape looks reasonable.

## Verified Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms.

### Proposed Workflow (UNVERIFIED — planning artifact only)

Use this as a pre-implementation planning and review checklist, not as proof that a migration is safe.

### Quick Reference

```bash
# Replace these with the repo-specific roots and helper/module names.
rg -n '\.log"|\.log'\''|Path\(.*\.log|/.*\.log|issue.*log|log.*issue' hephaestus tests scripts
rg -n 'address-|parse-error|feedback|iteration|issue_id|issue_number' hephaestus tests scripts

# After migration, run a stricter bypass audit and manually inspect every dynamic case.
python scripts/audit_generated_path_bypasses.py
rg -n 'Path\(.*\.log|f".*\.log|f'\''.*\.log|\.with_suffix|/.*\.log' hephaestus tests scripts

# Prove import placement by execution, not reasoning.
python -c "from hephaestus.automation import _review_utils; print('import OK')"

# Targeted behavior tests should assert exact filenames, not only helper return types.
pixi run pytest tests/unit/automation -q
pixi run ruff check hephaestus tests
```

### Detailed Steps

1. **Separate verified inventory from grep-derived inventory.**
   In the plan, label every call-site list with its source. "Found by `rg`" is useful but not complete:
   grep can miss split-line path construction, helper-prebuilt stems, wrapper variables, format calls, or filenames assembled away from the final `Path(...) / name` expression. State which files were actually opened and which were only inferred from search output.

2. **Treat issue text and line numbers as drift-prone inputs.**
   If the issue body lists affected files or a count, verify that list against disk in the current turn before relying on it. If you did not re-read it, mark it as an assumption. If the plan cites exact lines, say they are pre-edit snapshots and require re-derivation by function name or stable marker after each edit.

3. **Preserve filenames as behavior, not formatting.**
   Tests and reviewer checks should assert complete generated names for every variant: ordinary logs, parse-error logs, iteration-specific logs, and dynamic agent/prefix forms. A helper that "looks equivalent" can still break operators if `address-123.parse-error.log` becomes `address-123-parse-error.log`, or if iteration fields move.

4. **Call out dynamic prefixes and suffixes explicitly.**
   Dynamic pieces such as `f"{agent}-feedback"` may be intentionally safe and desirable, but they are not proven safe by a static helper signature alone. List each dynamic prefix/suffix case and require manual inspection after the AST or grep bypass audit.

5. **Check import-cycle risk before choosing the helper module.**
   A shared helper placed in a review utility module can be convenient, but any new import edge can create a cycle in automation code. The plan should include an executed import smoke test for the proposed module graph, not just a statement that the destination is "central."

6. **Do not hoist repeated assignments unless behavior is proven unchanged.**
   If a phase currently creates separate log path variables near separate subprocess calls, replacing them with one shared assignment may subtly change logging behavior, naming, or overwrite timing. Flag this as a reviewer focus item and add tests or direct code inspection for each call path.

7. **Run the bypass audit after migration and inspect the survivors.**
   The AST audit is a guardrail, not a verdict. It should find remaining direct generated-path construction outside the helper; reviewers must then classify survivors as intentional bypasses, false positives, or missed migrations.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Treat grep inventory as complete | Planned the migration from `rg` hits for issue-scoped `.log` paths | Grep can miss split-line strings, variables that hold stems, helper wrappers, and dynamic composition that does not contain the final literal in one place | Label the inventory as grep-derived until an AST audit and manual inspection prove coverage |
| Trust the issue's affected-file list and count | Used the issue's named files/count as the plan scope | Issue bodies drift; they may omit files that match the behavior or include stale files that no longer contain the pattern | Reconcile the issue list with current disk state before writing scope-sensitive migration steps |
| Test only the helper shape | Added helper tests for parameters but did not assert migrated call-site filenames | The helper can pass while migrated callers still change parse-error, iteration, or dynamic-prefix filenames | Add behavior tests or explicit assertions for complete generated filenames at representative call sites |
| Assume suffix normalization is harmless | Proposed suffix handling such as `suffix.removeprefix('.')` without enumerating every expected suffix | Normalization may preserve common cases but still change operational filenames if suffix semantics are misunderstood | List every suffix value and expected final filename before accepting normalization logic |
| Move the helper to a convenient utility module without import smoke tests | Chose a central review utility destination by architecture intuition | Automation modules often import each other; a new helper import can create a cycle that static reading misses | Prove the import graph with an executable smoke test before claiming the placement is safe |

## Results & Parameters

### Reviewer Focus Checklist

```text
Generated path refactor review checklist

- [ ] Which call-site inventory is grep-derived, and which files were directly read?
- [ ] Were issue-body file lists, counts, and cited lines re-verified in this turn?
- [ ] Does the migration preserve exact filenames for parse-error and iteration variants?
- [ ] Are dynamic prefixes/suffixes listed and manually inspected after the audit?
- [ ] Does the helper module introduce any import cycle? Is there an import smoke test?
- [ ] Did any phase hoist or reuse a log path variable in a way that could change overwrite/timing behavior?
- [ ] Do tests cover migrated call-site behavior, not only helper parameter formatting?
- [ ] Was the post-migration bypass audit run, and were every survivor and false positive classified?
```

### Assumptions to Mark as Unverified Until Rechecked

| Assumption Type | Why It Is Risky | Required Review Action |
|-----------------|-----------------|------------------------|
| Issue affected-file list/count | Issue text may be stale or incomplete | Re-run search and read each affected file before implementation |
| Grep call-site inventory | Search patterns miss split-line/prebuilt-variable forms | Pair grep with AST audit and manual survivor inspection |
| Out-of-issue files in scope | A matching file may belong in scope even if not listed by the issue | Explain why it is behaviorally in scope or leave it explicitly out |
| Suffix normalization | Dots, parse-error names, and final extensions can shift | Enumerate input suffixes and expected final filenames |
| Dynamic prefix preservation | Dynamic agent/reviewer labels may be intentional behavior | List each dynamic case and preserve exact generated output |
| Assignment hoisting | Reusing one generated path can affect timing or overwrites | Inspect the phase behavior and add targeted tests where possible |
| Intentional bypasses | Some direct paths may intentionally stay outside the helper | Document every remaining bypass after audit |

### Concrete Example

In a ProjectHephaestus planning session for issue #1396, the proposed change was to add a shared
`log_file_path()` helper in `hephaestus/automation/_review_utils.py` and migrate issue-scoped
automation `.log` path constructions to it. The highest-risk assumptions were:

- the issue's affected-file list and stated count were accurate;
- the grep inventory caught every issue-scoped log path, including split-line and prebuilt-variable forms;
- an extra `follow_up.py` match belonged in scope even though the issue did not list it;
- `suffix="parse-error.log"` preserved existing `address-<issue>.parse-error.log` names;
- dynamic prefixes such as `f"{agent}-feedback"` remained safe and desirable;
- `_implement_phase` could reuse one Claude log path without changing logging behavior;
- no intentional issue-scoped log path bypass should remain outside the helper module.

External inputs relied on without direct verification in that planning turn included the issue body,
exact line numbers from `rg` output, command availability for pytest/ruff/pixi, proposed AST parser
behavior, `suffix.removeprefix(".")` semantics for every expected suffix value, and branch/CI behavior.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Planning/review capture for issue #1396, proposing a shared helper for issue-scoped automation log paths | unverified — the implementation was not executed end-to-end; this records reviewer risks and assumptions to verify before approving or implementing |
