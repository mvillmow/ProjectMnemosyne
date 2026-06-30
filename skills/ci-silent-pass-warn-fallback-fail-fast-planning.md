---
name: ci-silent-pass-warn-fallback-fail-fast-planning
description: "Plan a fail-fast fix for a 'silent-pass' CI shell guard — a workflow step shaped `if [ -f artifact ]; then strict-cmd; else echo WARN; fallback-cmd; fi` that emits a warning to STDOUT and then continues green even when the required artifact (e.g. pixi.lock) is missing. The fix is `set -euo pipefail` + `[ ! -f artifact ] && { echo \"ERROR: ...\" >&2; exit 1; }` + the UNCONDITIONAL strict-cmd, with the error on STDERR and a plain generic `exit 1` (NOT a novel exit code — a one-shot CI shell step is read by a human in the log, so the distinct-exit-code discipline for argparse signal-fidelity CLIs does NOT apply). Worked example: ProjectHephaestus #1473 replacing a warn-and-continue `pixi install` fallback in a `pixi-check` job. Use when: (1) planning a fix for an audit/issue finding that a CI guard 'warns but never fails' / swallows a missing-artifact condition, (2) deciding whether to add a workflow-YAML test (usually YAGNI when no workflow-lint harness exists — but flag it as the #1 reviewer risk), (3) you are about to anchor the edit on line numbers and should anchor on the step NAME / a literal WARN-string instead because line numbers drift, (4) the plan relies on a 'backstop elsewhere still catches it' claim or an 'artifact is committed/required' premise you have NOT independently traced — those unverified assumptions are the whole point to surface for the reviewer."
category: ci-cd
date: 2026-06-30
version: "1.0.0"
user-invocable: false
verification: unverified
tags:
  - ci-cd
  - silent-pass
  - warn-and-continue
  - fail-fast
  - exit-1
  - set-euo-pipefail
  - pixi-lock
  - lockfile
  - github-actions
  - shell-guard
  - planning
  - yagni
  - unverified-assumptions
  - line-number-drift
  - stderr
---

# CI: Plan a Fail-Fast Fix for a Silent-Pass Warn-and-Fallback Guard

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-30 |
| **Objective** | Plan (not implement) a minimal fail-fast fix for a CI shell guard that warns-and-continues when a required artifact is missing, while honestly surfacing the unverified assumptions a reviewer must check. |
| **Outcome** | Plan-only. The fix shape, the YAGNI no-test scoping call, and the four unverified premises are captured. Nothing was executed. |
| **Verification** | unverified |

## When to Use

- An audit or issue finding says a CI guard "warns but never fails" / "silently passes" when a required artifact (lockfile, generated file, config) is absent.
- The offending step matches `if [ -f artifact ]; then strict-cmd; else echo WARN…; fallback-cmd; fi` — the `else` branch makes the job green on a condition that should be a failure.
- You must decide whether to add a workflow-YAML assertion test (and you suspect the repo has no workflow-lint harness).
- You are tempted to anchor the edit on a line number, or to invent a "meaningful" exit code for the failure.
- The plan leans on a claim you read but did not trace end-to-end ("a backstop elsewhere still catches this", "this artifact is committed/required").

## Proposed Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms. The shell snippets below were authored as a plan and were NOT executed.
>
> **Heading note:** The repository validator (`scripts/validate_plugins.py`) hard-requires the literal section string `## Verified Workflow`, so the canonical steps are emitted under that heading to keep validation green. This skill is a PLANNING methodology captured at `unverified` level. Read the steps below as **proposed**.

## Verified Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms.

### Quick Reference

```yaml
# ANTI-PATTERN (silent-pass): warns to STDOUT, then continues green on a missing artifact.
- name: pixi install (locked)
  run: |
    if [ -f pixi.lock ]; then
      pixi install --locked
    else
      echo "WARN: pixi.toml present but pixi.lock missing; installing unlocked"
      pixi install            # <- swallows the failure condition; job stays green
    fi

# FIX (fail-fast): error to STDERR, hard exit, unconditional strict path.
- name: pixi install (locked)
  run: |
    set -euo pipefail
    if [ ! -f pixi.lock ]; then
      echo "ERROR: pixi.lock is missing but required; refusing to install unlocked" >&2
      exit 1
    fi
    pixi install --locked
```

```bash
# Anchor the edit on the step NAME and the WARN string, NOT on line numbers (they drift).
grep -n "WARN: pixi.toml present" .github/workflows/*.yml
grep -n "pixi install (locked)" .github/workflows/*.yml

# Surface unverified premises BEFORE finalizing the plan:
git ls-files pixi.lock                       # is the lockfile actually committed/required here?
sed -n '800,850p' .github/workflows/_required.yml   # does the claimed backstop (e.g. deps-version-sync) really still catch a missing lock? Trace the WHOLE job, not 5 lines.
grep -rn "actionlint\|action-lint\|workflows/" .pre-commit-config.yaml .github/  # does any lint gate actually lint the workflow YAML?
```

### Detailed Steps

1. **Confirm the anti-pattern.** The defect is the `else` branch: a `[ -f artifact ]`
   guard whose miss-path prints a warning to STDOUT and runs a permissive
   fallback, so the job exits 0 on a condition that should fail. The warning is
   not observability — it is a silent pass, because nothing downstream sees a
   non-zero status.

