# Subagent-Driven Development (SDD)

> **When to use**: Implementing an approved plan with 3+ tasks that can be parallelized or pipelined across specialized agents
> **Time estimate**: Setup 30 min; execution time depends on task count and complexity
> **Prerequisites**: Approved plan in `docs/plans/`; plan-challenger has reviewed it; at least 3 discrete tasks

## Overview

3-role SDD pipeline from the `subagent-driven-development` skill: Implementer → Spec Reviewer → Quality Reviewer. Supports both Agent Tool (sequential sub-agents) and Agent Teams (TeamCreate for parallel streams). Each implementer is given the plan + relevant rules as explicit context — sub-agents do NOT see CLAUDE.md.

---

## Iron Law (from `skills/subagent-driven-development/SKILL.md`)

> **LOAD THE SUBAGENT-DRIVEN-DEVELOPMENT SKILL BEFORE DISPATCHING ANY AGENTS**
> Without the skill, review stages are skipped, orphan files accumulate, and teams are never shut down.

---

## When to Use SDD vs Do-It-Yourself

| Situation | Approach |
|-----------|---------|
| 1–2 tasks, single domain | Do it yourself |
| 3+ tasks, same domain, sequential | Do it yourself with TaskCreate |
| 3+ tasks, different domains, can parallelize | SDD with Agent Teams |
| Specialized review needed after implementation | Dispatch single reviewer agent |
| Long autonomous iteration | Ralph loop (see `ralph-loop-autonomous.md`) |

---

## 3-Role Pipeline

```
Implementer Agent
    → writes code per plan task
    → marks task completed

Spec Reviewer Agent  (read-only)
    → verifies implementation matches plan
    → checks completeness: all WHEN/THEN scenarios

Quality Reviewer Agent  (read-only)
    → checks code standards
    → checks tests
    → checks security
```

---

## Phases

### Phase 1 — Load Skill and Prepare Context Packet

**Skill**: Load `subagent-driven-development`

Sub-agents do NOT inherit:
- CLAUDE.md
- `.claude/rules/` files
- conversation history
- lessons.md

**Context packet** (must be passed explicitly to every agent):
```
Context for sub-agent:
1. Plan: [paste or reference docs/plans/YYYY-MM-DD-feature.md]
2. Tech stack: [Java 21 / NestJS 11 / Python 3.14 / etc.]
3. Code standards: [paste relevant sections of code-standards.md]
4. Error handling rule: Every catch block MUST log and rethrow or return error state. No silent failures.
5. Iron Laws: [paste from the relevant skill's Iron Law]
6. Task: [specific task from the plan]
7. Files to create/modify: [explicit list with paths]
```

---

### Phase 2 — Agent Tool (Sequential)

For sequential tasks where each depends on the previous:

```python
# Dispatch Implementer
agent_result = Agent(
    subagent_type="nestjs-api",
    description="Implement orders module",
    prompt=f"""
    {context_packet}

    Task: Implement the Orders NestJS module per the plan.
    Files to create:
    - src/orders/orders.module.ts
    - src/orders/orders.controller.ts
    - src/orders/orders.service.ts
    - src/orders/dto/create-order.dto.ts
    - src/orders/orders.controller.spec.ts

    Iron Law: Load nestjs-api skill. Query Context7 before writing code.
    """
)

# After implementer completes, dispatch Spec Reviewer
spec_review = Agent(
    subagent_type="nestjs-reviewer",
    description="Review orders module against spec",
    prompt=f"""
    Review the implementation at src/orders/ against this plan:
    {plan_excerpt}

    Check: All endpoints from plan are implemented.
    Check: All DTO fields match spec.
    Check: Error handling follows no-silent-failure rule.
    Check: Tests exist for happy path and error cases.
    Output: PASS or list of FAIL items with file:line evidence.
    """
)
```

---

### Phase 3 — Agent Teams (Parallel)

For independent tasks that can run simultaneously:

