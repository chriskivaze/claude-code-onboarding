# Blackbox Policy

At the end of any session where decisions were made, code was written,
or constraints were stated: append ONE entry to `blackbox/session-log.md`.

## Entry Format

```
## {ISO-8601-timestamp}
### Decisions
- [Key architectural or technical decision made]
### Constraints Stated by User
- [Any constraint the user explicitly gave — tech choice, scope limit, deadline]
### Files Modified
- [file path] — [one-line reason]
### Deferred
- [Anything explicitly deferred to a future session]
---
```

## Rules
- Append ONLY — never edit or delete existing entries
- Skip if session was purely conversational (no code, no decisions)
- Keep each entry under 20 lines — summarize, don't transcript
- NEVER load `blackbox/session-log.md` into context unless user explicitly asks
- On month boundary: rename `session-log.md` → `archive-YYYY-MM.md`, create fresh `session-log.md`

## Diagram Update Trigger

Naming convention: `src/auth/` maps to `docs/diagrams/auth-flow.md` (folder name = diagram prefix).

**If files modified this session are covered by an existing diagram in `docs/diagrams/`:**
- Update that diagram before ending the session
- Note in Decisions: "Updated docs/diagrams/[name].md — [one-line reason]"

**If a new service, flow, or integration was built and no diagram exists yet:**
- Generate a Mermaid diagram of the new component
- Save to `docs/diagrams/[feature-name]-flow.md`
- Note in Decisions: "Created docs/diagrams/[name].md"

**If unsure whether a diagram needs updating:** check `docs/diagrams/` for files whose prefix matches the folders you modified.
