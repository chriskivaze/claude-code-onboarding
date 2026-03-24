# API Design

> **When to use**: Before writing any controller, route handler, or service layer. Run this workflow when designing a new REST API or adding endpoints to an existing service.
> **Prerequisites**: Feature spec or requirements defined. Tech stack chosen (FastAPI / NestJS / Spring WebFlux).

## Overview

Design-first REST API workflow using the `api-design-principles` skill for conventions and checklist, then `openapi-spec-generation` for the contract, before any implementation begins.

---

## Commands and Skills

| Skill | Scope | What it produces |
|-------|-------|-----------------|
| `api-design-principles` | REST design phase | URL structure, status codes, pagination choice, caching, idempotency, bulk ops |
| `openapi-spec-generation` | Documentation phase | OpenAPI 3.1 spec, SDK generation, developer guide |
| `architecture-design` | System design | C4 diagrams, API contracts, deployment topology |
| `java-spring-api` | Implementation | Spring WebFlux controllers, services, tests |
| `nestjs-api` | Implementation | NestJS modules, controllers, DTOs |
| `python-dev` | Implementation | FastAPI routes, Pydantic models, tests |

---

## Phases

### Phase 1 — Pre-Design Checklist

Before writing any endpoint definition:

1. Load skill: `api-design-principles`
2. Read `assets/api-design-checklist.md` — run through all sections:
   - Resource naming conventions
   - HTTP method assignments
   - Status code mapping
   - Pagination strategy (offset vs cursor — decide now, not after)
   - Versioning strategy (URL path recommended: `/api/v1/`)
   - Rate limiting requirements
   - Idempotency requirements (payment, order, notification endpoints)
   - Caching requirements (read-heavy resources)
   - Bulk operation requirements
3. Document decisions — record pagination strategy, versioning, and idempotency choices before moving to Phase 2

---

### Phase 2 — REST Design

Apply design principles from `reference/rest-design-principles.md`:

**URL Design:**
```
# Correct
GET  /api/v1/users
POST /api/v1/users
GET  /api/v1/users/{id}
GET  /api/v1/users/{id}/orders   # max 2 levels

# Incorrect
GET  /api/getUsers
GET  /api/user
POST /api/createOrder
```

**Status Code Decisions:**

| Operation | Success | Error Cases |
|-----------|---------|-------------|
| GET collection | 200 | — |
| GET single | 200 | 404 |
| POST | 201 + Location | 400, 409, 422 |
| PATCH | 200 | 400, 404, 422 |
| DELETE | 204 | 404, 409 |
| Bulk | 207 | — (per-item errors in body) |

**Key patterns to decide during design (not implementation):**
- Will this endpoint need ETags? → Add Cache-Control + ETag to design
- Is this a mutation that could be retried? → Add Idempotency-Key header requirement
- Does this return a list? → Choose pagination strategy and document it
- Will this be called from a browser? → CORS configuration needed

---

### Phase 3 — OpenAPI Spec

Once the design is agreed, generate the contract with `openapi-spec-generation`:

**Design-first approach (recommended):**
```bash
# Start from the skeleton template
# Reference: .claude/skills/openapi-spec-generation/reference/openapi-skeleton-template.md

# Validate
spectral lint openapi.yaml

# Preview
redocly preview-docs openapi.yaml
```

**Code-first (for existing code):**
```bash
# FastAPI
python -c "import json; from main import app; print(json.dumps(app.openapi(), indent=2))" > openapi.json

# Spring Boot
curl http://localhost:8080/v3/api-docs > openapi.json

# NestJS / tsoa
npx tsoa spec
```

---

### Phase 4 — Implementation

With the API contract defined, use the stack-specific skill:

| Stack | Skill | Command |
|-------|-------|---------|
| Spring Boot WebFlux | `java-spring-api` | `/scaffold-spring-api` |
| NestJS 11.x | `nestjs-api` | `/scaffold-nestjs-api` |
| Python FastAPI | `python-dev` | `/scaffold-python-api` |

The spec is the contract — implementation must match it. Run `spectral lint` in CI to detect drift.

---

## Quick Reference

| Phase | Skill/Command | Gate |
|-------|--------------|------|
| 1 — Checklist | `api-design-principles` → checklist | All checklist items reviewed |
| 2 — REST Design | `api-design-principles` → principles | URL + status codes + patterns decided |
| 3 — OpenAPI | `openapi-spec-generation` | Spec validated with `spectral lint` |
| 4 — Implementation | Stack-specific skill | Code matches spec |

---

## Common Pitfalls

- **Starting implementation before choosing pagination strategy** — adding cursor pagination after offset pagination is in production is a breaking change
- **Using 403 for missing auth token** — 403 means the token is valid but insufficient; missing token is 401
- **No idempotency on POST payment/order endpoints** — retries cause duplicate charges
- **Wildcard CORS in production** (`allow_origins=["*"]`) — blocks credentialed requests and exposes the API to any origin
- **No Location header on 201** — clients cannot discover the new resource URL without it

---

## Related Workflows

- [`feature-python-fastapi.md`](feature-python-fastapi.md) — Full Python FastAPI feature lifecycle
- [`feature-java-spring.md`](feature-java-spring.md) — Full Spring Boot feature lifecycle
- [`feature-nestjs.md`](feature-nestjs.md) — Full NestJS feature lifecycle
- [`architecture-design.md`](architecture-design.md) — System-level design before API design
- [`security-audit.md`](security-audit.md) — Security review of API endpoints
