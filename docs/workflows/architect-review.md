# Architect Review Workflow

Use when reviewing system architecture or major design changes before implementation.

## When to Use

- Pre-implementation architecture gate for HIGH-impact changes
- Reviewing microservice boundaries or bounded contexts
- Evaluating distributed system design (Saga, Outbox, CQRS, event sourcing)
- Assessing scalability or resilience gaps in an existing system
- Reviewing architectural trade-offs before accepting an ADR

## How to Trigger

```bash
# Via skill
/architect-review [system or design doc]

# Via agent (dispatched automatically by code-reviewer for architecture concerns)
# Agent type: architect
```

## Workflow

```
1. Load skill
   └── /architect-review [target]

2. Provide context
   └── System goals, constraints, current state, proposed change

3. Review output sections
   ├── Impact Assessment (HIGH/MEDIUM/LOW per concern)
   ├── Pattern Compliance (✅ correct / ❌ violations)
   ├── Anti-Patterns Detected (with concrete fixes)
   └── Recommendations (prioritized, actionable)

4. Act on findings
   ├── HIGH findings → must resolve before implementation
   ├── MEDIUM findings → resolve or document accepted risk
   ├── LOW findings → optional, add to tech debt backlog
   └── ADR required → delegate to architecture-decision-records skill

5. Gate check (for HIGH-impact changes)
   └── All HIGH findings resolved OR explicitly accepted with risk documented
```

## Integration with Other Skills

| Skill | When to Chain |
|-------|---------------|
| `architecture-design` | Design first, review second |
| `architecture-decision-records` | Generate ADR for irreversible decisions found in review |
| `ddd-architect` | Deep DDD analysis if bounded context violations found |
| `plan-mode-review` | 5-phase gate review (includes Phase 1 architecture review) |
| `threat-modeling` | Security architecture concerns found during review |

## Difference from plan-mode-review

`plan-mode-review` Phase 1 covers architecture as 1 of 5 review phases — shallow pass.

`architect-review` is a dedicated deep-dive:
- Full distributed systems assessment (Saga, Outbox, service mesh)
- Anti-pattern catalog with concrete fixes
- Cloud-native and security architecture evaluation
- Appropriate for pre-implementation gate on HIGH-impact changes

## Example: NestJS Payment Service Review

```
/architect-review docs/plans/2026-03-15-payments.md

Output:
  Impact: HIGH — cross-service saga coordination, financial consistency required

  Pattern Compliance:
  ✅ Repository pattern present (src/payments/repositories/)
  ❌ No Outbox pattern — event publishing not atomic with DB write
  ❌ No circuit breaker for Stripe calls

  Anti-Patterns Detected:
  - Fat service: PaymentService handles orchestration + domain logic + notification

  Recommendations:
  1. Add Transactional Outbox (NestJS + Prisma implementation)
  2. Add circuit breaker: nestjs-resilience4j or @nestjs/axios with retry
  3. Split: PaymentDomainService (business rules) + PaymentOrchestratorService (saga)

  ADRs Required:
  - ADR for saga vs. choreography choice
  - ADR for Outbox implementation approach
```

## Tech Stack Applicability

| Stack | Coverage |
|-------|----------|
| Java / Spring Boot WebFlux | ✅ Full (Resilience4j, Spring Events, Saga) |
| NestJS / TypeScript | ✅ Full (Prisma Outbox, nestjs-resilience4j) |
| Python / FastAPI | ✅ Full (SQLAlchemy Outbox, Celery, tenacity) |
| Angular SPA | ✅ Frontend architecture (state management, lazy loading) |
| Flutter | ✅ Mobile architecture (Riverpod, Clean Architecture) |
| PostgreSQL | ✅ Data architecture (schema isolation, replication) |
