# Feature Development — Java / Spring Boot WebFlux

> **When to use**: Building a new feature or endpoint in a Java 21 / Spring Boot 3.5.x WebFlux service
> **Time estimate**: 1–4 hours per feature depending on complexity
> **Prerequisites**: Approved spec in `docs/specs/`, approved plan in `docs/plans/` (see [`ideation-to-spec.md`](ideation-to-spec.md))

## Overview

Full Java/Spring Boot WebFlux feature lifecycle from scaffold (if new project) through TDD implementation to reviewed, security-cleared PR. Uses reactive patterns throughout — no blocking calls in the pipeline.

## Phases

### Phase 1 — Project Scaffold (new projects only)

**Trigger**: No existing Spring Boot project
**Command**: `/scaffold-spring-api [project-name]`
**10-step process** (from `commands/scaffold-spring-api.md`):
1. Create `.gitignore` first
2. Read `java-spring-api` and `java-coding-standard` skills
3. Create Maven project via Spring Initializr or manual `pom.xml`
4. Set up package structure
5. Add `application.yml` with R2DBC + Flyway config
6. Create HealthController, sample entity, DTO, repository, service, controller
7. Add `GlobalExceptionHandler` with ProblemDetail (RFC 9457)
8. Add Flyway migration and Docker Compose
9. Add integration test with `WebTestClient`
10. Run `dependency-check` and verify compilation

**Produces**: Production-ready Spring Boot WebFlux project scaffold
**Gate**: `./mvnw compile` succeeds, `./mvnw test` passes

---

### Phase 2 — Load Skill (every feature, not just new projects)

**Trigger**: About to implement any Spring Boot feature
**Action**: Load `java-spring-api` skill (`skills/java-spring-api/SKILL.md`)
**Also load**: `java-coding-standard` skill for naming and pattern rules
**MCP**: Query Context7 for current Spring Boot APIs before writing code

**Key patterns from skill** (`skills/java-spring-api/SKILL.md:45-72`):
- DTOs — records with `@NotNull`, `@Size` validation
- Entities — immutable R2DBC entities with `@Table`
- Repositories — `ReactiveCrudRepository` extensions
- Services — `Mono<T>` / `Flux<T>` return types, no `.block()`
- Controllers — `@RestController` with `Mono<ResponseEntity<T>>`
- Error handling — `GlobalExceptionHandler` with `ProblemDetail` (RFC 9457)

**Gate**: Skill loaded, MCP queried for any API signatures being used

---

### Phase 3 — TDD: Write Failing Test First

**Trigger**: Skill loaded, implementation about to start
**Skill**: `test-driven-development` (`skills/test-driven-development/SKILL.md`)
**Iron Law** (from `skills/test-driven-development/SKILL.md:16-21`): `NO IMPLEMENTATION WITHOUT A FAILING TEST FIRST`

**For Java/Spring** (from `skills/test-driven-development/references/tdd-patterns-java.md`):
- Integration tests with `@SpringBootTest` + `WebTestClient`
- Repository tests with `@DataR2dbcTest`
- Unit tests with `Mockito` for service layer

**Red-Green-Refactor cycle**:
1. **Red** — Write test that defines the behaviour, confirm it fails
2. **Green** — Write minimum code to make it pass
3. **Refactor** — Clean up without changing behaviour (tests still pass)

**Produces**: Failing test that will pass when feature is complete
**Gate**: Test runs and fails for the right reason (not compile error)

---

### Phase 4 — Implement Feature

**Trigger**: Failing test exists
**Build order** (per approved plan from `docs/plans/`):
1. Flyway migration (if schema change) → `src/main/resources/db/migration/V<N>__<name>.sql`
2. Entity → `src/main/java/.../domain/<Name>.java`
3. DTO (request/response records) → `src/main/java/.../dto/`
4. Repository → `src/main/java/.../repository/<Name>Repository.java`
5. Service → `src/main/java/.../service/<Name>Service.java`
6. Controller → `src/main/java/.../controller/<Name>Controller.java`
7. Exception types if needed → `src/main/java/.../exception/`

**Java coding standards** (from `skills/java-coding-standard/SKILL.md:28-37`):
- Immutability — use records for DTOs and value objects
- Optional — never return `null`; use `Optional<T>` or `Mono.empty()`
- No blocking — `.block()` is forbidden in reactive chains
- Logging — use `log.info()/warn()/error()` with structured context, never `System.out.println()`
- Null checks — use `Objects.requireNonNull()` at service boundaries

**Produces**: Working feature implementation, tests pass
**Gate**: `./mvnw test` passes, no compilation warnings

