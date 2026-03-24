# Ideation to Spec

> **When to use**: Before writing a single line of code — when you have a feature idea and need a validated spec and approved plan.
> **Time estimate**: 30 min – 2 hours depending on feature complexity
> **Prerequisites**: None — this is the starting point. If the solution space is still open (you don't know yet which approach to take), run `/brainstorm` first.

## Overview

Transforms a raw feature idea into a challenge-tested specification and an approved implementation plan. Uses structured elicitation (feature-forge skill), adversarial critical reasoning (the-fool skill), and a gated plan review (plan-mode-review skill + plan-challenger agent) before any implementation begins.

## Phases

### Phase 0 — Divergent Exploration (optional, run when approach is undecided)

**Trigger**: You have a problem but haven't decided on an approach yet (e.g. "should this be REST or GraphQL?", "which DB fits here?", "WebSockets vs SSE?")
**Command**: `/brainstorm [topic]`

Generates ≥3 distinct alternatives with trade-offs and a Mermaid diagram. No code — exploration only. Run this before Phase 1 to avoid committing to the wrong approach in the spec.

**Gate**: An approach is selected (or explicitly left open) before proceeding to Phase 1.

---

### Phase 1 — Feature Elicitation

**Trigger**: You have a feature idea but no formal spec
**Skill**: Load `feature-forge` skill (`skills/feature-forge/SKILL.md`)
**5-step process** (from `skills/feature-forge/SKILL.md:36-41`):
1. **Discover** — What problem does this solve? Who is the user?
2. **Interview** — Structured questions: happy path, edge cases, error states, non-functional requirements
3. **Document** — Write EARS-format acceptance criteria (`WHEN [trigger] THE SYSTEM SHALL [behaviour]`)
4. **Validate** — Read spec back to user, confirm nothing missing
5. **Plan** — Convert spec into an implementation TODO list with file-level detail

**Produces**: `docs/specs/YYYY-MM-DD-<feature>.md` with acceptance criteria and implementation TODO
**Gate**: User explicitly confirms spec is complete and correct

---

### Phase 2 — Critical Reasoning (Pre-Mortem)

**Trigger**: Spec is confirmed — before writing the plan
**Skill**: Load `the-fool` skill (`skills/the-fool/SKILL.md`)
**Select mode** (from `skills/the-fool/SKILL.md:39-53`):

| Mode | When to use |
|------|-------------|
| Expose Assumptions | Spec has implicit dependencies or untested beliefs |
| Find Failure Modes | Feature touches critical path (auth, payments, data) |
| Attack This | Plan exists — adversarially stress-test it |
| Argue the Other Side | You're too confident in the approach |
| Test the Evidence | Requirements have weak or assumed data backing |

**Output format** (from `skills/the-fool/SKILL.md:96-114`):
- Steelman: Best version of the argument
- Challenge: Strongest objection
- Synthesis: What the challenge reveals

**Produces**: List of open risks, unchecked assumptions, and questions to resolve
**Gate**: All critical risks either resolved or explicitly accepted with rationale

---

### Phase 3 — Implementation Plan

**Trigger**: Spec validated, risks addressed
**Command**: `/plan-review [describe the change or paste the spec]`
**Skill**: `plan-mode-review` (`commands/plan-review.md`, `skills/plan-mode-review/SKILL.md`)

**Select review mode**:
- **Big Change** — architecture change, multi-service, schema migration
- **Small Change** — single service, bounded scope
- **Review Only** — reviewing someone else's plan

**5 review phases** (from `skills/plan-mode-review/SKILL.md:40-48`):
1. Architecture — structural decisions, dependencies, contracts
2. Code Quality — naming, patterns, standards compliance
3. Tests — coverage strategy, test types, edge cases
4. Performance — NFRs, latency targets, load expectations
5. Production Readiness — migrations, rollback, monitoring, secrets

**Phase 0 self-review** runs first — checks for contradictions in the plan itself

**Produces**: Reviewed plan saved to `docs/plans/YYYY-MM-DD-<feature>.md`
**Gate**: Plan review passes all 5 phases or open items are explicitly accepted

---

### Phase 4 — Adversarial Plan Challenge

**Trigger**: Plan is drafted — before implementation starts
**Agent**: `plan-challenger` (opus model, read-only)
**Vibe**: *"Optimism is the enemy — every plan has a flaw, it's just not found yet"*

**5 attack dimensions** (from `agents/plan-challenger.md`):
1. Assumptions — what does the plan take for granted?
2. Missing Cases — null, empty, concurrent, out-of-order, error states
3. Security — injection points, auth gaps, secret exposure
4. Architecture — coupling, circular dependencies, wrong abstraction level
5. Complexity Creep — is this over-engineered for the actual requirement?

**Refutation step**: Challenges are filtered — false positives eliminated before reporting

**Produces**: Prioritized list of genuine plan flaws with suggested mitigations
**Gate**: All CRITICAL and HIGH findings resolved or explicitly accepted

---

### Phase 5 — Finalize and Commit Spec

**Trigger**: Plan challenger findings resolved
**Actions**:
1. Update `docs/specs/YYYY-MM-DD-<feature>.md` with any requirement clarifications
2. Update `docs/plans/YYYY-MM-DD-<feature>.md` with mitigations from challenge
3. Commit both files: `docs: add spec and plan for <feature>`
4. Hand off to implementation workflow

**Produces**: Committed spec + plan, ready for implementation
**Gate**: Both files committed to git

---

## Quick Reference

| Phase | What to Run | Produces | Gate |
|-------|-------------|----------|------|
| 0 — Explore (optional) | `/brainstorm [topic]` | Approach selected | Approach decided |
| 1 — Elicitation | Load feature-forge skill | `docs/specs/YYYY-MM-DD-<feature>.md` | User confirms spec |
| 2 — Pre-mortem | Load the-fool skill | Risk list + open questions | Risks resolved or accepted |
| 3 — Plan review | `/plan-review` | `docs/plans/YYYY-MM-DD-<feature>.md` | All 5 phases pass |
| 4 — Challenge | `plan-challenger` agent | Flaw list with mitigations | CRITICAL/HIGH resolved |
| 5 — Finalize | Commit both files | Git commit | Files committed |

---

## Common Pitfalls

- **Skipping Phase 2** — the-fool catches assumptions that invalidate the plan before it's written, not after
- **Vague acceptance criteria** — use EARS format (`WHEN/THE SYSTEM SHALL`), not prose
- **Plan not saved to file** — if it's only in the chat, context loss erases it; always save to `docs/plans/`
- **Confusing spec and plan** — spec = WHAT (user-facing behaviour); plan = HOW (implementation steps with file paths)
- **Starting implementation before Phase 4** — plan-challenger runs on plans, not code; catches structural flaws cheaply

## Related Workflows

- [`architecture-design.md`](architecture-design.md) — for system-level design before feature-level spec
- [`feature-java-spring.md`](feature-java-spring.md) — implementation after spec is approved
- [`feature-nestjs.md`](feature-nestjs.md) — NestJS implementation after spec
- [`feature-flutter-mobile.md`](feature-flutter-mobile.md) — mobile implementation after spec
