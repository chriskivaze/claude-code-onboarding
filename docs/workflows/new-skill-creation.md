# New Skill Creation

> **When to use**: Adding a new technology, pattern, or domain to the Claude Code workspace — so future sessions have the right patterns, MCP servers, and agent mappings loaded automatically
> **Time estimate**: 1–2 hours for a focused skill; 4–8 hours for a comprehensive skill with full reference library
> **Prerequisites**: The technology or pattern you want to encode has been used at least once; patterns are proven, not speculative

## Overview

Skill creation using the `writing-skills` skill and `skill-reviewer` agent. A skill is a lazy-loaded instruction set that gives Claude the right patterns, templates, and MCP server references for a specific domain. Follows the Iron Law, uses a structured `SKILL.md` format with `reference/` library, and is reviewed before activation.

---

## Iron Law (from `skills/writing-skills/SKILL.md`)

> **A SKILL IS NOT A CHEAT SHEET — IT IS A BEHAVIORAL CONTRACT**
> Every rule in a skill must be actionable, verifiable, and derived from actual project experience.

---

## Skill Anatomy

```
.claude/skills/<skill-name>/
├── SKILL.md                    # Main skill file — ALWAYS loaded when skill is active
└── reference/
    ├── patterns.md             # Code patterns and templates
    ├── anti-patterns.md        # What to avoid and why
    └── [topic-specific].md     # Additional reference files
```

**`SKILL.md` structure** (from `writing-skills` skill):
```markdown
---
name: skill-name
description: [CSO trigger words — what the user says to activate this skill]
allowed-tools: [Read, Write, Edit, Bash, Grep, Glob]
---

# [Skill Name]

## Iron Law
[One non-negotiable rule that governs ALL behavior in this domain]
[In bold, 1–2 sentences]

## When to Load This Skill
[Specific trigger phrases and contexts]

## Key Decisions
[The 3–5 most important decisions in this domain with rationale]

## Pattern Quick Reference
[Table of common patterns and their templates or file references]

## Reference Files
[Pointers to reference/ library with one-line description each]

## MCP Servers
[Which MCP servers to query before writing code in this domain]

## Agent Mapping
[Which agents to dispatch for review]
```

---

## Phases

### Phase 1 — Load Writing-Skills and Define Scope

**Skill**: Load `writing-skills` (`.claude/skills/writing-skills/SKILL.md`)

**Define before writing**:
1. What domain does this skill cover?
2. What are the trigger phrases? (What would a developer say to need this?)
3. What is the single most important rule (Iron Law)?
4. What patterns are proven (used 3+ times)?
5. What MCP servers apply?
6. What reviewer agent applies?

**Check for existing skills**: Does a skill already cover this partially?
```bash
ls .claude/skills/
grep -r "your-domain" .claude/skills/*/SKILL.md
```

If existing skill covers it partially → update that skill rather than creating a new one.

---

### Phase 2 — Write SKILL.md

**File**: `.claude/skills/<skill-name>/SKILL.md`

**Size constraint** (from `writing-skills` skill): Body ≤ 500 lines. If exceeding, extract to `reference/` files.

**Required fields**:
- `name`: kebab-case, matches directory name
- `description`: CSO trigger words that describe WHEN to activate (not what it does)
- `allowed-tools`: list only tools the skill actually needs

**Iron Law format**:
```markdown
## Iron Law
**[ACTION IN CAPS] [specific rule] — [consequence of violating it]**
```

Example:
```markdown
## Iron Law
**ALWAYS QUERY CONTEXT7 FOR API SIGNATURES BEFORE WRITING CODE** — LangChain's API changes between minor versions; code from memory will be deprecated
```

**Progressive disclosure structure**:
1. Iron Law (most important, first)
2. When to load (trigger context)
3. Key decisions (3–5 most important choices)
4. Pattern quick reference (table form, links to reference/)
5. MCP servers (lookup order)
6. Agent mapping

---

### Phase 3 — Write Reference Library

**Directory**: `.claude/skills/<skill-name>/reference/`

