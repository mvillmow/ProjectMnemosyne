---
name: testing-sleep-free-timing-planning-risks
description: "Planning risk checklist for removing real sleeps from Python unit tests with fake clocks, condition/event coordination, and wait-timeout assertions. Use when: (1) replacing time.sleep() in tests, (2) injecting a monotonic clock into stateful timing code, (3) coordinating threaded tests with Condition.wait or Event, (4) verifying retry/backoff tests already patch wait primitives."
category: testing
date: 2026-06-30
version: "1.0.0"
user-invocable: false
verification: unverified
tags:
  - pytest
  - time-sleep
  - fake-clock
  - monotonic
  - threading
  - condition-wait
  - deterministic-tests
  - planning
  - circuit-breaker
  - backoff
---

# Sleep-Free Timing Tests: Planning Risks

## Overview

| Field | Value |
|-------|-------|
| **Date** | 2026-06-30 |
| **Objective** | Plan a deterministic replacement for real `time.sleep()` calls in timing-sensitive unit tests without overstating unverified source assumptions |
| **Outcome** | Plan produced for ProjectHephaestus issue #1469. The implementation was not executed during this learning capture. |
| **Verification** | unverified |

This skill records a planning-stage checklist for replacing wall-clock sleeps in tests. The concrete
ProjectHephaestus plan proposed three tactics: inject a monotonic clock into `CircuitBreaker`, advance a
`FakeClock` in circuit-breaker recovery tests, coordinate `StatusTracker` thread tests with
`threading.Event` plus observed `Condition.wait()` calls, and re-verify that NATS subscriber backoff tests
already patch `_stop_event.wait(timeout=...)` instead of sleeping.

> **Warning:** This workflow has not been validated end-to-end. Treat it as a planning hypothesis until
> the target repo implementation, tests, lint, and type checks pass.

## When to Use

- Planning or reviewing a test cleanup that removes real `time.sleep()` from unit tests.
- A stateful class uses `time.monotonic()` for recovery windows, retry-after values, circuit breakers, or cooldowns.
- Tests wait for background threads and currently depend on scheduler timing.
- A retry/backoff test claims it is deterministic because it patches an event or wait primitive.
- The plan cites exact source line numbers, issue acceptance criteria, or grep results that may drift before implementation.

## Verified Workflow

> **Warning:** This workflow has not been validated end-to-end. The section name is kept as
> `Verified Workflow` because the marketplace validator requires that heading, but the workflow itself is
> unverified until the target repo's test and type gates pass.

### Quick Reference

```bash
# Re-ground every premise on current HEAD before editing.
rg -n "time\\.sleep\\(" tests/unit/resilience/test_circuit_breaker.py \
  tests/unit/automation/test_status_tracker.py \
  tests/unit/nats/test_subscriber_backoff.py

rg -n "time\\.monotonic|Condition\\.wait|_stop_event\\.wait" \
  hephaestus/resilience/circuit_breaker.py \
  hephaestus/automation/status_tracker.py \
  tests/unit/nats/test_subscriber_backoff.py

# After implementation, prove the affected tests no longer sleep.
if rg -n "time\\.sleep\\(" tests/unit/resilience/test_circuit_breaker.py \
  tests/unit/automation/test_status_tracker.py \
  tests/unit/nats/test_subscriber_backoff.py; then
  exit 1
fi
```

```python
class FakeClock:
    def __init__(self, now: float = 0.0) -> None:
        self.now = now

    def __call__(self) -> float:
        return self.now

    def advance(self, seconds: float) -> None:
        self.now += seconds
```

### Detailed Steps

1. **Re-open the issue and current files before implementation.** Treat issue text, line numbers, and
   prior `rg` output as stale until re-checked. In the ProjectHephaestus #1469 plan, the cited
   `CircuitBreaker`, `StatusTracker`, and NATS line numbers were planning context, not fresh learn-step
   verification.

2. **Map every production clock read before adding injection.** If a class uses `time.monotonic()` in
   state transition, retry-after/ETA reporting, and failure timestamp recording, all three reads must
   use the injected clock. Missing one path leaves tests partly deterministic and partly wall-clock-bound.

3. **Use keyword-only constructor injection with a production default.** Prefer a module-level default
   wrapper such as `_default_clock() -> float: return time.monotonic()` and a keyword-only
   `clock: Callable[[], float] = _default_clock`. This preserves normal callers while giving tests a
   focused seam.

4. **Advance fake time instead of sleeping.** Recovery-window tests should trigger failures, assert the
   open state, call `fake_clock.advance(timeout + margin)`, and assert the half-open or closed transition.
   Include boundary cases exactly at timeout and before timeout.

5. **Treat thread scheduling as a synchronization problem, not a delay problem.** For tests that wait on
   availability or completion, use `threading.Event`, bounded joins, and explicit `not thread.is_alive()`
   assertions. A patched `Condition.wait` observer can prove a waiter entered the blocking path, but it is
   brittle and must be guarded by timeouts.

6. **Do not assume a `Condition.wait` observer is portable.** Patching a specific `Condition` instance
   can depend on Python implementation details and can miss fast paths where the waiter never blocks.
   Structure the test state so the waiter must block, then verify the observer actually fired before
   releasing the condition.

