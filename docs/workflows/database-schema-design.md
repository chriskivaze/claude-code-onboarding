# Database Schema Design

> **When to use**: Designing a new database schema, adding tables, or modifying existing schema for a feature
> **Time estimate**: 1–3 hours for a new domain; 30 min for additive changes
> **Prerequisites**: Feature spec in `docs/specs/` with entity and access pattern requirements

## Overview

Production-ready database schema design from requirements to reviewed, reversible Flyway migrations. Uses ERD diagrams, index strategy for actual query patterns, and mandatory postgresql-database-reviewer validation before any migration runs.

---

## Phases

### Phase 1 — Load Skill

**Action**: Load `database-schema-designer` skill (`skills/database-schema-designer/SKILL.md`)
**4-phase skill structure** (from `skills/database-schema-designer/SKILL.md:44-73`):
1. Analyze — understand domain, entities, relationships, access patterns
2. Design — normalization, constraints, data types
3. Optimize — indexes for actual query patterns
4. Migrate — reversible migrations with up + down

**Reference files in skill**:
- `reference/normalization-guide.md` — 3NF rules, when to denormalize
- `reference/indexing-strategy.md` — B-tree vs GiST vs GIN, composite index order
- `reference/data-types-reference.md` — PostgreSQL type selection
- `reference/constraints-and-relationships.md` — FK, CHECK, UNIQUE patterns
- `reference/migration-patterns.md` — expand-contract, backward-compatible changes
- `reference/nosql-design-patterns.md` — Firestore collection structure

---

### Phase 2 — Requirements Analysis

**Trigger**: Spec confirmed, about to design schema
**Questions to answer before touching SQL**:
1. What entities exist? (nouns in the domain)
2. What are the relationships? (one-to-many, many-to-many)
3. What queries will run? (determines indexes)
4. What's the expected data volume? (determines partitioning strategy)
5. What changes frequently vs rarely? (determines normalization vs denormalization)
6. Is Firestore also needed? (mobile real-time sync)

**Output**: Entity list with attributes and relationship descriptions
**Gate**: Access patterns documented before starting ERD

---

### Phase 3 — Design with /design-database

**Command**: `/design-database [domain or requirements]`
**Source**: `commands/design-database.md`
**Skill**: `database-schema-designer` + `database-designer` agent

**8-step output** (from `commands/design-database.md`):
1. Identify entities and relationships
2. Draw ERD in Mermaid → `docs/database/<feature>-erd.md`
3. Write Flyway migration SQL files → `src/main/resources/db/migration/V<N>__<name>.sql`
4. Include indexes for identified query patterns
5. Add `updated_at` trigger function
6. Firestore structure (if mobile sync needed)
7. Firestore security rules (if applicable)
8. Summarize design decisions

**Produces**: ERD diagram + Flyway migrations + Firestore structure in `docs/database/`

---

### Phase 4 — Review

**Agent**: `postgresql-database-reviewer` (sonnet)
**Vibe**: *"No migration ships without EXPLAIN ANALYZE and a rollback plan"*

**What it checks**:
- Index coverage for every identified query pattern
- Foreign key constraints with correct `ON DELETE` behavior
- CHECK constraints for enum-like columns
- Migration reversibility — every `V<N>__up.sql` needs a corresponding rollback strategy
- NOT NULL correctness — new NOT NULL columns on existing tables need a DEFAULT or migration data
- UNIQUE constraints vs unique indexes (use constraint for business rules)
- Timestamp columns — always `TIMESTAMPTZ`, never `TIMESTAMP`
- UUID vs BIGSERIAL choice justified

**Also run for vector columns**: `pgvector-schema-reviewer` agent

**Produces**: Review findings
**Gate**: Zero CRITICAL findings; migration is reversible; all query patterns have index coverage

---

### Phase 5 — Migration Safety Check

**Trigger**: Review passed
**Checks before running migration**:

| Risk | Check |
|------|-------|
| NOT NULL on existing column | Ensure DEFAULT or backfill in same migration |
| DROP COLUMN | Only after confirming no code references it |
| Adding index on large table | Use `CREATE INDEX CONCURRENTLY` — non-blocking |
| Renaming column | Use expand-contract: add new, copy data, update code, drop old |
| Changing column type | Requires explicit CAST; test with sample data first |

**Database Schema Gate** (from `verification-and-reporting.md`):
- [ ] Migration is reversible (has both up and down path)
- [ ] Indexes defined for expected query patterns
- [ ] `postgresql-database-reviewer` agent run on migration files
- [ ] No direct table drops without explicit human approval

**Gate**: All checklist items pass; human approves any DROP TABLE

---

### Phase 6 — Apply and Verify

**Apply to local/dev**:
```
# Java/Flyway
./mvnw flyway:migrate

# NestJS/Prisma
npx prisma migrate dev --name <migration-name>

# Python/Alembic
alembic upgrade head
```

**Verify**:
```sql
-- Check migration applied
SELECT version, description, installed_on FROM flyway_schema_history ORDER BY installed_rank DESC LIMIT 5;

-- Check indexes created
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = '<table>';

-- Run EXPLAIN ANALYZE on key queries
EXPLAIN ANALYZE SELECT * FROM <table> WHERE <indexed_column> = $1;
```

**Produces**: Migration applied, indexes verified, query plans confirmed
**Gate**: All key queries use index scan (not seq scan) on non-trivial datasets

---

## Quick Reference

| Phase | What to Run | Produces | Gate |
|-------|-------------|----------|------|
| 1 — Load skill | `database-schema-designer` | Reference patterns | Skill loaded |
| 2 — Requirements | Document queries + entities | Access pattern list | Patterns documented |
| 3 — Design | `/design-database` | ERD + Flyway SQL + Firestore | Files in `docs/database/` |
| 4 — Review | `postgresql-database-reviewer` agent | Findings | Zero CRITICAL |
| 5 — Safety check | Manual checklist | Sign-off | All items pass |
| 6 — Apply | Stack migration command | Applied migration | Index scans confirmed |

---

## Common Pitfalls

- **NOT NULL without DEFAULT on existing table** — locks table during migration; supply a DEFAULT or do it in 3 steps (add nullable → backfill → add constraint)
- **Index after the fact** — add indexes in the same migration that creates the table; never assume "we'll add them later"
- **`TIMESTAMP` instead of `TIMESTAMPTZ`** — breaks across timezones; always use `TIMESTAMPTZ`
- **Composite index column order wrong** — leading column must match the WHERE clause; `(a, b)` index doesn't help `WHERE b = ?`
- **DROP INDEX on production** — causes Seq Scans until next vacuum; use `DROP INDEX CONCURRENTLY`
- **Skipping EXPLAIN ANALYZE** — `postgresql-database-reviewer` requires it; query plans lie without real data

## Related Workflows

- [`pgvector-rag-pipeline.md`](pgvector-rag-pipeline.md) — when schema needs vector columns
- [`feature-java-spring.md`](feature-java-spring.md) — Flyway migrations in Spring context
- [`feature-nestjs.md`](feature-nestjs.md) — Prisma migrations in NestJS context
