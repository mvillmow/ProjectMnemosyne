---
name: hypothesis-property-fuzz-llm-output-parsers
description: "Procedure + pitfalls for PLANNING (and RE-PLANNING after a NOGO) Hypothesis property-based / fuzz tests for LLM-output string parsers in a pixi+pyproject dual-manifest Python repo. Use when: (1) adding property/fuzz tests for parsers that ingest free-form LLM output (verdict markers, ```json fences, bold/CRLF noise), (2) the goal is to assert a parser's FAIL-SAFE CONTRACT (never-raises + typed default) rather than parsed values, (3) you must add a new test/dev dependency (e.g. hypothesis) and need it in BOTH pyproject.toml [project.optional-dependencies].dev AND pixi.toml [feature.dev.dependencies] (decision rule: conda-forge package -> conda table, NOT pypi-dependencies; conda path is what `pixi run test` uses), (4) deciding whether a new dep triggers a floor-consistency guard, (5) composing st.text() with crafted fragments so Hypothesis reaches structured parser branches, (6) deciding to append a Test<Parser>Properties class vs create a new _property sibling test file, (7) RE-PLANNING after a NOGO in a TASK/PLAN/REVIEW pipeline where the reviewer sees ONLY the text you return -- never point at 'the plan above', re-emit the FULL plan, and promote unverified reliances to named risks. RELATED but distinct: llm-output-verdict-parse-last-line-not-substring (parser CORRECTNESS, not fuzz-test planning), dry-refactoring-workflow (test-structure-mirrors-source), and dependency-manifest-single-source-of-truth (manifest alignment)."
category: testing
date: 2026-06-30
version: "1.1.0"
user-invocable: false
verification: unverified
history: hypothesis-property-fuzz-llm-output-parsers.history
tags:
  - hypothesis
  - property-based-testing
  - fuzz-testing
  - llm-output-parsing
  - pixi
  - pyproject
  - dual-manifest
  - nogo-recovery
  - plan-review-pipeline
---

# Hypothesis Property/Fuzz Tests for LLM-Output Parsers (Planning)

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-30 |
| **Objective** | Plan (and re-plan, after a NOGO) Hypothesis property-based / fuzz tests for four LLM-output string parsers (`parse_review_verdict`, `_parse_coordinator_results`, `latest_verdict`, `_parse_addressed_block`) and add `hypothesis` to dev/test deps — ProjectHephaestus issue #1470. |
| **Outcome** | Plan only. No tests written, no `pixi install` run, no CI. Captured as a reusable planning + NOGO-recovery procedure plus a risk list. |
| **Verification** | unverified — the plan was never executed end-to-end. |
| **History** | v1.0.0 → v1.1.0 (this version): adds the self-review / meta-narrative NOGO trap, the conda-vs-pypi pixi-table decision rule, the floor-guard scope check, and the name-your-risks discipline. Prior snapshot in `hypothesis-property-fuzz-llm-output-parsers.history`. |

## When to Use

- You are adding property-based / fuzz tests (Hypothesis) for parsers that consume **free-form LLM output**: verdict markers, ```json code fences, bold markers, CRLF, multi-line verdict noise.
- The parser's job is to be **fail-safe**: never raise on garbage, always return a typed default. You want tests that assert that *contract*, not specific parsed values.
- You must add a new **test/dev dependency** (e.g. `hypothesis`) in a **pixi + pyproject dual-manifest** repo and must not leave one manifest behind.
- You are deciding whether a new dependency trips an existing floor/lockstep consistency guard.
- `st.text()` alone is not exercising the structured branches of the parser and you need to bias the generator.
- You are deciding where the new tests live (append a class vs. new sibling file).
- **You are RE-PLANNING after a NOGO** in a TASK/PLAN/REVIEW pipeline (the reviewer grades only the text you return). The plan body IS the reviewed artifact — see the meta-narrative trap below.

## Proposed Workflow

> **Warning:** This workflow has not been validated end-to-end. Treat as a hypothesis until CI confirms.

### Quick Reference

```bash
# 0. RE-PLANNING AFTER A NOGO (read this FIRST in a TASK/PLAN/REVIEW pipeline).
#    The reviewer sees ONLY the text you return -> the plan body IS the artifact.
#    DO re-emit the FULL plan markdown (every section, real file:line anchors, fenced
#       code, per-criterion runnable verification commands).
#    DON'T output "my plan above is correct/complete" or "as written above" -> that leaves
#       the reviewed artifact EMPTY -> guaranteed NOGO on every dimension.

