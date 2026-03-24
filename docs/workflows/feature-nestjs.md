# Feature Development — NestJS

> **When to use**: Building a new feature, module, or endpoint in a NestJS 11.x / Fastify / Prisma service
> **Time estimate**: 1–3 hours per feature
> **Prerequisites**: Approved spec in `docs/specs/`, approved plan in `docs/plans/` (see [`ideation-to-spec.md`](ideation-to-spec.md))

## Overview

Full NestJS feature lifecycle from scaffold (if new project) through TDD to reviewed, security-cleared PR. Uses Fastify adapter, Prisma ORM, Vitest for testing, and strict TypeScript throughout.

## Phases

### Phase 1 — Project Scaffold (new projects only)

**Trigger**: No existing NestJS project
**Command**: `/scaffold-nestjs-api [project-name]`
**10-step process** (from `commands/scaffold-nestjs-api.md`):
1. Create `.gitignore` first
2. Read `nestjs-api` skill and reference files
3. Create NestJS project via CLI
4. Install dependencies
5. Replace Jest with Vitest
6. Replace Express with Fastify adapter
7. Set up directory structure, config, Prisma, modules
8. Add global exception filter, validation pipe, logging interceptor, health module
9. Add infrastructure (Docker, docker-compose, .env)
10. Run `npm audit` and verify TypeScript

**Produces**: Production-ready NestJS API scaffold with Fastify + Prisma
**Gate**: `npm run build` succeeds, `npm test` passes

---

### Phase 2 — Load Skill

**Trigger**: About to implement any NestJS feature
**Action**: Load `nestjs-api` skill (`skills/nestjs-api/SKILL.md`)
**Also load**: `nestjs-coding-standard` skill
**MCP**: Context7 for NestJS/TypeScript current APIs, Prisma ORM docs

**11 core patterns** (from `skills/nestjs-api/SKILL.md:38-89`):
- Modules — `@Module({ imports, controllers, providers, exports })`
- Controllers — `@Controller('path')` with `@Get()`, `@Post()`, `@Body()`, `@Param()`
- Services — `@Injectable()`, constructor injection, no business logic in controllers
- DTOs — class-validator decorators, `class-transformer` for serialization
- Repositories — Prisma client wrapped in injectable service
- Error handling — `HttpException`, `ExceptionFilter`, problem details
- Config — `@nestjs/config` with validation schema
- Migrations — Prisma migrate, never edit existing migrations
- Guards — `@UseGuards()` on controller or route level
- Interceptors — logging, transform response shape
- Pipes — `ValidationPipe` globally, `ParseIntPipe` locally

**Gate**: Skill loaded, MCP queried

---

### Phase 3 — TDD: Write Failing Test First

**Iron Law** (from `skills/test-driven-development/SKILL.md:16-21`): `NO IMPLEMENTATION WITHOUT A FAILING TEST FIRST`

**For NestJS** (from `skills/test-driven-development/references/tdd-patterns-nestjs.md`):
- E2E tests with `supertest` + NestJS test module
- Unit tests with Vitest + `vi.mock()` for dependencies
- Prisma tests with in-memory SQLite or test database

**Red-Green-Refactor**:
1. **Red** — Write Vitest test that defines expected behaviour, confirm it fails
2. **Green** — Minimum code to pass
3. **Refactor** — Clean structure (tests still pass)

**Produces**: Failing test
**Gate**: Test fails for correct reason

---

### Phase 4 — Implement Feature

**Build order**:
1. Prisma schema update → `prisma/schema.prisma`
2. Prisma migration → `npx prisma migrate dev --name <feature>`
3. DTO (request) → `src/<module>/dto/<name>-request.dto.ts`
4. DTO (response) → `src/<module>/dto/<name>-response.dto.ts`
5. Repository → `src/<module>/<name>.repository.ts`
6. Service → `src/<module>/<name>.service.ts`
7. Controller → `src/<module>/<name>.controller.ts`
8. Module → `src/<module>/<name>.module.ts`
9. Register module in `AppModule`

