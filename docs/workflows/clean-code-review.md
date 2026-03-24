# Clean Code Review

> **When to use**: When writing new code, reviewing a PR, or refactoring legacy code across any stack
> **Prerequisites**: Code is written (at least a draft)
> **Skill**: `clean-code`

## Overview

The `clean-code` skill applies Robert C. Martin's *Clean Code* principles as a language-agnostic quality baseline. It complements stack-specific reviewers (`spring-reactive-reviewer`, `nestjs-reviewer`, `riverpod-reviewer`) by enforcing readability and structural correctness that transcends any one framework.

Use this workflow when you want to verify that code is not just *correct* but *clean* — readable by someone else six months from now.

---

## When to Use vs. Related Workflows

| Situation | Workflow |
|-----------|----------|
| Writing new code — enforce quality from the start | This workflow |
| Reviewing a PR for structural quality | This workflow + [`code-review.md`](code-review.md) |
| Refactoring legacy code | This workflow first, then [`tech-debt-cleanup.md`](tech-debt-cleanup.md) |
| Security-focused review | [`security-audit.md`](security-audit.md) |
| Full pre-merge review | [`pr-shipping.md`](pr-shipping.md) |

---

## Workflow Steps

### Step 1 — Load the Skill

```
/clean-code
```

Or invoke via the Skill tool before writing or reviewing code.

### Step 2 — Apply the 9-Principle Checklist

Work through each principle against the code being written or reviewed. Verify at file:line — do not claim violations without reading actual code.

| # | Principle | Key Question |
|---|-----------|-------------|
| 1 | **Meaningful Names** | Are all names intention-revealing and searchable? |
| 2 | **Functions** | Does each function do exactly one thing? ≤2 args? |
| 3 | **Comments** | Are comments explaining *why*, not *what*? Could the code say it instead? |
| 4 | **Formatting** | High-level logic at top, details at bottom (newspaper metaphor)? |
| 5 | **Objects & Data Structures** | Any Law of Demeter violations (`a.getB().getC()`)? |
| 6 | **Error Handling** | No null returns? No null params? Exceptions over return codes? |
| 7 | **Unit Tests** | Tests are F.I.R.S.T.? Failing test written before production code? |
| 8 | **Classes** | Single responsibility? Stepdown rule (callers before callees)? |
| 9 | **Smells** | Rigidity, fragility, viscosity, needless complexity present? |

### Step 3 — Run the Implementation Checklist

```
[ ] Is this function smaller than 20 lines?
[ ] Does this function do exactly one thing?
[ ] Are all names searchable and intention-revealing?
[ ] Have I avoided comments by making the code clearer?
[ ] Am I passing too many arguments (3+ = introduce parameter object)?
[ ] Is there a failing test for this change?
```

### Step 4 — Dispatch Code Reviewer (Optional, Recommended)

For PRs, dispatch the stack-appropriate reviewer after the clean-code check:

```
/review-code
```

This adds stack-specific checks (reactive correctness, DTO patterns, Riverpod lifecycle) on top of the language-agnostic clean-code baseline.

---

## Stack-Specific Examples

### Java / Spring Boot

```java
// VIOLATION: Law of Demeter (clean-code §5)
String city = order.getCustomer().getAddress().getCity();

// FIX: delegate to the owning object
String city = order.getCustomerCity(); // method added to Order
```

```java
// VIOLATION: function does more than one thing (clean-code §2)
public ResponseEntity<?> createOrder(OrderDto dto) {
    // validates, persists, sends email, publishes event — 4 things
}

// FIX: extract single-responsibility methods
public ResponseEntity<?> createOrder(OrderDto dto) {
    validate(dto);
    Order order = persist(dto);
    notifyCustomer(order);
    publishEvent(order);
    return ResponseEntity.ok(order);
}
```

### NestJS / TypeScript

```typescript
// VIOLATION: 3+ args without parameter object (clean-code §2)
async createUser(name: string, email: string, role: string, tenantId: string) {}

// FIX: parameter object (already enforced by NestJS DTO pattern)
async createUser(dto: CreateUserDto) {}
```

```typescript
// VIOLATION: comment explaining bad code (clean-code §3)
// Check if not null because the DB sometimes returns undefined
if (user !== null && user !== undefined) { ... }

// FIX: make code express intent — validate at boundary
const user = await this.userService.findOrThrow(id); // throws NotFoundException
```

### Python / FastAPI

```python
# VIOLATION: returns null on error (clean-code §6)
def get_user(id: int):
    try:
        return db.find(id)
    except Exception:
        return None  # forces every caller to null-check

# FIX: raise, let caller handle or let FastAPI return 404
def get_user(id: int) -> User:
    user = db.find(id)
    if user is None:
        raise HTTPException(status_code=404, detail=f"User {id} not found")
    return user
```

### Flutter / Dart

```dart
// VIOLATION: misleading name + null return (clean-code §1, §6)
Future<List<Item>?> getData() async { ... }

// FIX: intention-revealing name + never return null
Future<List<CartItem>> fetchUserCart() async {
  // throws on error, never returns null
}
```

### Angular / TypeScript

```typescript
// VIOLATION: function name doesn't reveal intent (clean-code §2)
process(data: any) { ... }

// FIX: intention-revealing verb + no `any`
transformApiResponseToViewModel(response: ApiUserResponse): UserViewModel { ... }
```

---

## Common Violations by Stack

| Stack | Most Common Clean Code Violation |
|-------|----------------------------------|
| Java / Spring | Law of Demeter chains; classes doing too much |
| NestJS | 3+ arg functions instead of DTOs; misleading service names |
| Python | Null returns on error; comment-heavy code vs self-documenting |
| Flutter | Ambiguous `data` / `result` variable names; mixed abstraction levels |
| Angular | Logic in templates instead of components; god-component syndrome |

---

## Related Workflows

- [`code-review.md`](code-review.md) — stack-specific review after clean-code check
- [`tech-debt-cleanup.md`](tech-debt-cleanup.md) — systematic removal of accumulated smells
- [`test-driven-development.md`](test-driven-development.md) — enforce F.I.R.S.T. principles from the start
- [`pr-shipping.md`](pr-shipping.md) — full pre-merge gate (includes clean-code as quality signal)