# 1. Add the dep in BOTH manifests (conda path is what CI runs).
#    pyproject.toml  [project.optional-dependencies] dev   -> "hypothesis>=6.0,<7"   (pip path)
#    pixi.toml       [feature.dev.dependencies]  hypothesis = ">=6.0,<7"             (conda path)
#    DECISION RULE: on conda-forge? (hypothesis IS) -> [feature.dev.dependencies] (conda),
#                   NOT [feature.dev.pypi-dependencies]. Mirror pytest/pytest-cov
#                   (pyproject.toml:49-50 + pixi.toml:87-88).

# 2. RESOLVE the env BEFORE trusting the floor (this step was NEVER run during planning -> top risk).
pixi install            # may fail to solve >=6.0,<7 against pixi.lock; verify, don't assume.
                        # FALLBACK: if the solver fails, raise/relax the floor and re-run.

# 3. Check whether a floor-consistency guard even applies to the new dep.
grep -nE "PyGithub|pytest|mypy" tests/unit/scripts/test_dependency_floor_consistency.py
#    -> guard only enforces PyGithub/pytest/mypy floors; arbitrary new deps are NOT lockstep-checked.
#       So no new consistency test is forced — but keep floors matched across manifests for hygiene.

# 4. Derive each parser's FAIL-SAFE CONTRACT by READING source, then assert THAT (not parsed values).

# 5. Compose generators so Hypothesis reaches structured branches.
#    st.text() | crafted fragments (verdict tokens, ```json fences, **bold**, \r\n, multiple verdict lines)

# 6. Place tests: append Test<Parser>Properties to the parser's EXISTING test module;
#    create a new test_*_property.py sibling only when the existing module is large or absent.
#    (test-structure-mirrors-source -- cross-ref dry-refactoring-workflow.)

pixi run test           # property tests can be slow/flaky under --cov; use @settings(deadline=None)
                        # if the default Hypothesis deadline trips under coverage.

# 7. NAME unverified reliances as RISKS in the plan (do not hide them). A plan that lists
#    "floor unsolved against pixi.lock; pixi install never run; fallback = raise floor" reviews
#    BETTER than one that silently claims verification.
```

### Detailed Steps

1. **Re-planning after a NOGO: re-emit the FULL plan, never a self-affirmation (the meta-narrative trap).** In a TASK/PLAN/REVIEW pipeline the reviewer grades **only the text the agent returns** — the artifact IS the plan body. After doing the analysis in-context, it is tempting to output a paragraph like "my plan above is correct and complete" or "as written above" instead of re-emitting the plan. That leaves the reviewed artifact **empty**, which fails *every* grading dimension (requirements alignment, completeness, concreteness, verification) by construction — there is literally nothing to grade. This is the #455/#468/#484/#693 pattern. **Always re-emit the complete plan markdown** (all sections, real `file:line` anchors, fenced code, runnable per-criterion verification commands) — never a diff, summary, or affirmation of an earlier message.

2. **Dual-manifest dependency add — with the concrete pixi-table decision rule (the #1 footgun).** A new test dep must go in BOTH `pyproject.toml [project.optional-dependencies].dev` (pip install path) AND `pixi.toml [feature.dev.dependencies]` (conda path — what CI actually runs via `pixi run test`). **Decision rule for WHICH pixi table:** if the package is on **conda-forge** (`hypothesis` IS), use `[feature.dev.dependencies]` (conda), **not** `[feature.dev.pypi-dependencies]`. Mirror the existing `pytest` / `pytest-cov` convention (`pyproject.toml:49-50` + `pixi.toml:87-88`). Adding to only one manifest leaves either CI or pip-dev installs missing the dep.

3. **Check the floor-consistency guard scope BEFORE assuming lockstep is enforced.** `tests/unit/scripts/test_dependency_floor_consistency.py` only enforces floor parity for **PyGithub, pytest, and mypy** — not arbitrary new deps. So a new `hypothesis` dep forces **no** new consistency test, but match the floor (`>=6.0,<7`) across both manifests anyway for hygiene.

4. **Name unverified load-bearing steps as EXPLICIT RISKS — don't hide them.** A plan that honestly lists "the `hypothesis>=6.0,<7` floor was not solved against `pixi.lock`; `pixi install` was never run; raise the floor if the solver fails" **plus a fallback step** reviews BETTER than one that silently claims verification. The reviewer rewards a named-risk + mitigation over false confidence. Promote every unverified reliance (see Risks below) to a stated risk with a mitigation.

5. **Assert the parser's documented FAIL-SAFE CONTRACT, not parsed values.** Derive the contract by reading the parser source. The four contracts captured here:
   - `parse_review_verdict` → always returns a `ReviewVerdict`; missing marker → `verdict="AMBIGUOUS"`, `raw == input`. Never raises.
   - `latest_verdict` → returns `"GO"` / `"NOGO"` / `None`; **last-match-wins** (`re.findall(...)[-1]`). Cross-ref the `llm-output-verdict-parse-last-line-not-substring` skill for *why* last-match-wins matters.
   - `_parse_coordinator_results` → returns `list[dict]`; malformed JSON inside a ```json fence is **skipped, not raised**; prose with no fence → `[]`.
   - `_parse_addressed_block` → delegates to `_review_utils.parse_json_block` (`use_last_block` default `True`, read at `_review_utils.py:553-563`); always returns a `dict`; junk → default `{"addressed": [], "replies": {}}`.

