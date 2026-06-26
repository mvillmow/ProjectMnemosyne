---
name: parser-consolidation-review-risk-capture
description: "Capture reviewer risks before consolidating duplicated parser bodies into one canonical helper. Use when: (1) a plan replaces several fenced-JSON or LLM-output parsers with one shared implementation, (2) each caller has different default, fallback, trace, or warning behavior, (3) private tests patch wrapper functions or module-qualified helpers, (4) structural grep scans are used to prove duplicate parser removal, (5) the plan relies on drift-prone rg counts, line numbers, or private seam assumptions."
category: architecture
date: 2026-06-26
version: "1.0.0"
user-invocable: false
verification: unverified
tags: [parser-consolidation, json, refactoring, planning, review-risk, caller-contracts, trace-files, fallback-behavior, test-seams, structural-scan]
---

# Parser Consolidation Review Risk Capture

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-26 |
| **Objective** | Preserve the planning risks for consolidating duplicated fenced-JSON parser bodies into one canonical helper while keeping each caller's behavior stable. |
| **Outcome** | Plan captured; implementation was not executed in this learn session. Reviewer should treat all file paths, line numbers, rg counts, and private seam assumptions as drift-prone until re-run immediately before implementation. |
| **Verification** | unverified - planning artifact only; no parser implementation or tests were run in this session. |

## When to Use

- Reviewing or authoring a plan that collapses duplicate fenced-JSON, markdown-code-block, or LLM-output parser bodies into a single canonical helper.
- The callers have different contracts for default shape, parse-error default, first-vs-last block choice, raw JSON fallback, trace-file writing, warning/callback behavior, or scalar/non-object JSON handling.
- A plan removes a private parser function from one module but keeps thin adapters in others because tests or external workflows patch those seams.
- A plan uses a structural negative scan to prove no parser regex or `json.loads` variants remain outside the canonical helper.
- The implementation plan cites issue-plan line numbers, prior `rg` counts, or exact duplicate locations that were not re-verified in the current implementation turn.

## Proposed Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat it as a review checklist for an unexecuted plan until implementation tests and CI confirm the behavior.

### Quick Reference

```bash
# Re-run discovery immediately before implementation; do not trust stale plan counts.
rg -n "```json|json\\.loads|re\\.findall|re\\.search|parse_json_block|_parse_json_block" \
  hephaestus/automation tests/unit/automation

# After consolidation, run a structural negative scan. Review its blind spots manually.
python3 - <<'PY'
from pathlib import Path

allowed = {
    Path("hephaestus/automation/_review_utils.py"),
    Path("hephaestus/automation/audit_reviewer.py"),
}
patterns = ("```json", "json.loads", "re.findall", "re.search")
for path in Path("hephaestus/automation").glob("*.py"):
    if path in allowed:
        continue
    text = path.read_text()
    hits = [p for p in patterns if p in text]
    if hits:
        print(f"{path}: {hits}")