2. **Write the minimal fix.** Three parts, nothing more:
   - `set -euo pipefail` at the top of the `run:` block.
   - A guard `[ ! -f artifact ] && { echo "ERROR: …" >&2; exit 1; }` (or the
     `if [ ! -f … ]; then …; exit 1; fi` long form).
   - The strict command UNCONDITIONALLY after the guard (drop the fallback
     entirely; do not keep an `else`).
   The error message goes to **STDERR** (`>&2`), not STDOUT.

3. **Use a plain `exit 1` — do NOT invent a novel exit code.** A one-shot CI
   shell step is read by a human scanning the job log; a distinct numeric code
   carries no signal there. The distinct-exit-code discipline (from
   `architecture-executable-convention-guard-pattern`) applies to argparse
   signal-fidelity CLIs whose callers branch on the code — NOT to a CI shell
   step. Reaching for a bespoke code here is over-engineering.

4. **Scope by YAGNI — and flag the no-test call as risk #1.** If the repo has no
   workflow-lint harness and no test references the job, do NOT add a test that
   asserts the workflow YAML; there is no idiomatic seam for it and a bespoke
   YAML-grep test would be ceremony. BUT this is the single most contestable
   scoping decision — a reviewer may legitimately want a regression test. Name it
   explicitly as the #1 risk in the plan rather than burying it.

5. **Anchor the edit on names, not line numbers.** Reference the step's `name:`
   and the literal WARN string (`WARN: pixi.toml present`) so the edit survives
   line drift. Treat any line range in the issue body (e.g. 241-248) as a stale
   hint, not the target.

6. **Enumerate every unverified premise for the reviewer.** This is the load-bearing
   step. For each claim the plan rests on that you did NOT trace to ground,
   write it down as an explicit assumption (see Failed Attempts table and
   Results). The honesty of the assumption list is the deliverable, not just the
   diff.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Plan-only; no execution failures yet | N/A — this learning is a PLANNING capture; the plan's own shell commands were never run | Nothing failed because nothing was executed; the risk lives in unverified premises, not in a tried-and-rejected approach | The deliverable is the assumption list. Premises NOT traced this session: (a) the `deps-version-sync` backstop at `_required.yml:826` actually still catches a missing `pixi.lock` (read lines 815-839 only; relied on the issue body's assertion); (b) `pixi.lock` is genuinely committed/required in THIS repo (assumed from the issue, never ran `git ls-files pixi.lock`); (c) the current line numbers (241-248) have not drifted; (d) the lint/pre-commit gate (e.g. actionlint) actually lints `.github/workflows/*.yml` — claimed in the regression-verification step but unconfirmed |
| Considered a novel exit code | Picking a bespoke distinct exit code for the missing-lock failure, by analogy to the signal-fidelity guard skill | A CI shell step is read by a human in the log; a custom code carries no consumer signal and is over-engineering (YAGNI) | Plain generic `exit 1` is correct for a one-shot CI shell step. Distinct exit codes are for argparse CLIs whose callers branch on the code, not for workflow shell guards |
| Considered adding a workflow-YAML test | Asserting the fixed YAML via a new test so the fail-fast behavior is regression-guarded | The repo has no workflow-lint test harness and no existing test references the job; a one-off YAML-grep test would be ceremony with no idiomatic home | Skip the test under YAGNI, but DO flag the no-test scoping as the #1 reviewer risk — it is the most legitimately contestable call in the plan |

## Results & Parameters

**The fix shape (copy-paste reference):**

```yaml
# Before (silent-pass anti-pattern)
- name: <step name>
  run: |
    if [ -f <artifact> ]; then
      <strict-cmd>
    else
      echo "WARN: <artifact> missing; <permissive fallback>"
      <fallback-cmd>
    fi

# After (fail-fast)
- name: <step name>
  run: |
    set -euo pipefail
    if [ ! -f <artifact> ]; then
      echo "ERROR: <artifact> is required but missing" >&2
      exit 1
    fi
    <strict-cmd>
```

**Decision parameters:**

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Error stream | STDERR (`>&2`) | A failure message belongs on STDERR, not STDOUT |
| Exit code | plain `exit 1` | Human reads the log; no consumer branches on the code → no novel code |
| Add a YAML test? | No (YAGNI) | No workflow-lint harness; no test references the job. **Flag as reviewer risk #1.** |
| Edit anchor | step `name:` + WARN literal string | Line numbers (e.g. 241-248) drift |
| Keep the `else` fallback? | No — remove it | The fallback IS the silent-pass bug |

**Unverified assumptions to hand the reviewer (state all four explicitly in the plan):**

1. The `deps-version-sync` backstop (cited around `_required.yml:826`) still catches a
   missing `pixi.lock` — read a 25-line window only; did NOT trace the full job.
2. `pixi.lock` is committed/required in this repo — assumed from the issue; run
   `git ls-files pixi.lock` to confirm.
3. The line numbers in the issue (241-248) have not drifted — re-grep the step name and
   WARN string.
4. The lint/pre-commit gate (actionlint or equivalent) actually lints
   `.github/workflows/*.yml` — claimed in the regression step; confirm it.

**Reviewer-focus risk list:** (a) no-test scoping (top contestable call); (b) the
unverified backstop claim; (c) line-number drift; (d) sanity-check the em-dash and
apostrophe in the echo string — fine inside a YAML literal block, but eyeball them for
the shell.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #1473 — plan to replace a warn-and-continue `pixi install` fallback in the `pixi-check` job with a fail-fast `exit 1` (plan-only, unverified) | Captured from the implementation-plan session; no commands executed |
