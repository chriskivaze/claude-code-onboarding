# Database Query Optimization — PostgreSQL

> **When to use**: Diagnosing and fixing slow queries, N+1 problems, pagination bottlenecks, or high database CPU/load in any backend service.
> **Applies to**: Python/FastAPI (SQLAlchemy + asyncpg), NestJS (Prisma), Java/Spring Boot (R2DBC + Flyway)
> **Prerequisites**: Access to the PostgreSQL database; `pg_stat_statements` extension enabled (or slow query log)

## Overview

Structured workflow for finding slow queries, profiling them, and applying the right optimization pattern. Follows: Identify → Profile → Fix → Verify.

## Phases

### Phase 1 — Identify the Slow Query

**Trigger**: Endpoint latency SLA breach, high database CPU, user complaints about slowness

**Option A: pg_stat_statements (preferred)**
```sql
SELECT LEFT(query, 120) AS query, calls, ROUND(mean_exec_time::numeric, 1) AS mean_ms
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 10;
```

**Option B: Application logs**
- Enable SQL logging in your ORM: SQLAlchemy `echo=True`, Prisma `log: ['query']`, R2DBC logging
- Filter for queries > 100ms

**Option C: PostgreSQL slow query log**
```sql
-- In postgresql.conf or via ALTER SYSTEM:
SET log_min_duration_statement = 100;  -- log all queries > 100ms
```

**Skill to load**: `sql-optimization-patterns`

**Gate**: Slow query identified with average execution time > target threshold

---

### Phase 2 — Profile with EXPLAIN

**Trigger**: Slow query identified in Phase 1

**Action**: Run `EXPLAIN (ANALYZE, BUFFERS)` on the exact slow query with representative parameter values

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
<your slow query here>;
```

**What to look for:**
- `Seq Scan` on tables > 10,000 rows → index needed
- `Nested Loop` on large outer result → may need hash join hint or query rewrite
- `Rows=100000` but `Actual Rows=1` → stale statistics, run `ANALYZE <table>`
- High `Buffers: read` (vs `hit`) → cache miss, check `shared_buffers`
- `Hash Batches=4` → memory spill, increase `work_mem` for session

**Reference**: `sql-optimization-patterns/resources/implementation-playbook.md` §1 (EXPLAIN annotation guide)

**Gate**: EXPLAIN output reviewed, bottleneck identified (specific node and table named)

---

### Phase 3 — Apply the Right Pattern

**Trigger**: EXPLAIN bottleneck identified

Match the diagnosis to the pattern:

| EXPLAIN Shows | Root Cause | Pattern |
|---|---|---|
| `Seq Scan` on WHERE column | Missing index | Add B-Tree index |
| `Seq Scan` on multi-column filter | Wrong composite index order | Rebuild index: equality → range |
| N queries for N rows (app logs) | N+1 from ORM | Pattern 1: JOIN or batch load |
| `OFFSET 50000` in query | Keyset pagination missing | Pattern 2: cursor pagination |
| Repeated expensive GROUP BY | No pre-aggregation | Pattern 8: materialized view |
| Loop inserts in application | Missing batch operation | Pattern 5: batch INSERT/UPDATE |
| Correlated subquery per row | Scalar subquery | Pattern 4: rewrite as JOIN or window |
| Large table, date range filter | No partitioning | Pattern 9: range partitioning |

**Skill**: `sql-optimization-patterns` — load `resources/implementation-playbook.md` for the specific pattern SQL

**Stack-specific guidance:**
- **Python/FastAPI**: `selectinload`/`joinedload` for N+1; `execute_many` for batch; cursor pagination in response schema
- **NestJS/Prisma**: `include` for eager load; `prisma.$queryRaw` for complex queries; Prisma cursor API for pagination
- **Java/Spring Boot**: `DatabaseClient` with JOIN for N+1; `executeBatch()` for batch; R2DBC Flux for streaming results

**Gate**: Fix written; not yet merged — pending review

---

### Phase 4 — Review the Fix

**Trigger**: Query rewrite or index change ready

**Action**: Dispatch `postgresql-database-reviewer` agent

**The agent will:**
1. Load `reference/postgresql-review-checklist.md`
2. Verify index type and column order are correct
3. Check for new query anti-patterns introduced
4. Verify the fix doesn't break write performance (over-indexing check)

**Verdict required**: `✅ SAFE TO APPLY` before proceeding

**Gate**: `postgresql-database-reviewer` verdict = SAFE TO APPLY (zero CRITICAL, zero HIGH)

---

### Phase 5 — Verify Improvement

**Trigger**: Fix approved in Phase 4, applied to staging

**Actions:**
1. Re-run `EXPLAIN (ANALYZE, BUFFERS)` on the same query
2. Confirm plan change: `Seq Scan` → `Index Only Scan` (or equivalent improvement)
3. Measure actual execution time before vs. after
4. Run existing test suite — confirm no regressions

**Required evidence format (per verification-and-reporting.md):**
```
Query: [table + operation]
Before: Seq Scan, Execution Time: 4200ms
After:  Index Only Scan, Execution Time: 12ms
Improvement: 350x
Test suite: N passing, 0 failing
```

**Gate**: Measured improvement documented; tests passing

---

### Phase 6 — Schema Change (if partitioning or structural)

**Trigger**: Fix requires table partitioning or schema redesign

**Skill**: `database-schema-designer`
**Agent**: `postgresql-database-reviewer`
**Command**: `/design-database`

**Additional gates (from Database Schema Gate in verification-and-reporting.md):**
- [ ] Migration is reversible (up + down script)
- [ ] Indexes defined on all partitions
- [ ] `postgresql-database-reviewer` verdict = SAFE TO APPLY
- [ ] No direct table drops without explicit human approval

---

## Common Pitfalls

- Running `EXPLAIN` without `ANALYZE` — cost estimates only, not real execution
- Testing on empty or small test database — `EXPLAIN ANALYZE` on staging with production-scale data
- Adding index without checking write frequency — high-write tables need fewer indexes
- Fixing the symptom (slow endpoint) without finding the cause (which query, which table)
- Claiming improvement without before/after `EXPLAIN ANALYZE` comparison

## Related Skills

- `sql-optimization-patterns` — All SQL patterns and EXPLAIN annotation guide
- `sql-pro` — When the fix requires HTAP, analytics tier, or cloud DB migration
- `database-schema-designer` — When the fix requires schema restructuring or partitioning
- `vector-database` — When the slow query is on a pgvector embedding column

## Related Commands

- `/design-database` — Schema redesign if partitioning needed
- `/debug` — If the slow query is inside complex application logic
