# Receiving Code Review

> **When to use**: Before implementing any suggestion from a code review — especially when feedback is unclear, conflicts with prior decisions, or seems technically questionable
> **Time estimate**: 15–30 min per review cycle
> **Prerequisites**: PR opened; review comments received on GitHub or in-session

## Overview

Structured approach to receiving and processing code review feedback using the `receiving-code-review` skill. Covers evaluation of reviewer suggestions before implementing them, handling conflicting feedback, maintaining professional disagreement, and the two-stage implementation protocol.

---

## Iron Law (from `skills/receiving-code-review/SKILL.md`)

> **EVALUATE EVERY SUGGESTION BEFORE IMPLEMENTING IT**
> A reviewer's suggestion is a hypothesis about improvement — not a command. You are responsible for the correctness of what you commit.

---

## Phases

### Phase 1 — Read All Feedback Before Responding

**Don't implement the first comment immediately.**

Read ALL review comments first to:
1. Identify contradictions between reviewers
2. Understand the full picture before changing anything
3. Group related comments (all style vs all logic vs all architecture)
4. Identify which comments require discussion vs which are clear fixes

---

### Phase 2 — Categorize Each Comment

For each review comment, classify it:

| Category | Example | Response |
|----------|---------|---------|
| **Bug / Correctness** | "This will NPE when list is empty" | Fix immediately, write a test |
| **Security** | "This endpoint has no auth guard" | Fix immediately |
| **Clear improvement** | "Extract this to a named constant" | Implement |
| **Style / Preference** | "I'd name this `orderId` not `id`" | Implement if consistent with codebase style |
| **Questionable** | "Use pattern X instead of Y" | Evaluate before implementing |
| **Disagreement** | "This architecture is wrong" | Discuss before implementing |
| **Out of scope** | "While you're here, fix Z" | Acknowledge, create separate ticket |

---

### Phase 3 — Load `receiving-code-review` Skill

**Skill**: Load `receiving-code-review` (`.claude/skills/receiving-code-review/SKILL.md`)

**For questionable suggestions, evaluate**:

1. Does the reviewer understand the full context? (or do they only see the changed file?)
2. Is the suggestion backed by a specific concern? (or is it preference?)
3. Does implementing it conflict with prior decisions? (check `docs/adr/` or `docs/plans/`)
4. Is there a performance, correctness, or maintainability reason?

**If the suggestion is correct**: Implement it, credit the reviewer.

**If uncertain**: Ask for clarification before implementing.

**If you disagree**: State your position with evidence, not just "I prefer this way."

---

### Phase 4 — Professional Disagreement

When you believe a suggestion is wrong or harmful:

**Pattern** (from `receiving-code-review` skill):
```
"I see your concern about X. My concern with applying Y instead is:
[specific technical reason with evidence]

The original approach handles [edge case Z] which Y wouldn't because [reason].

Happy to be corrected if I'm missing something — could you elaborate on how Y handles Z?"
```

**Never**:
- Change code you believe is correct just to end the conversation
- Claim "I'll look at it later" and ignore the comment
- Dismiss without engaging: "That's just preference"

**Always**:
- Engage with the substance of the concern
- Provide file:line evidence for your position
- Accept the reviewer's decision if they explain a reason you hadn't considered

---

### Phase 5 — Implementation Protocol

**Two-stage protocol** (from `receiving-code-review` skill):

**Stage 1 — Internal analysis** (before responding to reviewer):
1. Re-read the original code and the suggestion side by side
2. Run the test suite with the suggestion applied locally
3. Check if the suggestion breaks any edge cases
4. Form your position: implement / discuss / decline

**Stage 2 — Respond**:
- For fixes: implement, commit, reply "Fixed in [commit SHA]"
- For discussions: reply with your analysis, ask specific question
- For out-of-scope: reply "Created ticket [N] to address this separately"

---

### Phase 6 — After Implementing

After addressing all comments:

```bash
# Run full test suite
./mvnw test / npx vitest run / uv run pytest / flutter test

# Re-run design system compliance (if UI changed)
/lint-design-system

# Re-run security review (if security-related changes)
/audit-security [changed files]
```

**Update PR description** to summarize what was addressed:
```
## Review Feedback Addressed
- [x] Fixed NPE when order list is empty (commit abc123)
- [x] Added auth guard to /orders/bulk endpoint (commit def456)
- [x] Extracted orderId constant (commit ghi789)
- [ ] Architecture suggestion for OrderService — created separate ticket #45 for discussion
```

---

## Quick Reference

| Phase | Action | Gate |
|-------|--------|------|
| 1 — Read all | Read every comment before responding | All comments read |
| 2 — Categorize | Bug / security / improvement / questionable / disagreement | Category per comment |
| 3 — Evaluate | Load skill; assess questionable suggestions | Decision per comment |
| 4 — Disagree | State position with evidence | Not changing code to end conversation |
| 5 — Implement | Two-stage: analyze then respond | Tests pass after each fix |
| 6 — Summarize | Update PR description | All comments have response |

---

## Common Patterns

**"Could you do X instead of Y?"**
- If X is clearly better: implement it
- If unclear: ask "What specific concern does X address that Y doesn't?"

**"This is too complex"**
- Valid feedback: "Can you point to what you find complex? I want to simplify the right part"
- Not: defensively explain how it's not complex

**"You should use [pattern] here"**
- Check: is it already used in 3+ places? (Rule of Three — if yes, probably right)
- Check: does it apply to this use case? (patterns that are correct in other contexts may not be here)

**"I'd write this differently"**
- Preference without reason: "Happy to change if there's a specific concern — what would be better about the other approach?"
- With reason given: evaluate the reason; if valid, implement

---

## Related Workflows

- [`code-review.md`](code-review.md) — the other side: giving code review
- [`pr-shipping.md`](pr-shipping.md) — shipping the PR after review is complete
- [`iterate-pr.md`](iterate-pr.md) — automated CI fix loop after review
