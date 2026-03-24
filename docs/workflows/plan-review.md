# Plan Review

> **When to use**: After creating an implementation plan but before starting code — structured self-review + adversarial challenge + approval
> **Time estimate**: 30–60 min for a focused plan; 2 hours for a complex multi-service plan
> **Prerequisites**: Plan document exists (inline or in `docs/plans/`)

## Overview

Structured plan review using the `plan-mode-review` skill and `plan-challenger` agent. Runs Phase 0 self-review, 5-phase analysis (Architecture, Code Quality, Tests, Performance, Production Readiness), approval scope triage, and adversarial challenge. Required before any significant implementation.

---

## When Plan Review is Required

From `leverage-patterns.md` — use plan review for:
- Architecture changes or multi-service impact
- Schema changes (irreversible structural decisions)
- New agents, files, or significant abstractions
- Any work touching more than 2 components

Skip for: trivial single-file changes, typo fixes, config tweaks.

---

## Command

**Command**: `/plan-review`
**Source**: `commands/plan-review.md`
**Skill**: `plan-mode-review` (`.claude/skills/plan-mode-review/SKILL.md`)

---

## Phases

### Phase 0 — Self-Review (Before External Challenges)

Before presenting a plan for review, Claude performs self-review:

```
SELF-REVIEW CHECKLIST:
1. Does the plan solve the stated problem? (not a different problem)
2. Are all assumptions explicit?
3. Are dependencies between phases identified?
4. Is there a rollback plan for irreversible changes?
5. Is this the simplest correct solution? (core-behaviors.md §4)
6. Does any phase require coordination with other teams?
```

**Output**: Self-review findings that are fixed before plan-challenger sees the plan.

---

### Phase 1 — Architecture Review

**Questions asked**:
- Are component boundaries clear? (each service has one responsibility)
- Are data flows explicit? (no implicit coupling)
- Are external dependencies identified? (what breaks if X is unavailable)
- Is the communication pattern right? (sync/async, retry, circuit breaker)
- Are there circular dependencies?
- Is there a single point of failure?

**Approval scope triage** (from `plan-mode-review` skill):

| Color | Decision Type | Action |
|-------|--------------|--------|
| 🔴 RED | Irreversible structural change | Explicit approval required |
| 🟡 YELLOW | Reversible change with side effects | Inform and proceed |
| 🟢 GREEN | Isolated change | Just do it |

---

### Phase 2 — Code Quality Review

**Questions asked**:
- Does the plan introduce abstractions before 3 proven use cases? (Rule of Three)
- Are there opportunities to reuse existing shared utilities?
- Will this create technical debt?
- Are error handling patterns consistent with the codebase?
- Does the plan follow the Three-Pass Development approach?

---

### Phase 3 — Test Coverage Review

**Questions asked**:
- What tests will verify each phase?
- Are edge cases and failure modes covered?
- Is there a regression test plan for changed behavior?
- Will the plan break any existing tests?
- Is E2E testing planned for critical paths?

---

### Phase 4 — Performance Review

**Questions asked**:
- Are there any N+1 query risks?
- Are indexes planned for new query patterns?
- Is there caching strategy for expensive operations?
- What is the expected latency for critical paths?
- Is there a load estimate?

---

### Phase 5 — Production Readiness

**Questions asked**:
- Is the change backward-compatible? (or is a migration plan needed)
- Are secrets handled via environment variables / Secret Manager?
- Are security controls in place? (auth, input validation, rate limiting)
- Is observability planned? (logs, metrics, alerts)
- Is there a deployment plan? (staged rollout, feature flag, or direct)
- Are database migrations reversible?

---

### Phase 6 — Adversarial Challenge (plan-challenger)

**Agent**: `plan-challenger` (opus, read-only)

**Dispatch**:
```
Review the plan at docs/plans/[date]-[feature].md or paste inline.
Attack it on: Assumptions, Missing Cases, Security, Architecture, Complexity Creep.
```

**5-dimension attack** (from plan-challenger agent):
1. **Assumptions** — what is taken as true without proof?
2. **Missing Cases** — what failure modes are not addressed?
3. **Security** — what attack vectors are unmitigated?
4. **Architecture** — what structural anti-patterns are present?
5. **Complexity Creep** — what is more complex than it needs to be?

**Refutation check**: plan-challenger applies false-positive elimination before reporting — not every identified risk is a real issue.

**Gate**: All CRITICAL findings addressed before implementation starts.

---

### Phase 7 — Approval and Save

**Approval format**:
```
PLAN APPROVED:
- Self-review: PASS
- Architecture: PASS (findings addressed: [list])
- Code quality: PASS
- Tests: PASS
- Performance: PASS
- Production readiness: PASS
- plan-challenger: [N CRITICAL → addressed / 0 CRITICAL]

Proceeding with implementation.
Saved to: docs/plans/YYYY-MM-DD-<feature>.md
```

**Save**: Approved plan to `docs/plans/YYYY-MM-DD-<feature>.md` before starting implementation.

**Why save**: Gives a persistent, git-committed reference for the session and future sessions. If context is lost (compaction, new session), the plan file is the recovery point.

---

## Quick Reference

| Phase | Check | Gate |
|-------|-------|------|
| 0 — Self-review | Internal consistency, assumptions, simplicity | Issues fixed before external review |
| 1 — Architecture | Boundaries, data flow, dependencies | No circular deps, no SPOF |
| 2 — Code quality | DRY, Rule of Three, error handling | No premature abstractions |
| 3 — Tests | Coverage plan, regression plan | Test plan exists |
| 4 — Performance | N+1 risk, indexes, caching | Estimates exist for critical paths |
| 5 — Production | Backward compat, security, observability | Migration plan if needed |
| 6 — plan-challenger | 5-dimension adversarial review | Zero CRITICAL findings |
| 7 — Save | `docs/plans/` commit | File saved and committed |

---

## Common Pitfalls

- **Plan review as a formality** — rubber-stamping your own plan; the challenger should genuinely challenge it
- **Skipping for "small" changes** — schema changes are never small; use plan review for any irreversible decision
- **Plan too detailed** — a plan should describe WHAT and WHY, not HOW line-by-line; over-specified plans become outdated immediately
- **Not saving the plan** — plans discussed verbally are lost between sessions; always write to `docs/plans/`
- **Implementing before approval** — starting code while the plan is still being challenged leads to wasted work if the plan changes

## Related Workflows

- [`architecture-design.md`](architecture-design.md) — produces the plan that this workflow reviews
- [`adr-creation.md`](adr-creation.md) — ADRs capture key decisions within the plan
- [`ideation-to-spec.md`](ideation-to-spec.md) — spec before plan, plan before implementation