**Each reference file**:
- One focused topic per file (patterns, anti-patterns, migration-guide, etc.)
- Concrete code examples over prose
- Cross-references to official docs

**Standard reference files** (not all required — include what's relevant):

| File | Contents |
|------|---------|
| `patterns.md` | Proven code templates with comments |
| `anti-patterns.md` | What to avoid, with "why" |
| `naming-conventions.md` | File, class, method, variable naming rules |
| `error-handling.md` | Stack-specific error handling patterns |
| `testing-patterns.md` | How to write tests for this domain |
| `migration-guide.md` | How to upgrade or migrate |

---

### Phase 4 — Update CLAUDE.md Mapping Tables

After writing the skill, update `CLAUDE.md` to reference it:

**Code Conventions table**:
```markdown
| [Technology] | `.claude/skills/<skill-name>/` | `<agent-name>` | `/<command>` |
```

**Code Review Agents table** (if a reviewer agent applies):
```markdown
| [Domain] | `<reviewer-agent>` |
```

---

### Phase 5 — Run `skill-reviewer` Agent

**Agent**: `skill-reviewer`

**Dispatch with**:
```
Review the new skill at .claude/skills/<skill-name>/
Check against the writing-skills spec:
- Iron Law presence and format
- Description CSO trigger words
- allowed-tools declaration
- Body ≤ 500 lines
- Progressive disclosure structure
- No forbidden files (no CLAUDE.md content duplicated)
```

**What it checks** (from `skill-reviewer` agent description):
- Iron Law presence in correct format
- Description trigger words (CSO = Concrete, Specific, Observable)
- `allowed-tools` declaration present
- Body line count ≤ 500
- Progressive disclosure structure
- Forbidden files not created (no README, no second CLAUDE.md)
- Cross-reference patterns (does reference/ match what SKILL.md points to?)

**Gate**: `skill-reviewer` returns PASS with no CRITICAL findings.

---

### Phase 6 — Register in Skill Tool (if user-invocable)

If the skill should be user-invocable (slash command style):

1. Check if a command file should be created: `.claude/commands/<skill-name>.md`
2. Add to the skill listing in CLAUDE.md
3. Test by loading the skill explicitly and verifying behavior

---

## Quick Reference

| Phase | Action | Gate |
|-------|--------|------|
| 1 — Scope | Define Iron Law, triggers, MCP servers | Written answers to 6 questions |
| 2 — SKILL.md | Write main skill file | ≤ 500 lines, Iron Law present |
| 3 — Reference | Write reference/ library | Code examples, not just prose |
| 4 — Update CLAUDE.md | Add to mapping tables | Both tables updated |
| 5 — Review | `skill-reviewer` agent | PASS, no CRITICAL findings |
| 6 — Register | Command file (if applicable) | Skill appears in tool listing |

---

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Skill directory | kebab-case | `agentic-ai-dev` |
| SKILL.md name field | kebab-case | `agentic-ai-dev` |
| Reference files | kebab-case | `error-handling.md` |
| Agent name (if new) | kebab-case | `agentic-ai-reviewer` |
| Command (if new) | kebab-case | `/scaffold-agentic-ai` |

---

## Common Pitfalls

- **Duplicating CLAUDE.md content in skills** — skills are lazy-loaded supplements, not copies of global rules
- **Iron Law that's vague** — "always use good practices" is not an Iron Law; "ALWAYS QUERY CONTEXT7 FOR API VERSION BEFORE WRITING CODE" is
- **No proven patterns** — skills should encode what has worked, not what might work; require 3+ real uses
- **Skills over 500 lines** — extract prose to reference/ files; SKILL.md should be scannable in 2 minutes
- **No agent mapping** — every skill needs a reviewer agent; if none exists, use `code-reviewer`

## Related Workflows

- [`hookify-management.md`](hookify-management.md) — creating hook-based enforcement rules
- [`developer-onboarding.md`](developer-onboarding.md) — skills are part of onboarding setup
- [`documentation-generation.md`](documentation-generation.md) — skills generate documentation artifacts
