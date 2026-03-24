# NFR Requirements Elicitation

> **When to use**: During Phase 2 (Details) or Phase 3 (Edge Cases) of a feature-forge interview — any feature with >100 users, any API endpoint, or any shared service.
> **Time estimate**: 10–30 min depending on compliance scope
> **Prerequisites**: Feature-forge interview started (Phase 1 Discovery complete)

## Overview

Structured elicitation of Non-Functional Requirements (NFRs) before writing the spec. Vague NFRs (e.g., "it should be fast", "high availability") become untestable acceptance criteria and incorrect architecture choices. This workflow replaces ad-hoc NFR gathering with a structured question-and-checklist approach integrated into the feature-forge interview.

---

## When to Trigger

Run NFR elicitation when ANY of the following are true:

| Trigger | Reason |
|---------|--------|
| Feature has >100 concurrent users | Horizontal scaling and latency targets must be explicit |
| Feature exposes an API endpoint | SLA, rate limiting, and error contract must be defined |
| Feature stores or processes PII, financial, or health data | Compliance controls depend on data classification |
| Feature is a shared service (used by 2+ other services) | Reliability and versioning NFRs affect all consumers |
| Feature involves file upload, export, or batch processing | Async vs sync choice, size limits, and retention policy |
| Feature has a payment, auth, or audit trail component | Security and observability requirements differ from standard |

If none of the above apply (e.g., purely internal, low-traffic admin utility), skip to the overconfidence check: ask "Are there any latency, availability, or compliance requirements?" If the answer is "none", document that explicitly in the spec.

---

## Phases

### Phase 1 — Load NFR Checklist

**When**: After Phase 1 (Discovery) of the feature-forge interview is complete.
**Action**: Load `skills/feature-forge/references/nfr-checklist.md`

The checklist organizes NFRs into 7 categories:
1. Performance — latency, throughput, caching
2. Scalability — load, growth, rate limiting
3. Availability / Reliability — SLA, RTO, RPO, failover
4. Security — data classification, auth, compliance
5. Observability and Logging — structured logs, tracing, alerting
6. Maintainability — testability, versioning, feature flags
7. Compliance and Data Residency — GDPR, HIPAA, PCI, data location

---

### Phase 2 — Elicit NFRs Using AskUserQuestions

**Goal**: Replace vague NFR statements with measurable targets.

Use `AskUserQuestions` for structured choices (latency tier, SLA tier, data sensitivity). Use open-ended follow-up for anything that cannot be predetermined.

**Overconfidence check — mandatory before proceeding**:

| Vague Answer | Required Follow-Up |
|-------------|-------------------|
| "it should be fast" | "What is the p95 latency target? For how many concurrent users?" |
| "high availability" | "What uptime SLA? 99.9%? 99.99%?" |
| "it should scale" | "What is the load today? In 12 months?" |
| "it should be secure" | "What data class? PII? PCI? What compliance standard?" |

Do not proceed with vague answers. Mark them as OPEN NFR items — implementation cannot start until resolved.

**Batch questions by domain** (see `references/nfr-checklist.md` for complete question sets):

- Batch 1: Performance and scale (latency tier, concurrent users, throughput)
- Batch 2: Availability (SLA tier, RTO, RPO, failover behavior)
- Batch 3: Data and compliance (data classification, compliance standards)
- Batch 4: Operations (log retention, alerting thresholds, feature flag needed?)

---

### Phase 3 — Map NFR Answers to Acceptance Criteria

**Goal**: Every NFR answer becomes a testable EARS-format acceptance criterion in the spec.

Transform raw NFR answers before writing the spec:

| NFR Answer | Acceptance Criterion (EARS format) |
|------------|-----------------------------------|
| "< 500ms p95 response" | WHEN 100 concurrent users submit requests THE SYSTEM SHALL respond within 500ms at the p95 percentile |
| "99.99% uptime" | WHEN a single instance fails THE SYSTEM SHALL failover within 30 seconds without user-visible errors |
| "PII must not appear in logs" | WHEN any request containing user email is processed THE SYSTEM SHALL redact the email field in all log output |
| "GDPR right to erasure" | WHEN a user submits a deletion request THE SYSTEM SHALL remove all associated PII within 30 days |
| "Rate limit per user" | WHEN a user exceeds 100 requests per minute THE SYSTEM SHALL return HTTP 429 with a Retry-After header |