6. **Pure `st.text()` rarely reaches the structured branches.** Compose `st.text()` with crafted fragments — verdict tokens, ```json fences, `**bold**` markers, CRLF, multiple verdict lines — so Hypothesis exercises BOTH the "never crashes on noise" property AND **metamorphic invariants** (e.g. appending a later verdict line flips `latest_verdict`'s result → demonstrates last-match-wins).

7. **Mirror test structure to source.** Append a `Test<Parser>Properties` class to the parser's EXISTING test module. Only create a new `_property` sibling file when the existing module is large or the parser lacks a dedicated test file (here `_parse_addressed_block` got a new `test_address_review_property.py`). This is the test-structure-mirrors-source discipline — cross-ref `dry-refactoring-workflow`.

8. **Resolve and run before claiming done.** Run `pixi install` (verify the floor actually solves against `pixi.lock`) and `pixi run test`. Watch for Hypothesis flakiness/slowness under `--cov`; the default `--deadline` may trip under coverage — mitigate with `@settings(deadline=None)` on the property tests.

## Verified Workflow

_Not applicable._ This skill was captured from a planning (and re-planning-after-NOGO) session and is `unverified`: the plan was never executed, no `pixi install` ran, and no tests/CI confirmed it. The actionable, hypothesis-level methodology lives under **Proposed Workflow** above and must be treated as unvalidated until CI confirms it. (This placeholder section exists only because `scripts/validate_plugins.py` requires the literal `## Verified Workflow` heading; it makes no verification claim.)

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| 1 | Plan to add `hypothesis` to only `pyproject.toml [optional-dependencies].dev`. | `pixi run test` (CI) installs from the conda/pixi env, not pip — the dep would be missing at CI runtime. | Dual-manifest repos need the dep in BOTH `pyproject` dev extras AND `pixi.toml [feature.dev.dependencies]`. |
| 2 | Add `hypothesis` to `pixi.toml [feature.dev.pypi-dependencies]` (the pip-on-conda table). | `hypothesis` is on conda-forge; the pypi table installs it via pip into the conda env, fighting the solver and diverging from the pytest/pytest-cov convention. | Decision rule: conda-forge package → `[feature.dev.dependencies]` (conda); only use `pypi-dependencies` for deps NOT on conda-forge. |
| 3 | Assume a new dep must be added to a floor-consistency guard test. | `test_dependency_floor_consistency.py` only enforces PyGithub/pytest/mypy floors; arbitrary deps are not lockstep-checked. | Read the guard's scope before writing a "required" consistency test; don't invent enforcement that doesn't exist. |
| 4 | Plan property tests with bare `st.text()` and assert parsed values. | `st.text()` almost never produces a ```json fence or a verdict token, so structured branches go untested; asserting parsed values fights the fail-safe contract. | Compose generators with crafted fragments; assert the never-raises + typed-default CONTRACT, plus metamorphic invariants. |
| 5 | Pick floor `hypothesis>=6.0,<7` by convention and treat the plan as ready. | The floor was never resolved against `pixi.lock` (`pixi install` was never run) — solvability is unverified. | Run `pixi install` to confirm the floor solves before committing the constraint; an unverified floor is the top risk. Add a fallback (raise the floor) step. |
| 6 | **On re-plan after a NOGO, return a meta-narrative** ("my plan above is correct and complete" / "as written above") instead of re-emitting the plan. | The reviewer grades ONLY the returned text — the artifact IS the plan body. Pointing at "the plan above" leaves the reviewed artifact EMPTY → guaranteed NOGO on requirements alignment, completeness, concreteness, AND verification (nothing to grade). The #455/#468/#484/#693 pattern. | When re-planning after a NOGO, ALWAYS re-emit the FULL plan markdown (all sections, real file:line anchors, fenced code, runnable per-criterion verification commands). Never a diff, summary, or self-affirmation. |
| 7 | **Hide unverified load-bearing steps** to make the plan look finished (silently imply the floor solved, the contract was executed). | A plan that quietly claims verification reads as false confidence; when the reviewer probes a load-bearing step it has no risk/mitigation and the plan collapses. | Promote every unverified reliance to a NAMED RISK + mitigation step. An honest "floor not solved; pixi install never ran; fallback = raise floor" reviews BETTER than implied verification. |

## Results & Parameters

**Re-plan-after-NOGO checklist (TASK/PLAN/REVIEW pipeline):**

```text
[ ] Re-emit the FULL plan markdown — never "the plan above" / a diff / a self-affirmation.
[ ] Every load-bearing claim has a real file:line anchor confirmed by a file read.
[ ] Every acceptance criterion has a runnable verification command.
[ ] Every unverified reliance appears as a NAMED RISK with a mitigation/fallback step.
```

**Dependency add (both manifests, floor matched, conda table):**

```toml
# pyproject.toml  (pip path)
[project.optional-dependencies]
dev = [
  # ...
  "hypothesis>=6.0,<7",
]

