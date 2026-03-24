# ADR Creation (Architecture Decision Records)

> **When to use**: Making an architectural decision that is hard to reverse, has multiple viable options, or will be questioned in future code reviews
> **Time estimate**: 30–60 min per ADR
> **Prerequisites**: The decision context and options are understood; feature spec or architecture plan exists

## Overview

Architecture Decision Records (ADRs) using the `architecture-decision-records` skill. Documents key decisions with context, alternatives, and consequences in a lightweight, searchable format. Stored in `docs/adr/` as numbered markdown files.

---

## When to Write an ADR

Write an ADR for decisions that are:
- **Hard to reverse** — database choice, event model, API versioning strategy
- **Contested** — where team members had different opinions
- **Non-obvious** — future reader might question why this was chosen
- **High-impact** — affects multiple services or teams

**Don't write ADRs for**:
- Implementation details that are easily changed
- Decisions mandated externally (no real choice)
- Obvious choices with no real alternatives

---

## Skill

**Skill**: Load `architecture-decision-records` (`.claude/skills/architecture-decision-records/SKILL.md`)

---

## ADR Lifecycle

```
Proposed → Accepted → Superseded / Deprecated
```

- **Proposed**: Under discussion
- **Accepted**: Decision made, implementation follows this
- **Superseded by ADR-[N]**: A later decision changed this; both documents kept
- **Deprecated**: Context changed; no longer applies

---

## File Naming and Location

```
docs/adr/
├── 0001-use-postgresql-for-primary-storage.md
├── 0002-sync-vs-async-order-processing.md
├── 0003-nestjs-over-express-for-api-layer.md
└── 0004-weaviate-over-pgvector-for-rag.md
```

**Naming**: `<sequential-number>-<kebab-case-decision-title>.md`

**Sequential numbering**: Never reuse numbers. Never skip numbers. If an ADR is superseded, create a new higher-numbered ADR — don't edit the old one.

---

## ADR Template

```markdown
# ADR-[N]: [Decision Title]
Date: [YYYY-MM-DD]
Status: Proposed | Accepted | Superseded by ADR-[M] | Deprecated

## Context

[Why is this decision needed? What forces, constraints, or requirements led here?
What is the current situation that necessitates a choice?
2-4 sentences.]

## Decision

[What was decided? State it clearly in one or two sentences.
The decision should be unambiguous — anyone reading this should know exactly what was chosen.]

## Alternatives Considered

| Option | Pros | Cons | Why Not Chosen |
|--------|------|------|---------------|
| [A — rejected] | ... | ... | [specific reason] |
| [B — rejected] | ... | ... | [specific reason] |
| **[C — chosen]** | **...** | **...** | **Chosen** |

## Consequences

### Positive
- [What this decision enables]
- [What complexity it eliminates]

### Negative
- [What this decision makes harder]
- [What technical debt it incurs]
- [What future options it forecloses]

### Neutral
- [What changes but isn't clearly better or worse]

## Implementation Notes

[Optional: specific constraints or patterns that must be followed because of this decision.
E.g., "All queries must use the cosine operator <=> to match the HNSW index ops class."]
```

---

## Phases

### Phase 1 — Gather the Decision Context

Before writing:
1. What specific problem led to this decision?
2. What are the real constraints? (budget, expertise, existing infrastructure)
3. What were the options actually evaluated? (not just mentioned, but considered)
4. What made the chosen option win? (be specific — not "it's better")

---

### Phase 2 — Fill the Template

**Context** (why this decision exists):
- Not background knowledge — why THIS choice needed to be made NOW
- What would break or be blocked without making this decision?

**Decision** (what was chosen):
- One clear sentence
- Start with "We will use..." or "We decided to..."

**Alternatives** (show the work):
- Include at least 2 alternatives, even if they were quickly dismissed
- Be honest about cons of the chosen option

**Consequences** (the real cost of the decision):
- Positive: what this unlocks
- Negative: what this costs or forecloses
- Neutral: what changes without value judgment

---

### Phase 3 — Review with `the-fool`

**Skill**: Load `the-fool`

Ask: "Challenge this ADR — what's wrong with this decision? What did we miss?"

Pre-mortem for the decision:
- "What would make us regret this choice in 2 years?"
- "What assumption are we making that might be wrong?"
- "What would change our answer?"

Incorporate any valid challenges into the ADR's Consequences / Negative section.

---

### Phase 4 — Commit and Link

**Commit the ADR**:
```bash
git add docs/adr/0004-weaviate-over-pgvector-for-rag.md
git commit -m "docs: ADR-0004 choose Weaviate over pgvector for RAG pipeline"
```

**Link from related files**:
```markdown
# In the feature plan or implementation doc
See: [ADR-0004: Weaviate over pgvector for RAG](../adr/0004-weaviate-over-pgvector-for-rag.md)
```

**Reference in code** (when an unintuitive implementation decision traces to an ADR):
```typescript
// Uses cosine distance (<=>), not L2 (<->), per ADR-0004 (Weaviate HNSW ops class)
const results = await db.query('SELECT * FROM docs ORDER BY embedding <=> $1', [queryEmbedding]);
```

---

## Example ADR

```markdown
# ADR-0004: Use Weaviate Over pgvector for RAG Pipeline
Date: 2026-03-13
Status: Accepted

## Context
We need to add semantic search and RAG to the product. Our data is primarily unstructured text
(support articles, product descriptions) with no existing PostgreSQL tables. We need a managed
vectorizer to avoid running a separate embedding service.

## Decision
We will use Weaviate Cloud as the vector database for the RAG pipeline, with `text2vec-openai`
as the vectorizer.

## Alternatives Considered

| Option | Pros | Cons | Why Not Chosen |
|--------|------|------|---------------|
| pgvector | In our existing PostgreSQL | Requires separate embedding service; no native generative module | Would add 2 additional managed services |
| Pinecone | Fully managed, fast | No built-in vectorizer; higher cost; no generative RAG | Cost and no vectorizer |
| **Weaviate Cloud** | Built-in vectorizer; native generative RAG; managed | External dependency | **Chosen** |

## Consequences

### Positive
- No embedding service to manage — text2vec-openai handles it
- Native generative RAG API (Generate module) without building a pipeline
- Auto-scaling with Weaviate Cloud

### Negative
- External dependency on Weaviate Cloud availability
- Migration cost if we need to change vector DB later (full re-embedding required)
- All search goes through Weaviate, not standard SQL

### Neutral
- Team needs to learn Weaviate Python client v4 API
```

---

## Quick Reference

| Phase | Action | Gate |
|-------|--------|------|
| 1 — Context | Gather why/what/options | Decision context written |
| 2 — Template | Fill all sections | No "TBD" or empty sections |
| 3 — Challenge | `the-fool` pre-mortem | Valid challenges incorporated |
| 4 — Commit | `docs/adr/<N>-<title>.md` | File committed, linked from plan |

---

## Related Workflows

- [`architecture-design.md`](architecture-design.md) — architecture design produces ADR-worthy decisions
- [`plan-review.md`](plan-review.md) — plans reference ADRs for key decisions
- [`documentation-generation.md`](documentation-generation.md) — ADRs are a documentation output type