These criteria appear in the spec under section 3: Non-Functional Requirements.

---

### Phase 4 — NFR → Implementation Plan Integration

**Goal**: NFR decisions drive concrete implementation tasks in the TODO checklist.

Map each NFR category to implementation tasks:

| NFR Category | Typical Implementation Tasks |
|-------------|------------------------------|
| Performance | Add caching layer, query index review, async offload to queue |
| Scalability | Stateless service design, connection pool sizing, rate limiter middleware |
| Availability | Health check endpoint, circuit breaker config, retry policy |
| Security | Input validation, auth middleware, secrets in env (not code) |
| Observability | Structured logging setup, trace ID propagation, alert rule config |
| Maintainability | OpenAPI spec, feature flag config, migration reversibility check |
| Compliance | PII masking in logs, data retention policy, GDPR erasure endpoint |

---

### Phase 5 — Handoff to Plan Review

**Trigger**: All NFRs documented and mapped to acceptance criteria
**Action**: Include NFR section in spec before handing to `/plan-review`

The `plan-mode-review` skill Phase 4 checks Performance NFRs explicitly. By completing this workflow first, the plan reviewer has concrete targets to validate against rather than assumed ones.

**Gate**: All NFR fields in the spec template are populated — no field reads "TBD" or "to be determined".

---

## Output: NFR Section in Spec Template

The spec's NFR section must include:

```markdown
## Non-Functional Requirements

### Performance
- Latency: p95 < [Xms] for [Y] concurrent users
- Throughput: [N] requests/second sustained
- Async: [yes/no — operation runs in background queue]

### Availability
- Uptime SLA: [99.X%]
- RTO: [X minutes]
- RPO: [X minutes of data loss acceptable / zero]

### Security
- Data classification: [PII / financial / health / non-sensitive]
- Auth: [public / authenticated / role-based: list roles]
- Compliance: [GDPR / HIPAA / PCI / none]

### Observability
- Events logged: [list business events that must appear in logs]
- PII masking: [list fields that must be redacted]
- Alerting: [conditions that trigger on-call alert]
- Log retention: [N days / N years]

### Open NFR Items
- [Any NFR that could not be resolved during the interview — MUST be resolved before implementation starts]
```

---

## Quick Reference

| Phase | Action | Tool | Gate |
|-------|--------|------|------|
| 1 — Load | Load `nfr-checklist.md` | Read | Checklist loaded |
| 2 — Elicit | Structured NFR questions | AskUserQuestions | No vague answers remain |
| 3 — Map | Convert to EARS criteria | Write spec | All NFRs testable |
| 4 — Plan | Map NFRs to TODO tasks | Implementation checklist | Every NFR has an owner task |
| 5 — Handoff | Pass to plan-review | `/plan-review` | Spec NFR section complete |

---

## Common Pitfalls

- **Accepting "it should be fast"** — always follow up with a number. "Fast" is not an acceptance criterion.
- **Skipping compliance questions** — data classification determines entire security control set; cannot be assumed.
- **NFRs added after spec is approved** — NFRs change the architecture, not just the implementation. Elicit before writing the spec, not after.
- **Treating SLA as a preference** — 99.9% vs 99.99% is the difference between a single instance and an HA cluster. Confirm explicitly.
- **Missing the open NFR section** — if any NFR is unresolved, mark it explicitly. Silent omission causes it to be treated as "no requirement" during implementation.

## Related Workflows

- [`ideation-to-spec.md`](ideation-to-spec.md) — full feature elicitation workflow that includes NFR phase
- [`feature-java-spring.md`](feature-java-spring.md) — NFR targets map to Spring WebFlux configuration
- [`feature-nestjs.md`](feature-nestjs.md) — NFR targets map to Fastify and NestJS middleware config
- [`architecture-design.md`](architecture-design.md) — availability and scalability NFRs often require architecture changes before implementation
