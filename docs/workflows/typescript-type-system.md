# TypeScript Type System Workflow

> **When to use**: Strengthening TypeScript type safety, designing advanced type patterns, resolving type errors, setting up TypeScript infrastructure for Angular 21.x, NestJS 11.x, or MCP Builder projects
> **Prerequisites**: TypeScript project exists with `tsconfig.json` configured

## Overview

Three-skill TypeScript type system pipeline covering type-level programming patterns (generics, conditional types, mapped types), architecture design (decorator patterns, strict config), and infrastructure (monorepo, build performance, ESM/CJS interop). Applies to Angular 21.x, NestJS 11.x, and MCP Builder TypeScript projects only — not Flutter, Java, or Python.

## Skill Selection Guide

| Task | Primary Skill | When to Add Others |
|------|---|---|
| Generic DTOs, conditional types, mapped types, utility types | **typescript-advanced-types** | + nestjs-api (NestJS) or angular-spa (Angular) |
| Strict type config, decorator design, module type contracts | **typescript-pro** | + nestjs-coding-standard |
| Monorepo project refs, build perf, ESM/CJS, JS→TS migration | **typescript-expert** | + framework skill |
| All three together | Start with **typescript-advanced-types** | Then pro (architecture), then expert (infra) |

---

## Phase 1 — Type Pattern Implementation (typescript-advanced-types)

**Trigger**: Need type-safe generics, conditional types, mapped types, or custom utility types
**Skill**: `typescript-advanced-types`
**Reference**: `resources/implementation-playbook.md` (717 lines of patterns)

### When to load this skill
- Building type-safe DTO hierarchies (NestJS `DeepPartial<CreateUserDto>`)
- Creating generic service factories (Angular type-safe HTTP clients)
- Designing Zod schemas with branded types (MCP Builder tool definitions)
- Implementing `DeepPartial`, `DeepReadonly`, `PickByType`, or `Branded` utility types
- Using template literal types for string-based APIs

### Workflow
1. Load `typescript-advanced-types` skill
2. Open `resources/implementation-playbook.md` for the relevant pattern section
3. Identify which pattern applies: Generics / Conditional Types / Mapped Types / Template Literals / Utility Types
4. Implement with workspace framework (nestjs-api or angular-spa patterns)
5. Validate: `npx tsc --noEmit` — zero errors required

**Gate**: `tsc --noEmit` passes, no `any` types introduced

---

## Phase 2 — Type Architecture Design (typescript-pro)

**Trigger**: Designing shared type contracts, strict configuration, or decorator patterns
**Skill**: `typescript-pro`

### When to load this skill
- Architecting type-safe NestJS configuration (branded Port/Environment types)
- Designing custom NestJS decorators with metadata type extraction
- Creating module-level type contracts that prevent implementation leakage
- Establishing enterprise-grade shared type libraries across services

### Workflow
1. Load `typescript-pro` skill
2. Define runtime targets and strictness requirements
3. Model types and contracts for critical surfaces
4. Implement with compiler and linting safeguards
5. Validate build performance and developer ergonomics

**Gate**: All existing tests pass, no `strict: false` introduced

---

## Phase 3 — TypeScript Infrastructure (typescript-expert)

**Trigger**: Monorepo setup, build performance issues, ESM/CJS interop, or JS→TS migration
**Skill**: `typescript-expert`
**Reference files**: `references/tsconfig-strict.json`, `references/ts_diagnostic.py`, `references/typescript-cheatsheet.md`

### When to load this skill
- Setting up monorepo with shared types between Angular + NestJS (`composite: true`, project references)
- TypeScript build is slow — run `npx tsc --extendedDiagnostics` first
- MCP Builder (ESM) needs to interop with NestJS (CommonJS)
- Migrating JavaScript service to TypeScript strict mode
- Authoring `.d.ts` declaration files for untyped packages

### Workflow
1. Load `typescript-expert` skill
2. Analyze project setup: read `tsconfig.json`, `package.json`, detect monorepo
3. Identify problem category: perf / interop / migration / monorepo
4. Apply solution strategy from `references/tsconfig-strict.json` or run `scripts/ts_diagnostic.py`
5. Validate:
   ```bash
   npm run -s typecheck || npx tsc --noEmit
   npm test -s || npx vitest run --reporter=basic --no-watch
   npm run -s build
   ```

**Gate**: `tsc --noEmit` passes, build succeeds, test suite green

---

## Common Combinations

### NestJS: Type-safe DTO Hierarchy
1. **typescript-advanced-types** — `DeepPartial<T>`, `PickByType<T,U>`, mapped type Update DTOs
2. **nestjs-coding-standard** — Enforce `strict: true`, `unknown` over `any`
3. **nestjs-api** — Wire into controllers and services

### Angular: Generic HTTP Service Factory
1. **typescript-advanced-types** — Conditional + mapped types for `ApiClient<T>`
2. **angular-spa** — Integrate into Angular service layer
3. **browser-testing** — Verify type-safe service in E2E tests

### Monorepo: Shared Types (Angular + NestJS)
1. **typescript-expert** — Set up project references, `composite: true`, shared `tsconfig.base.json`
2. **typescript-advanced-types** — Design the shared type library (DTOs, API responses)
3. **nestjs-api** + **angular-spa** — Consume shared types in each framework

### MCP Builder: Type-safe Tool Definitions
1. **typescript-advanced-types** — Branded types for tool inputs, Zod schema inference
2. **mcp-builder** — Wire into MCP server tool registration
3. **typescript-expert** — Resolve ESM/CJS interop if needed

---

## Troubleshooting

| Problem | Skill | Action |
|---------|-------|--------|
| "Type instantiation is excessively deep" | typescript-expert | Break recursion, use type aliases, split unions |
| "The inferred type of X cannot be named" | typescript-expert | Export the type explicitly or use `ReturnType<>` |
| Slow `tsc` (>5s on medium project) | typescript-expert | Run `--extendedDiagnostics`, enable incremental |
| `any` types leaking through generics | typescript-advanced-types | Apply `unknown` + type guards pattern |
| DTO update type requires 20 `@IsOptional()` | typescript-advanced-types | Use `DeepPartial<T>` mapped type |
| NestJS + Angular can't share DTOs | typescript-expert | Monorepo project references |
| MCP SDK (ESM) breaks NestJS (CJS) build | typescript-expert | ESM/CJS interop patterns |

---

## Related Workflows

- [`feature-nestjs.md`](feature-nestjs.md) — Full NestJS feature with TypeScript
- [`feature-angular-spa.md`](feature-angular-spa.md) — Angular SPA feature development
- [`mcp-server-setup.md`](mcp-server-setup.md) — MCP Builder with TypeScript