PY
```

### Detailed Steps

1. **Write every caller contract before merging implementations.**
   For each existing parser body, record the returned default shape, parse-error default, whether it uses the first or last fenced block, whether raw JSON is accepted, whether scalar or list JSON is valid, whether trace files are written, and how errors are reported.

2. **Make helper knobs explicit and backward-compatible.**
   A shared helper can reduce duplication with options such as `default`, `parse_error_default`, `trace_dir`, `trace_name`, `raw_json_fallback`, `use_last_block`, and `on_error`. The no-keyword path must preserve the old canonical behavior, and every adapter must pass only the knobs required for its caller contract.

3. **Preserve private seams deliberately.**
   Removing one wrapper can be safe if runtime calls switch to a module-qualified helper that tests can patch. Keeping other wrappers can be necessary when private tests or external workflows already call them. State which seams are deleted, which remain as thin adapters, and which patch target tests should use after the refactor.

4. **Treat structural negative scans as review aids, not proof.**
   A scan for parser regexes and `json.loads` variants is useful, but it can miss equivalent implementations using different APIs or helper names. It can also flag intentionally different parsers. Review every exclusion and explain why it is non-equivalent before accepting the scan as sufficient.

5. **Keep intentionally different parsers out of scope only with a contract reason.**
   A parser that consumes every audit block, skips malformed blocks, and returns `list[dict]` is not equivalent to a helper that returns one parsed object or a default. Excluding it is reasonable only when the plan records that behavioral difference.

## Verified Workflow

_Not applicable._ This skill was captured from a planning session and is `unverified`: the parser consolidation was not implemented, local tests were not run, and CI was not confirmed. Use the **Proposed Workflow** above as a hypothesis-level review checklist.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Trust stale discovery output | The plan relied on prior `rg` results naming duplicate parser bodies and line numbers without re-running them in the learn turn | Parser locations, wrappers, and tests can drift between plan authoring and implementation | Re-run discovery immediately before implementation and cite fresh file paths, counts, and line anchors |
| Merge parser bodies before writing caller contracts | Treated several fenced-JSON parsers as "the same" because each used regex extraction plus `json.loads` | Their observable behavior may differ in default shape, first-vs-last block choice, raw fallback, trace writing, warning text, and scalar handling | Write down each caller's contract first, then map each contract to explicit helper options or adapters |
| Delete private wrappers as obvious duplication | Removed module-private parser functions without checking whether tests, patch targets, or external workflows call them | Even redundant wrappers can be public enough to be compatibility seams in an agent automation codebase | Delete wrappers only when patch/import seams are intentionally moved; otherwise keep thin adapters |
| Treat a negative grep scan as complete proof | Planned a structural scan for remaining parser regex and `json.loads` variants | Equivalent parser logic can hide under different APIs, and intentionally different parsers can be mistaken for leftovers | Use the scan to find likely misses, then manually inspect exclusions and near-matches |
| Assume warning and trace payload compatibility is cosmetic | Collapsed error handling and trace writing into shared behavior without specifying payload shape or warning text expectations | Operators and tests may depend on trace filename, raw response payload, parsed/default object shape, or exact warning path | Capture trace schema and warning/callback behavior as part of the caller contract |

## Results & Parameters

### Caller Contract Matrix

Fill this out before implementation:

| Caller | Success Shape | Parse Error Shape | Block Choice | Raw JSON Fallback | Trace Behavior | Error/Warning Behavior | Seam to Preserve |
|--------|---------------|-------------------|--------------|-------------------|----------------|------------------------|------------------|
| PR review | `{"comments": ..., "summary": ...}` | caller-specific default | first or last, verify live | verify live | none or verify live | verify live | patch target after refactor |
| Address review | `{"addressed": ..., "replies": ...}` | caller-specific default | first or last, verify live | verify live | parse trace file | verify live | keep adapter if tests call it |
| CI driver | `{}` or parsed object | `{}` | first-block/raw fallback, verify live | yes, verify ordering | none or verify live | verify live | keep private adapter if tests call it |
| Intentional exclusion | `list[dict]` from every valid block | skip malformed blocks | all blocks | no, verify live | verify live | verify live | explain non-equivalence |

### Reviewer Focus

- Mutable/default-copy semantics: does the helper return a fresh copy for dict/list defaults, or can callers mutate shared defaults?
- Non-object JSON behavior: does scalar or list JSON get accepted, rejected, or replaced with the default in each caller?
- Trace compatibility: are trace filenames, directories, raw payload fields, and parsed/default payloads unchanged?
- Fallback ordering: does fenced-block parsing happen before raw JSON fallback, and does first-vs-last block behavior match each caller?
- Import and patch seams: were private wrappers deleted only where tests now patch the module-qualified canonical helper?
- Warning compatibility: is exact warning text observable, or is callback behavior enough?

### Issue #1385 Planning Capture

The captured ProjectHephaestus plan aimed to make `hephaestus/automation/_review_utils.py` own fenced-JSON regex extraction, `json.loads`, defaults, raw fallback, and optional trace-file writing. Intended stable caller behavior was:

- PR review returns `{"comments": ..., "summary": ...}`.
- Address review returns `{"addressed": ..., "replies": ...}` and keeps parse traces.
- CI driver returns `{}` with first-block and raw-JSON fallback behavior.
- `audit_reviewer._parse_coordinator_results` stays out of scope because it parses every audit block, skips malformed blocks, and returns `list[dict]`.

The plan proposed helper knobs `default`, `parse_error_default`, `trace_dir`, `trace_name`, `raw_json_fallback`, `use_last_block`, and `on_error`. It also proposed deleting `pr_reviewer._parse_json_block` while switching runtime calls to `hephaestus.automation._review_utils.parse_json_block`, and preserving `CIDriver._parse_json_block` plus `_parse_addressed_block` as thin adapters for existing private seams.

### Verification Level

`unverified`: this skill captures a plan, not an executed implementation. The reviewer should require fresh `rg`, focused pytest for the affected automation modules, a structural negative scan with manual inspection of exclusions, ruff on modified files, and broader automation unit tests before treating the consolidation as complete.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #1385 planning session for consolidating duplicated fenced-JSON parser bodies into `hephaestus/automation/_review_utils.py` | Plan produced, not executed; this skill records uncertain assumptions and reviewer focus areas. |