---

### Phase 5 — Review (run all 3 in parallel)

**Trigger**: Tests pass, implementation complete
**Run these agents concurrently**:

**Agent 1**: `spring-reactive-reviewer`
- Vibe: *"One blocking call in a reactive chain kills the whole thread pool"*
- Checks: blocking calls in reactive chains, Resilience4j patterns, R2DBC usage, WebTestClient coverage
- Model: sonnet

**Agent 2**: `silent-failure-hunter`
- Vibe: *"An empty catch block is not error handling — it's a lie to the operator"*
- Checks: swallowed exceptions, catch blocks returning empty, missing error logging
- Model: sonnet

**Agent 3**: `security-reviewer`
- Vibe: *"Assumes every input is hostile until the code proves otherwise"*
- Checks: input validation, SQL injection (parameterized queries), auth on endpoints, secrets in code
- Model: opus

**Produces**: Review findings by severity (CRITICAL / HIGH / MEDIUM / LOW)
**Gate**: Zero CRITICAL findings; HIGH findings resolved or explicitly accepted with justification

---

### Phase 6 — Pre-Commit Validation

**Trigger**: All review findings resolved
**Command**: `/validate-changes`
**Agent**: `output-evaluator` (haiku model, LLM-as-judge)
**Vibe**: *"Defaults to NEEDS_REVIEW — APPROVE requires evidence, not optimism"*

**Verdicts**:
- `APPROVE` → proceed to PR
- `NEEDS_REVIEW` → fix flagged issues, re-run
- `REJECT` → do not commit; address all issues first

**Also run**: `./mvnw dependency-check:check` — zero HIGH/CRITICAL CVEs
**Gate**: `output-evaluator` returns `APPROVE`

---

### Phase 7 — PR Review

**Trigger**: Changes committed, PR opened
**Command**: `/review-pr`
**6 roles dispatched** (from `commands/review-pr.md:15-20`):

| Role | Agent | Focus |
|------|-------|-------|
| Comment accuracy | `comment-analyzer` | Outdated Javadoc, misleading comments |
| Test coverage | `pr-test-analyzer` | Behavioral gaps, missing edge case tests |
| Error handling | `silent-failure-hunter` | Swallowed exceptions, silent fallbacks |
| Type design | `type-design-analyzer` | Records allowing invalid states |
| Code quality | `code-reviewer` | General quality, patterns, DRY violations |
| Simplification | `code-simplifier` | Over-engineering, unnecessary abstraction |

**Produces**: Aggregated review findings
**Gate**: All CRITICAL + HIGH findings resolved before merge

---

## Quick Reference

| Phase | What to Run | Produces | Gate |
|-------|-------------|----------|------|
| 1 — Scaffold | `/scaffold-spring-api` | Full project skeleton | `./mvnw test` passes |
| 2 — Load skill | Load `java-spring-api` + `java-coding-standard` | Pattern reference loaded | MCP queried |
| 3 — TDD | Write failing test first | Failing test | Test fails for right reason |
| 4 — Implement | Build entity→DTO→repo→service→controller | Working code | `./mvnw test` passes |
| 5 — Review | `spring-reactive-reviewer` + `silent-failure-hunter` + `security-reviewer` (parallel) | Findings report | Zero CRITICAL |
| 6 — Pre-commit | `/validate-changes` | APPROVE/NEEDS_REVIEW/REJECT | APPROVE verdict |
| 7 — PR | `/review-pr` | 6-role review | All CRITICAL+HIGH resolved |

---

## Common Pitfalls

- **Blocking calls** — `.block()`, `Thread.sleep()`, JDBC in a WebFlux chain cause thread pool starvation
- **Skipping Flyway** — schema changes without migrations break other environments
- **DTOs as mutable classes** — use `record` for immutability; DTOs should not be JPA entities
- **Missing `GlobalExceptionHandler`** — uncaught exceptions leak stack traces; always handle at the controller boundary
- **No RFC 9457 ProblemDetail** — the project standard for error responses; don't return raw strings
- **`.subscribe()` without error handler** — silent failure in reactive chains; always handle `onError`

## Related Workflows

- [`ideation-to-spec.md`](ideation-to-spec.md) — spec and plan before implementation
- [`database-schema-design.md`](database-schema-design.md) — schema design if feature needs DB changes
- [`security-audit.md`](security-audit.md) — deeper security audit before production
- [`pr-shipping.md`](pr-shipping.md) — complete PR lifecycle after review
- [`deployment-ci-cd.md`](deployment-ci-cd.md) — deploying to Cloud Run after merge
