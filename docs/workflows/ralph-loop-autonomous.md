# Ralph Loop (Autonomous Iteration)

> **When to use**: Long tasks where Claude should work autonomously until completion — without manual re-prompting between steps
> **Time estimate**: Variable — the loop runs until the completion promise is satisfied or explicitly cancelled
> **Prerequisites**: Task is well-defined with a clear completion condition; `/ralph-loop` command available

## Overview

Autonomous iteration using the Ralph loop: Claude works on a task, and when it tries to exit, the stop hook (`stop-ralph-loop.sh`) feeds the same prompt back, forcing continued work until the completion promise is satisfied. Use for long feature implementations, full security audits, test suite runs, or any task that requires many sequential steps without human intervention.

---

## Commands

| Command | Action |
|---------|--------|
| `/ralph-loop [task description]` | Start autonomous iteration on a task |
| `/cancel-ralph` | Immediately stop the running Ralph loop |

---

## How It Works

1. `/ralph-loop` starts with your task description
2. Claude works on the task (writes code, runs tests, fixes failures)
3. When Claude tries to exit (stop event), the `stop-ralph-loop.sh` hook intercepts
4. The hook reads the completion promise from the loop state file
5. If the promise is NOT satisfied (e.g., "all tests pass" = false), the hook feeds the prompt back
6. Claude continues working
7. When the promise IS satisfied, the hook allows the session to end

**State file**: `.claude/ralph-loop-state.md` — tracks current task and completion promise

---

## Phases

### Phase 1 — Define the Completion Promise

A good completion promise is **binary** and **verifiable**:

```
✅ Good completion promises:
- "All tests pass: npx vitest run exits 0"
- "Security audit returns zero CRITICAL findings"
- "PR CI is green: gh pr checks shows all passed"
- "All 43 workflow docs exist in docs/workflows/"

❌ Bad completion promises:
- "Feature is implemented" (not verifiable)
- "Code looks good" (subjective)
- "Done" (circular)
```

---

### Phase 2 — Start the Loop

**Command**:
```
/ralph-loop implement the full Orders NestJS module:
- src/orders/orders.module.ts
- src/orders/orders.controller.ts
- src/orders/orders.service.ts
- src/orders/orders.service.spec.ts
Completion: npx vitest run exits 0 with all orders tests passing
```

**What Ralph does**:
1. Loads relevant skill (nestjs-api)
2. Reads the plan if one exists
3. Implements each file
4. Runs `npx vitest run` after implementation
5. If tests fail → fixes the failure → reruns
6. If tests pass → completion promise satisfied → loop ends

---

### Phase 3 — Monitor (Background)

The Ralph loop runs without human intervention. You can:
- Watch the output scroll
- Step away and come back
- Check `.claude/ralph-loop-state.md` for current status

**Loop state file**:
```markdown
# Ralph Loop State
Task: Implement Orders NestJS module
Completion Promise: npx vitest run exits 0
Current Step: Fixing test failure in orders.service.spec.ts:42
Attempts: 3
Started: 2026-03-13T10:00:00Z
```

---

### Phase 4 — Cancel If Needed

**Immediate stop**:
```
/cancel-ralph
```

This deletes the state file, so the next stop event is not intercepted and the session ends normally.

**When to cancel**:
- Ralph is stuck in a loop (same failure repeating)
- The task definition was wrong and needs to change
- An unexpected blocker requires human judgment
- The task was completed manually

---

### Phase 5 — Review After Loop Completes

After Ralph finishes, review the work:

1. **Read the diff**:
   ```bash
   git diff HEAD
   ```

2. **Verify tests still pass**:
   ```bash
   npx vitest run  # Or stack-equivalent
   ```

3. **Run code review**:
   ```
   /review-code    # Dispatches reviewer agents
   ```

4. **Check for scope creep** (core-behaviors.md §5):
   - Did Ralph touch files outside the task definition?
   - Are all changes traceable to the task?

---

## Best Use Cases

| Use Case | Good For Ralph? | Why |
|----------|----------------|-----|
| Implement multi-file feature with TDD | ✅ | Clear tests = clear completion promise |
| Fix all CI failures on a PR | ✅ | `gh pr checks` is binary |
| Full security audit + fix all findings | ✅ | "zero CRITICAL" is verifiable |
| Generate all 43 workflow docs | ✅ | File count is verifiable |
| Design a new architecture | ❌ | Requires human judgment on trade-offs |
| Ambiguous bug fix | ❌ | Completion not verifiable without understanding |
| Exploratory debugging | ❌ | Needs human interaction during diagnosis |

---

## Quick Reference

| Action | Command | Notes |
|--------|---------|-------|
| Start loop | `/ralph-loop [task + completion promise]` | Completion promise must be binary |
| Cancel loop | `/cancel-ralph` | Deletes state file immediately |
| Check state | `cat .claude/ralph-loop-state.md` | Current step and attempts |
| Review output | `git diff HEAD` + `/review-code` | After loop completes |

---

## Common Pitfalls

- **Vague completion promise** — "done" is not a completion promise; "all tests pass" is
- **Not cancelling when stuck** — if the same error loops 5+ times, Ralph is stuck; cancel and investigate manually
- **No post-loop review** — autonomous code still needs human review before merging; run `/review-code` after
- **Scope drift** — Ralph can expand scope to fix a failing test in a way that touches unintended files; check the diff carefully
- **Running Ralph on architectural decisions** — autonomous iteration works for mechanical tasks, not judgment calls

## Related Workflows

- [`subagent-driven-development.md`](subagent-driven-development.md) — parallel multi-agent alternative to Ralph loop
- [`iterate-pr.md`](iterate-pr.md) — CI fix iteration (can be run inside a Ralph loop)
- [`bug-fix.md`](bug-fix.md) — for complex bugs that need human-in-the-loop debugging
