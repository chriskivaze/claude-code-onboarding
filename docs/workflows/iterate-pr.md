# Iterate PR (CI Fix Loop)

> **When to use**: CI is failing, review comments need addressing, or a PR needs autonomous iteration until it's green
> **Time estimate**: 15–30 min per CI fix cycle; up to 2 hours for complex failures
> **Prerequisites**: PR opened; CI pipeline running; access to GitHub Actions logs

## Overview

Autonomous PR iteration using the `iterate-pr` skill and command. Fetches CI failure logs, diagnoses the root cause, applies the fix, pushes, and repeats until CI is green — without manual intervention per cycle. Uses `error-detective` agent for complex log analysis and `silent-failure-hunter` for subtle issues.

---

## Command

**Command**: `/iterate-pr`
**Source**: `commands/iterate-pr.md`
**Skill**: `iterate-pr` (`.claude/skills/iterate-pr/SKILL.md`)

---

## Phases

### Phase 1 — Assess the Current State

**Fetch CI status**:
```bash
# Check PR CI status
gh pr checks <PR-number>

# Get specific job failure log
gh run view <run-id> --log-failed

# List recent runs for current branch
gh run list --branch $(git branch --show-current)
```

**Classify the failure**:

| Failure Type | Signature | Action |
|-------------|-----------|--------|
| Test failure | `FAILED: [test name]` | Fix the code or test |
| Lint/type error | ESLint, tsc, dartanalyze errors | Fix the violation |
| Build failure | Compilation error | Fix the syntax or import |
| Docker build | Layer error | Fix Dockerfile or dependencies |
| Security scan | CRITICAL CVE or semgrep finding | Fix the vulnerability |
| Flaky test | Passes sometimes | Add retry or fix timing |

---

### Phase 2 — Diagnose the Failure

**For test failures**:
```bash
# Run the specific failing test locally
npx vitest run src/orders/orders.service.spec.ts    # NestJS
./mvnw test -Dtest="OrderServiceTest#createOrder"  # Java
uv run pytest tests/test_orders.py::test_create -v # Python
flutter test test/order_test.dart                   # Flutter
```

**For build failures** — read the error output:
```
error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'
  at src/orders/order.service.ts:42:25
```
→ Read `src/orders/order.service.ts:42`, fix the type mismatch.

**For complex failures** — dispatch `error-detective`:
```
Review this CI failure log and identify root cause:
[paste log output]
```

---

### Phase 3 — Apply Fix and Push

**Apply the minimum fix** (scope discipline — fix ONLY what CI is failing on):

```bash
# Stage the fix
git add -p    # Interactive staging — only stage the fix

# Commit
git commit -m "fix: [specific thing that was failing]

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"

# Push
git push
```

**Run tests locally before pushing** (don't push hoping CI will tell you if it works):
```bash
npx vitest run        # Run all tests
tsc --noEmit          # TypeScript type check
```

---

### Phase 4 — Monitor and Repeat

**After pushing**:
```bash
# Wait for CI to start
gh run list --branch $(git branch --show-current) --limit 1

# Watch CI progress
gh run watch <run-id>
```

**If CI passes**: Done. Update PR description with what was fixed.

**If CI fails again**: Repeat Phase 1 — do not assume the same root cause; read the new failure.

**Stop condition** (when to escalate to human):
- Same test fails 3 times with different fixes
- Failure is in a system you don't have access to (secrets, external service)
- Fix requires architectural change
- Two different failures now appear

---

### Phase 5 — Address Review Comments in Same Loop

If CI is passing but review comments need addressing, continue iterating:

**Prioritize**:
1. CRITICAL findings from review (security, correctness bugs) — fix immediately
2. HIGH findings — fix before merging
3. Style / preference — fix if straightforward, discuss otherwise

**For each comment cycle**:
1. Read comment
2. Evaluate (see `receiving-code-review.md`)
3. Implement fix
4. Run local tests
5. Push
6. Confirm CI still green

---

### Phase 6 — `/iterate-pr` Autonomous Mode

For complex iteration with many CI failures, use the full autonomous loop:

```
/iterate-pr
```

**What it does** (from skill):
1. Fetch current CI status via `gh pr checks`
2. Read failure logs via `gh run view --log-failed`
3. Identify root cause
4. Apply fix
5. Run local verification
6. Commit and push
7. Monitor until green OR escalation condition met

**Escalation conditions** (it stops and asks you):
- Same failure 3 times (stuck)
- Security vulnerability fix requires architectural decision
- Fix would change behavior beyond fixing the test
- Resource constraint (out of retries, rate limited)

---

## Quick Reference

| Phase | Action | Tool | Gate |
|-------|--------|------|------|
| 1 — Assess | `gh pr checks` + `gh run view --log-failed` | GitHub CLI | Failure classified |
| 2 — Diagnose | Run specific failing test locally | Stack test runner | Root cause identified |
| 3 — Fix | Minimum fix + push | git | Local tests pass |
| 4 — Monitor | `gh run watch` | GitHub CLI | CI green |
| 5 — Review | Address review comments | Manual | Comments resolved |
| 6 — Done | `/iterate-pr` for autonomous loop | Skill | PR: all checks green, no open comments |

---

## Common Pitfalls

- **Pushing without local test run** — CI time is slow feedback; run tests locally first
- **Fixing symptoms, not cause** — CI failure at line 50 might be caused by a change at line 10; read the full stack trace
- **Big fixes per push** — small atomic commits are easier to revert if they introduce new failures
- **Not reading the new failure** — after a fix, CI can fail for a different reason; always re-read the log
- **Ignoring `error-detective`** — for complex multi-step CI failures with correlated logs, the agent finds root cause faster than manual inspection

## Related Workflows

- [`pr-shipping.md`](pr-shipping.md) — running `/ship` before opening the PR prevents most CI failures
- [`receiving-code-review.md`](receiving-code-review.md) — addressing review comments in the iteration loop
- [`bug-fix.md`](bug-fix.md) — when the CI failure reveals a real bug (not just a broken test)