```python
# Create team
team = TeamCreate(name="feature-implementation-team")

# Send parallel tasks to team members
SendMessage(team_id=team.id, to="implementer-1",
    message=f"""
    {context_packet}
    Task 1: Implement Order model and migrations.
    Files: src/orders/models/order.ts, prisma/migrations/add-orders.sql
    """)

SendMessage(team_id=team.id, to="implementer-2",
    message=f"""
    {context_packet}
    Task 2: Implement Auth module (parallel — no dependency on Task 1).
    Files: src/auth/auth.module.ts, src/auth/jwt.strategy.ts
    """)

# Wait for both to complete via TaskList
# Then dispatch reviewer
SendMessage(team_id=team.id, to="reviewer-1",
    message="Review both implementer outputs against the plan spec...")

# REQUIRED: Shut down team when done
SendMessage(team_id=team.id, to="implementer-1", message="shutdown_request")
SendMessage(team_id=team.id, to="implementer-2", message="shutdown_request")
SendMessage(team_id=team.id, to="reviewer-1", message="shutdown_request")
TeamDelete(team_id=team.id)
```

**Critical**: ALWAYS send `shutdown_request` to every teammate and call `TeamDelete`. Orphaned teams waste resources.

---

### Phase 4 — Orphan Prevention

After any SDD session:

```bash
# Check for agent-memory orphans
ls .claude/agent-memory/

# Consolidate useful findings
/promote-lessons
```

Orphaned `.claude/agent-memory/` files are NOT reliably read by future agents. Consolidate into `lessons.md` or skill reference files via `/promote-lessons`.

---

### Phase 5 — Quality Gate

After all implementers complete:

**Dispatch quality reviewers** (can be parallel):
```
[security-reviewer] — scan all new files for OWASP Top 10
[nestjs-reviewer] — check NestJS module correctness, tests, Prisma patterns
[silent-failure-hunter] — check for swallowed exceptions
```

**Gate**: All reviewers return PASS or all findings are CRITICAL and addressed.

---

## Sub-Agent Context Injection Template

Copy this template when dispatching any sub-agent:

```
## Context (Sub-Agent — you do not inherit CLAUDE.md)

**Project**: [project name]
**Tech Stack**: [stack details]
**Task**: [specific task description]
**Files to create/modify**: [explicit list]

**Non-negotiable rules**:
1. No silent failures: every catch block must log + rethrow or return error state
2. No hardcoded secrets — use environment variables
3. No deprecated APIs — query Context7 for current syntax
4. No console.log — use centralized logger
5. Load the [relevant] skill before writing code

**Plan reference**: [paste relevant plan section OR file path]
**Completion criteria**: [how to know when done]
```

---

## Quick Reference

| Scenario | Approach | Tools |
|----------|---------|-------|
| Sequential 3-role pipeline | Agent Tool (3 sequential dispatches) | `subagent-driven-development` skill |
| Parallel independent tasks | Agent Teams (TeamCreate) | TeamCreate, SendMessage, TaskList |
| Post-SDD cleanup | Promote orphan memories | `/promote-lessons` |
| Team shutdown | Send shutdown_request + TeamDelete | SendMessage, TeamDelete |

---

## Common Pitfalls

- **Not passing context** — agents that only get "implement the orders module" produce generic code; pass the full context packet
- **Not shutting down teams** — idle teammates accumulate; always `TeamDelete` after a session
- **Orphan agent-memory files** — agents write `.claude/agent-memory/` that other agents won't read; run `/promote-lessons`
- **Parallel when sequential** — tasks that depend on each other cause race conditions in parallel execution; map dependencies first
- **No spec reviewer** — implementers pass their own tests; the spec reviewer checks that tests match the plan, not just each other

## Related Workflows

- [`architecture-design.md`](architecture-design.md) — produces the plan SDD implements
- [`plan-review.md`](plan-review.md) — validates the plan before SDD starts
- [`ralph-loop-autonomous.md`](ralph-loop-autonomous.md) — autonomous iteration for single long tasks
