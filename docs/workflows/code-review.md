# Code Review

> **When to use**: After writing or modifying code — before committing, creating a PR, or merging
> **Time estimate**: 5–20 minutes depending on scope
> **Prerequisites**: Code is implemented and tests pass

## Overview

Two commands cover code review in this workspace: `/review-code` for stack-specific quality + security review during development, and `/review-pr` for a 6-role comprehensive review before merge. Both dispatch specialized agents — the right agent per stack.

---

## When to Use Which Command

| Situation | Command |
|-----------|---------|
| Just wrote code, want quick feedback | `/review-code` |
| About to open a PR | `/review-pr` |
| Want to topic-filter (e.g. only test coverage) | `/review-pr tests` |
| Need security-focused review only | `/review-code` (always runs security-reviewer) |
| Reviewing someone else's PR | `/review-pr` |

---

## `/review-code` — Stack-Aware Review

**Command**: `/review-code [optional: path or file]`
**Source**: `commands/review-code.md:14-23`

### Stack Detection and Agent Routing

| Files Changed | Primary Reviewer | Also Runs |
|---------------|-----------------|-----------|
| `*.java` | `spring-reactive-reviewer` | `security-reviewer` |
| `*.ts` with NestJS decorators | `nestjs-reviewer` | `security-reviewer` |
| `*.ts` / `*.html` (Angular) | `code-reviewer` | `security-reviewer` |
| `*.dart` | `riverpod-reviewer` | `security-reviewer` |
| `*.py` with LangChain/LangGraph | `agentic-ai-reviewer` | `security-reviewer` |
| `*.py` (general) | `code-reviewer` | `security-reviewer` |
| `*.sql` | `postgresql-database-reviewer` | `security-reviewer` |
| Mixed / other | `code-reviewer` | `security-reviewer` |

`security-reviewer` **always runs** — regardless of stack.

### What Each Reviewer Checks

**`spring-reactive-reviewer`** (sonnet)
Vibe: *"One blocking call in a reactive chain kills the whole thread pool"*
- Blocking calls in reactive chains (`.block()`, JDBC, `Thread.sleep()`)
- Resilience4j circuit breaker usage
- R2DBC patterns, reactive operator correctness
- WebTestClient test coverage

**`nestjs-reviewer`** (sonnet)
Vibe: *"Module correctness is non-negotiable — wiring errors fail silently in prod"*
- Module imports/exports wiring
- JWT security and guard coverage
- Prisma usage patterns
- Vitest coverage

**`riverpod-reviewer`** (sonnet)
Vibe: *"Wrong provider type = wrong lifecycle = subtle state bugs in prod"*
- Provider type selection (`Provider` vs `FutureProvider` vs `StreamProvider` vs `NotifierProvider`)
- `ref.watch` vs `ref.read` usage
- `AsyncValue.when()` pattern completeness
- Lifecycle correctness

**`agentic-ai-reviewer`** (sonnet)
Vibe: *"Finds the infinite loop before production does"*
- LangGraph graph correctness, no infinite loops
- Guardrail coverage
- Iteration limits
- Cost efficiency
- Production readiness

**`postgresql-database-reviewer`** (sonnet)
Vibe: *"No migration ships without EXPLAIN ANALYZE and a rollback plan"*
- Index coverage for expected query patterns
- Migration reversibility
- Constraint correctness
- Query plan implications

**`code-reviewer`** (sonnet)
Vibe: *"Finds real bugs, not style preferences — ≥80% confidence before raising an issue"*
- General quality, DRY violations, dead code
- Unused imports, variables
- Pattern consistency

**`security-reviewer`** (opus)
Vibe: *"Assumes every input is hostile until the code proves otherwise"*
- Secrets / hardcoded credentials
- OWASP Top 10: injection, XSS, SSRF, auth bypass, insecure deserialization
- Unsafe crypto (MD5, SHA1, ECB mode)
- Missing input validation
- Debug endpoints exposed

---

## `/review-pr` — 6-Role Comprehensive Review

**Command**: `/review-pr [optional: topic filter]`
**Source**: `commands/review-pr.md:15-20`

Dispatches **6 agents concurrently**, each with a distinct concern:

| Role | Agent | What It Checks |
|------|-------|---------------|
| Comment accuracy | `comment-analyzer` | Outdated Javadoc/JSDoc/docstrings vs actual implementation |
| Test coverage | `pr-test-analyzer` | Behavioral gaps (critical paths, edge cases, error conditions) |
| Error handling | `silent-failure-hunter` | Swallowed exceptions, catch returning empty, missing logging |
| Type design | `type-design-analyzer` | Types that allow invalid states, missing invariants |
| Code quality | `code-reviewer` | General quality, patterns, DRY |
| Simplification | `code-simplifier` | Over-engineering, premature abstraction, unnecessary nesting |

### Topic Filtering

Run specific roles only:
```
/review-pr comments    # comment-analyzer only
/review-pr tests       # pr-test-analyzer only
/review-pr errors      # silent-failure-hunter only
/review-pr types       # type-design-analyzer only
/review-pr code        # code-reviewer only
/review-pr simplify    # code-simplifier only
```

### Interpreting Findings

**CRITICAL** — must fix before merge; correctness or security issue
**HIGH** — should fix; significant quality or safety risk; requires explicit justification to skip
**MEDIUM** — fix in this PR or create a follow-up ticket
**LOW** — optional; style or minor improvement

### How to Respond to Findings

1. **CRITICAL**: Stop. Fix it. Re-run `/review-pr` to confirm resolved.
2. **HIGH**: Fix or document reason for deferral in PR description.
3. **MEDIUM**: Fix inline or create ticket. Link ticket in PR comment.
4. **LOW**: Fix if trivial (<5 min). Otherwise ignore.

---

## Agents Involved (Quick Lookup)

| Agent | Model | Permission | Writes files? |
|-------|-------|-----------|---------------|
| `spring-reactive-reviewer` | sonnet | default | No |
| `nestjs-reviewer` | sonnet | default | No |
| `riverpod-reviewer` | sonnet | default | No |
| `agentic-ai-reviewer` | sonnet | default | No |
| `postgresql-database-reviewer` | sonnet | default | No |
| `code-reviewer` | sonnet | default | No |
| `security-reviewer` | opus | default | No |
| `comment-analyzer` | sonnet | default | No |
| `pr-test-analyzer` | sonnet | default | No |
| `silent-failure-hunter` | sonnet | default | No |
| `type-design-analyzer` | sonnet | default | No |
| `code-simplifier` | sonnet | default | No |

All review agents are **read-only** — they inspect and report, never modify.

---

## Common Pitfalls

- **Running the wrong reviewer** — stack detection is automatic with `/review-code`; don't manually dispatch `nestjs-reviewer` on Java files
- **Ignoring MEDIUM findings** — they compound; 10 MEDIUM findings across 5 PRs = tech debt sprint
- **Dismissing `code-simplifier` feedback** — over-engineering is a correctness risk; complex code hides bugs
- **Skipping `security-reviewer`** — it runs automatically with `/review-code`; if running agents manually, include it
- **Not re-running after fixes** — findings addressed but not re-verified; always re-run the specific reviewer after fixing CRITICAL/HIGH

## Related Workflows

- [`pre-commit-validation.md`](pre-commit-validation.md) — run before this workflow
- [`pr-shipping.md`](pr-shipping.md) — run after this workflow
- [`security-audit.md`](security-audit.md) — full codebase security audit
- [`tech-debt-cleanup.md`](tech-debt-cleanup.md) — addressing accumulated MEDIUM/LOW findings