**NestJS coding standards** (from `skills/nestjs-coding-standard/SKILL.md`):
- TypeScript strict mode — no `any`, no implicit `any`
- Immutability — DTOs with `readonly` fields
- Error handling — every `catch` must log and either rethrow or return error state
- DTOs — always use class-validator; never accept raw `object`
- Logging — NestJS `Logger` only; never `console.log()`
- Prisma 7.x — use `$transaction()` for multi-step writes

**Produces**: Working feature, `npm test` passes
**Gate**: All tests pass, `npm run build` clean

---

### Phase 5 — Review (run all 3 in parallel)

**Agent 1**: `nestjs-reviewer`
- Vibe: *"Module correctness is non-negotiable — wiring errors fail silently in prod"*
- Checks: module imports/exports correctness, JWT security, Prisma patterns, Vitest coverage, Resilience4j patterns

**Agent 2**: `silent-failure-hunter`
- Vibe: *"An empty catch block is not error handling — it's a lie to the operator"*
- Checks: swallowed exceptions, catch blocks returning empty arrays, missing NestJS Logger calls

**Agent 3**: `security-reviewer`
- Vibe: *"Assumes every input is hostile until the code proves otherwise"*
- Checks: input validation on all endpoints, SQL injection via raw Prisma queries, JWT validation, secrets in code

**Gate**: Zero CRITICAL findings; HIGH findings resolved or accepted

---

### Phase 6 — Pre-Commit Validation

**Command**: `/validate-changes`
**Agent**: `output-evaluator`
**Vibe**: *"Defaults to NEEDS_REVIEW — APPROVE requires evidence, not optimism"*

**Also run**: `npm audit --audit-level=high` — zero high/critical CVEs

**Verdicts**:
- `APPROVE` → proceed to PR
- `NEEDS_REVIEW` / `REJECT` → fix issues, re-run

**Gate**: `APPROVE` verdict + zero high CVEs

---

### Phase 7 — PR Review

**Command**: `/review-pr`
**6 roles** (from `commands/review-pr.md:15-20`): comment-analyzer + pr-test-analyzer + silent-failure-hunter + type-design-analyzer + code-reviewer + code-simplifier

**Gate**: All CRITICAL + HIGH resolved

---

## Quick Reference

| Phase | What to Run | Produces | Gate |
|-------|-------------|----------|------|
| 1 — Scaffold | `/scaffold-nestjs-api` | Full NestJS skeleton | `npm test` passes |
| 2 — Load skill | `nestjs-api` + `nestjs-coding-standard` skills | Pattern reference | MCP queried |
| 3 — TDD | Write failing Vitest test | Failing test | Fails for right reason |
| 4 — Implement | schema → DTO → repo → service → controller → module | Working code | `npm test` passes |
| 5 — Review | `nestjs-reviewer` + `silent-failure-hunter` + `security-reviewer` | Findings | Zero CRITICAL |
| 6 — Pre-commit | `/validate-changes` | APPROVE/NEEDS_REVIEW/REJECT | APPROVE |
| 7 — PR | `/review-pr` | 6-role review | CRITICAL+HIGH resolved |

---

## Common Pitfalls

- **Not registering the module** — adding a service without importing its module in `AppModule` causes silent `undefined` injection
- **Editing Prisma migrations** — never edit a migration that has been applied; create a new one
- **Using `console.log()`** — use NestJS `Logger`; raw console leaks to prod and bypasses log aggregation
- **DTOs without class-validator** — `ValidationPipe` does nothing without decorators on the DTO class
- **Missing `@Injectable()`** — NestJS DI silently fails; always decorate services
- **Raw Prisma in controllers** — always go through repository service; never access `prisma` directly from a controller

## Related Workflows

- [`ideation-to-spec.md`](ideation-to-spec.md) — spec and plan before implementation
- [`database-schema-design.md`](database-schema-design.md) — Prisma schema changes
- [`security-audit.md`](security-audit.md) — deeper security audit
- [`pr-shipping.md`](pr-shipping.md) — PR lifecycle after review