7. **Re-verify backoff tests before declaring them out of scope.** A test that patches
   `_stop_event.wait(timeout=...)` and records timeout values is already deterministic, but this must be
   confirmed on current HEAD. Do not rely on the plan's old grep result.

8. **Run targeted behavior tests plus lint/type gates.** At minimum run the affected test files, a negative
   grep for remaining `time.sleep()`, ruff on changed files, and mypy if the constructor signature changed.

## Failed Attempts

| Attempt | What Was Tried | Why It Failed | Lesson Learned |
|---------|----------------|---------------|----------------|
| Trust old line-number evidence | Planned from cited source lines and grep output without re-opening files during the learn step | Source lines, issue text, and test locations can drift before implementation | Re-ground every cited path and line on current HEAD before coding |
| Patch only one clock read | Inject a fake clock for the state transition but leave ETA or failure timestamp on `time.monotonic()` | The test suite still depends on real time through the unconverted path | Audit every `time.monotonic()` use in the class and route all timing semantics through the seam |
| Use sleeps as thread scheduling | Wait briefly for a worker to reach `Condition.wait()` before releasing a slot | Scheduler timing is nondeterministic under load and coverage; the sleep can be too short or unnecessarily slow | Coordinate with `Event`/`Condition` observation and bounded `join()` assertions |
| Assume instance-level `Condition.wait` patching is always safe | Patch `tracker.condition.wait` and use the patch as the only readiness signal | Python version or fast-path behavior may bypass the expected call, deadlocking or masking notify behavior | Force the blocked precondition, require the observer to fire, and keep timeouts on waits and joins |
| Treat NATS backoff scope as unchanged | Leave backoff tests untouched because an earlier grep showed `_stop_event.wait` patching | Tests may have changed since planning, or a new `time.sleep()` may have appeared | Re-run the grep and targeted NATS backoff tests before excluding them from edits |

## Results & Parameters

### ProjectHephaestus #1469 Plan Shape

| Area | Planned Change | Reviewer Risk |
|------|----------------|---------------|
| `CircuitBreaker` | Add keyword-only `clock` injection and replace direct `time.monotonic()` reads | Ensure all transition, retry-after/ETA, and failure timestamp paths use the fake clock |
| Circuit-breaker tests | Replace `time.sleep()` and module-level `time.monotonic` patching with `FakeClock.advance(...)` | Confirm boundary tests cover before-timeout and exactly-at-timeout behavior |
| `StatusTracker` tests | Replace scheduling sleeps with `threading.Event`, a wait observer, bounded joins, and alive checks | Ensure observer cannot deadlock, miss fast paths, or mask `notify_all()` behavior |
| NATS backoff tests | Verify existing `_stop_event.wait(timeout=...)` patching and recorded timeout assertions | Do not change retry semantics unless current tests actually sleep |

### Unverified Reliances to Surface in Review

- GitHub issue #1469 acceptance details were assumed from the planning prompt, not freshly fetched.
- ProjectHephaestus files and `rg` results were referenced from planning context, not re-opened during this
  learn capture.
- Python `threading.Condition` patchability and behavior were assumed from standard-library expectations,
  not checked across every supported Python version.
- The plan assumed all `CircuitBreaker` callers tolerate a new keyword-only parameter with a default; mypy
  and any subclass/factory call sites must prove that.
- The plan assumed NATS backoff tests already use `_stop_event.wait` patching; re-check before excluding them.

### Reviewer Checklist

```text
1. No real time.sleep() remains in the affected unit-test files.
2. Every time.monotonic() path in the timing class uses the injected clock.
3. Production constructor calls still work without passing clock=.
4. Fake-clock tests cover pre-timeout, exactly-at-timeout, and post-timeout transitions.
5. Thread tests use bounded joins and assert no worker thread is left alive.
6. The Condition.wait observer cannot deadlock or hide broken notify/notify_all behavior.
7. NATS backoff tests still validate delay values without changing retry semantics.
8. ruff and mypy accept the changed constructor API and test helpers.
```

### Suggested Verification Commands

```bash
pixi run pytest tests/unit/resilience/test_circuit_breaker.py -q
pixi run pytest tests/unit/automation/test_status_tracker.py -q
pixi run pytest tests/unit/nats/test_subscriber_backoff.py -q
bash -lc 'if rg -n "time\\.sleep\\(" tests/unit/resilience/test_circuit_breaker.py tests/unit/automation/test_status_tracker.py tests/unit/nats/test_subscriber_backoff.py; then exit 1; fi'
pixi run ruff check hephaestus/resilience/circuit_breaker.py \
  tests/unit/resilience/test_circuit_breaker.py \
  tests/unit/automation/test_status_tracker.py \
  tests/unit/nats/test_subscriber_backoff.py
pixi run mypy
```

## Verified On

| Project | Context | Details |
|---------|---------|---------|
| ProjectHephaestus | Issue #1469 planning session | unverified - plan only; implementation, tests, lint, type checks, and CI were not executed during this learn capture |