# pixi.toml  (conda path — what `pixi run test` uses; hypothesis is on conda-forge ->
#             [feature.dev.dependencies], NOT [feature.dev.pypi-dependencies])
[feature.dev.dependencies]
hypothesis = ">=6.0,<7"
```

**Parser fail-safe contracts (assert THESE, derived by reading source):**

| Parser | Return type | Garbage/empty behavior | Key invariant |
|--------|-------------|------------------------|---------------|
| `parse_review_verdict` | `ReviewVerdict` | `verdict="AMBIGUOUS"`, `raw == input` | never raises |
| `latest_verdict` | `"GO"` / `"NOGO"` / `None` | `None` when no marker | **last-match-wins** (`findall[-1]`) |
| `_parse_coordinator_results` | `list[dict]` | `[]` on prose; bad JSON-in-fence **skipped** | never raises on malformed fence |
| `_parse_addressed_block` | `dict` | default `{"addressed": [], "replies": {}}` | delegates to `_review_utils.parse_json_block` (`use_last_block=True`, `_review_utils.py:553-563`) |

**Generator composition sketch:**

```python
from hypothesis import given, settings, strategies as st

VERDICT_TOKENS = st.sampled_from(["GO", "NOGO", "AMBIGUOUS", "go", "nogo"])
FENCE = st.builds(lambda body: f"```json\n{body}\n```", st.text())
NOISE = st.text() | st.just("\r\n") | st.just("**bold**")

llm_output = st.lists(st.one_of(NOISE, VERDICT_TOKENS, FENCE)).map("\n".join)

@settings(deadline=None)  # default deadline may trip under --cov
@given(llm_output)
def test_latest_verdict_never_raises_and_typed(s: str) -> None:
    assert latest_verdict(s) in {"GO", "NOGO", None}
```

**Risks the reviewer should focus on (unverified reliances — named, not hidden):**

- Floor `hypothesis>=6.0,<7` chosen by convention, **not** verified to solve against `pixi.lock` — `pixi install` was never run. Mitigation: run `pixi install`; if the solver fails, raise/relax the floor and re-run.
- `_parse_addressed_block`'s real parsing is transitive via `_review_utils.parse_json_block` (`use_last_block` default `True`, read at `_review_utils.py:553-563`); the default-shape contract is asserted transitively but delegation under adversarial input was never executed.
- Property tests can be flaky/slow under coverage; Hypothesis's default `--deadline` may trip under `--cov`. Mitigation: `@settings(deadline=None)` on the property tests.
- Class names (`TestParseReviewVerdict`, etc.) and import line numbers were confirmed by reading files, **not** by running the suite.

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #1470 — planning Hypothesis fuzz tests for 4 LLM-output parsers (plan only, never executed); v1.1.0 re-plan after a self-review/meta-narrative NOGO | unverified |
