# Hookify Rule Management

> **When to use**: Creating enforcement rules to prevent repeated mistakes, managing existing hooks, or auditing which hooks are active
> **Time estimate**: 15–30 min per rule
> **Prerequisites**: A repeated behavior you want to block or enforce; access to `.claude/hookify.*.local.md` files

## Overview

Hookify is a YAML-based rule system that creates behavioral guardrails enforced on every Claude Code session. Rules are stored in `.claude/hookify.*.local.md` files and checked automatically via hooks. Use `/hookify` to create rules, `/hookify-list` to view active rules, and `/hookify-configure` to enable/disable.

---

## Commands

| Command | What it does |
|---------|-------------|
| `/hookify [description]` | Create a new Hookify rule from a description or from conversation context |
| `/hookify-list` | Show all rules with name, event, pattern, enabled/disabled status |
| `/hookify-configure` | Interactively enable or disable rules |

---

## Rule File Structure

Hookify rules live in `.claude/hookify.*.local.md` files:

```yaml
---
name: no-console-log
event: PreToolUse
tool: Write
pattern: "console\\.log|console\\.warn|console\\.error"
message: "Use centralized logger — console.* statements leak internal info to devtools"
enabled: true
---
```

**Fields**:
| Field | Values | Description |
|-------|--------|-------------|
| `name` | kebab-case | Unique identifier |
| `event` | `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `SessionStart` | When the rule fires |
| `tool` | `Write`, `Edit`, `Bash`, `*` | Which tool triggers it |
| `pattern` | regex | What to match in the tool input |
| `message` | string | Warning shown when rule fires |
| `enabled` | `true` / `false` | Active or inactive |

---

## Phases

### Phase 1 — Identify the Behavior to Block

Hookify rules should encode **proven recurring mistakes**, not speculative concerns.

**Good candidates for Hookify rules**:
- `console.log` in TypeScript files (should use centralized logger)
- Hardcoded API keys or secrets in code
- `TODO` comments in committed code
- Design system violations (hardcoded colors, raw spacing values)
- `debugPrint` / `kDebugMode` checks left in Flutter release code

**Bad candidates** (too broad, would block legitimate work):
- "Don't write bad code" — not specific enough
- "Always use design system" — needs to be a specific pattern match

---

### Phase 2 — Create a Rule with `/hookify`

**Invoke**:
```
/hookify console.log in TypeScript files should use the centralized logger
```

**What happens**:
1. Skill `hookify` loads (`skills/hookify/SKILL.md`)
2. Analyzes the description or recent conversation for the behavior to block
3. Generates the YAML rule
4. Writes to `.claude/hookify.<rule-name>.local.md`

**Generated file example**:
```yaml
---
name: no-console-log-typescript
event: PreToolUse
tool: Write
pattern: "console\\.(?:log|warn|error|debug)"
message: |
  HOOKIFY BLOCKED: Direct console.* usage detected.
  Use the centralized logger instead:
  - Angular: inject Logger service or use ErrorHandler
  - NestJS: inject Logger from @nestjs/common
  Raw console statements expose internals to devtools.
enabled: true
---
```

---

### Phase 3 — List Active Rules

```
/hookify-list
```

**Output**:
```
Active Hookify Rules:
1. no-console-log-typescript — PreToolUse/Write — ENABLED
   Pattern: console\.(?:log|warn|error|debug)
2. no-hardcoded-colors — PreToolUse/Write — ENABLED
   Pattern: color:\s*(#[0-9a-fA-F]{3,6}|rgb\()
3. no-todo-comments — PreToolUse/Write — DISABLED
   Pattern: //\s*TODO
4. no-debuggable-true — PreToolUse/Write — ENABLED
   Pattern: android:debuggable="true"
```

---

### Phase 4 — Configure Rules

**Enable or disable** (without deleting):
```
/hookify-configure
```

**Interactive selection**:
- Choose rule by name
- Toggle enabled/disabled
- Saves to the rule file

**Manual edit** (for power users):
```yaml
# In .claude/hookify.<name>.local.md
enabled: false  # Temporarily disabled
```

---

### Phase 5 — Active Hooks Inventory

Current active Hookify rules in this workspace (from `.claude/hookify.*.local.md`):

**Design System Rules**:
- Hardcoded color patterns in Flutter/Angular code
- Raw spacing values outside of design token usage
- Inline TextStyle instead of theme references

**Security Rules**:
- Hardcoded API keys, tokens, credentials
- `android:debuggable="true"` in production code

**Code Quality Rules**:
- `console.log/warn/error` in TypeScript
- `print()` in Python (should use logging)

View all with `/hookify-list`.

---

### Phase 6 — Promote Patterns from lessons.md

When a mistake has been corrected 3+ times (lessons.md rule hits `[x3]`):

**From `CLAUDE.md` Self-Improvement Loop**:
> If the entry is now `[x3]` — promote Rule to the matching rules file, then delete entry from lessons.md

Hookify is an alternative destination: if the mistake is **detectable via pattern match**, create a Hookify rule instead of adding to a rules file.

```
Lesson: "Don't use console.log in TypeScript [x3]"
→ Create Hookify rule: no-console-log-typescript
→ Delete lessons.md entry (now enforced automatically)
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Create a new rule | `/hookify [description]` |
| View all rules | `/hookify-list` |
| Enable/disable rules | `/hookify-configure` |
| Cancel an active loop | `/cancel-ralph` |

---

## Common Pitfalls

- **Rules too broad** — a rule that fires on legitimate code will interrupt constantly; narrow the pattern
- **Blocking without helpful message** — the `message` field must explain what to do instead, not just what's wrong
- **Never disabling rules** — some rules are context-specific (e.g., debug prints OK in tests); `enabled: false` when not needed
- **Duplicating lessons.md content** — if a lesson is pattern-matchable, make it a Hookify rule and delete the lesson entry

## Related Workflows

- [`new-skill-creation.md`](new-skill-creation.md) — skills and hooks together form the behavioral guardrail system
- [`developer-onboarding.md`](developer-onboarding.md) — new developers should review active hooks during onboarding
- [`security-hardening.md`](security-hardening.md) — security-related patterns (hardcoded secrets) make ideal Hookify rules
