# Bug Fix

> **When to use**: Reproducing and fixing a bug — from vague report to verified fix with regression test
> **Time estimate**: 30 min for clear bugs with stack trace; 2–4 hours for intermittent or complex bugs
> **Prerequisites**: Bug report with reproduction steps, error message, or stack trace

## Overview

Structured bug fix using the `systematic-debugging` skill and the Autonomy Ladder pattern. Signal clarity determines how much investigation is needed before coding. Produces a regression test before the fix — TDD applied to bugs.

---

## Autonomy Ladder (from `leverage-patterns.md`)

| Signal Clarity | Action |
|---------------|--------|
| **Clear** — stack trace, failing test, specific error | Act first: fix + test, report what you did |
| **Moderate** — error log pointing to code region | Investigate first (root cause), then fix |
| **Ambiguous** — "it's slow", "sometimes fails", no repro | Ask first: clarify scope and reproduction |

---

## Phases

### Phase 1 — Classify the Bug

**Read the bug report and classify**:

```
BEFORE I PROCEED, let me classify this bug:
Signal clarity: [CLEAR / MODERATE / AMBIGUOUS]
Reproduction: [can reproduce: YES / NO / SOMETIMES]
Stack trace: [present / absent]
Error location: [file:line or unknown]
```

**If AMBIGUOUS** — ask:
1. "What steps reproduce the issue?"
2. "What is the expected behavior?"
3. "What actually happens?"
4. "Is it reproducible on a specific environment?"

Do not touch code until you can reproduce the bug.

---

### Phase 2 — Enter Debug Mode

**Command**: `/debug [paste error message or describe the bug]`

This loads the `systematic-debugging` skill and enforces the 4-phase root-cause-first protocol automatically. Output format:
```
🔍 Symptom → 🔎 Investigation → 🎯 Root Cause → ✅ Fix → 🛡️ Prevention
```

**4-phase protocol** (from `.claude/skills/systematic-debugging/SKILL.md`):
1. **Root Cause Investigation** — read errors completely, reproduce consistently, check recent changes
2. **Pattern Analysis** — find working equivalent, list every difference
3. **Hypothesis and Testing** — form one hypothesis, test minimally, one variable at a time
4. **Implementation** — write failing test first, fix root cause, verify

**Iron Law** (`systematic-debugging/SKILL.md:28`): NO fixes without root cause investigation first.

---

### Phase 3 — Reproduce the Bug

**Write a failing test first** (TDD for bugs):

```java
// Java — write test that captures the bug before fixing
@Test
void createOrder_withDuplicateId_shouldReturnConflict_notThrow500() {
    // This test FAILS currently — demonstrates the bug
    webTestClient.post().uri("/api/orders")
        .bodyValue(Map.of("id", "order-1", "itemId", "item-1"))
        .exchange()
        .expectStatus().isCreated();

    webTestClient.post().uri("/api/orders")
        .bodyValue(Map.of("id", "order-1", "itemId", "item-1"))
        .exchange()
        .expectStatus().isEqualTo(409);  // Should be 409, currently 500
}
```

**Run it to confirm it fails**:
```bash
./mvnw test -Dtest="OrderControllerIT#createOrder_withDuplicateId_shouldReturnConflict_notThrow500"
# Expected: FAIL (the bug is confirmed)
```

If you can't write a failing test, you don't understand the bug yet — go back to Phase 2.

---

### Phase 4 — Root Cause Analysis

**Trace the execution path**:
1. Start from the failing test or error location
2. Read the actual code (not what you think it should be)
3. Trace data through the stack: HTTP layer → service → repository → DB
4. Find where behavior diverges from expectation

**Hypothesis format**:
```
Hypothesis 1: [specific claim about cause]
Evidence for: [what in the code supports this]
Evidence against: [what contradicts this]
Test: [how to confirm/deny without code changes]

Hypothesis 2: ...
```

**Common bug categories and where to look**:

| Bug Type | Where to Look |
|----------|--------------|
| NullPointerException | Null not checked before use; find the source of null |
| 500 instead of 4xx | Exception not caught; exception mapper not configured |
| Wrong data returned | Query filter incorrect; wrong field mapped |
| Intermittent failure | Race condition; connection pool exhausted; retry logic missing |
| Auth bypass | Guard not applied to route; RBAC check missing |
| Performance regression | N+1 query; missing index; eager vs lazy load |

**Tools**:
```bash
# Grep for the error class
# Find where the exception is thrown
grep -r "DataIntegrityViolationException" src/

# Read the stack trace from logs
# Trace the method chain
```

---

### Phase 5 — Apply the Fix

**Minimum viable fix** — fix the root cause only. Do not refactor surrounding code.

**After fixing**:
```bash
# Run the regression test you wrote in Phase 3
# It should now PASS
./mvnw test -Dtest="OrderControllerIT#createOrder_withDuplicateId_shouldReturnConflict_notThrow500"

# Then run the full suite — no regressions
./mvnw test
```

**If the regression test still fails**: your fix didn't address the root cause — go back to Phase 4.

---

### Phase 6 — Verify No Side Effects

**After all tests pass**, verify the fix didn't break adjacent behavior:

```
VERIFICATION:
- Failing test from Phase 3: ✅ NOW PASSES
- Full test suite: ✅ N passed, 0 failed
- Edge cases checked:
  - [Edge case 1]: [result]
  - [Edge case 2]: [result]
- Files changed:
  - [file:line] — [what changed and why]
- Files NOT changed (intentional):
  - [file] — [why left alone]
```

---

### Phase 7 — Dispatch `error-detective` for Complex Bugs

For runtime errors in production logs, intermittent failures, or multi-service bugs:

**Agent**: `error-detective`

Capabilities (from agent description):
- Parse logs with regex
- Correlate errors across services
- Identify root cause from runtime data
- NOT for code review — for investigating running system errors

```
Dispatch error-detective with:
- Log samples or error traces
- Time range of the failure
- Which services are involved
- What was deployed before the failure started
```

---

## Quick Reference

| Phase | Action | Gate |
|-------|--------|------|
| 1 — Classify | Signal clarity: CLEAR / MODERATE / AMBIGUOUS | Clarity determined |
| 2 — Debug mode | `/debug [description]` — loads systematic-debugging | Root cause identified |
| 3 — Reproduce | Write failing test; confirm it fails | Test output shows FAIL |
| 4 — Root cause | Trace + hypothesize + evidence | Specific cause identified (file:line) |
| 5 — Fix | Apply minimum fix; run regression test | Regression test PASSES |
| 6 — Verify | Run full suite; check side effects | All tests PASS, no regressions |

---

## Common Pitfalls

- **Fixing symptoms, not cause** — changing error handling to hide the error instead of fixing what causes it
- **No failing test first** — fixing without a regression test means the bug can silently return
- **Scope creep** — finding a bug and refactoring the surrounding code; fix only what's broken
- **Guessing without reading** — "it's probably in X" without actually reading X; always read the code
- **Not running full suite** — the fix passed your test but broke 3 others; run everything before reporting done
- **Production debugging without staging test** — reproduce in staging first; never debug in production

## Related Workflows

- [`test-driven-development.md`](test-driven-development.md) — the regression test pattern
- [`production-incident.md`](production-incident.md) — when the bug is live in production
- [`code-review.md`](code-review.md) — review the fix before shipping
