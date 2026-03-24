---
name: design-system
description: "Unified design system enforcement for Flutter and Angular. Routes all UI tasks through the correct stack-specific tokens, rules, and interaction contracts. Use when auditing UI compliance, reviewing design consistency, or building any user-facing surface."
allowed-tools: Read, Glob, Grep, Write, Edit
metadata:
  triggers: design system, UI audit, design tokens, design review, design lint, visual consistency, theme compliance, design drift
  related-skills: ui-standards-tokens, angular-spa, flutter-mobile, frontend-design, accessibility-auditor
  domain: frontend
  role: specialist
  scope: design
  output-format: document
last-reviewed: "2026-03-15"
---

**Iron Law:** Never hardcode colors, spacing, or typography — always use design tokens; run /lint-design-system before declaring any UI change complete.

# Design System — Unified Routing Hub

Single entry point for all design system enforcement across Flutter and Angular.

**When to use:** Any UI task — building screens, reviewing components, auditing design drift, or running design lint.

## Routing Table

Determine the stack from file context, then load the correct references:

```
What are you working on?
    |
    +-- Flutter (lib/**/*.dart, pubspec.yaml)
    |   |
    |   +-- Tokens (colors, spacing, typography, radius)
    |   |   → Read: .claude/skills/ui-standards-tokens/reference/ui-design-tokens.md
    |   |
    |   +-- Accessibility (semantics, focus, contrast)
    |   |   → Read: .claude/skills/ui-standards-tokens/reference/ui-accessibility-patterns.md
    |   |
    |   +-- Polish (animations, glassmorphism, theme extensions)
    |       → Read: .claude/skills/flutter-mobile/reference/flutter-design-polish.md
    |
    +-- Angular (src/app/**/*.ts|html|scss)
    |   |
    |   +-- Component styling (daisyUI classes, semantic tokens)
    |   |   → Read: .claude/skills/angular-spa/reference/daisyui-v5-components.md
    |   |
    |   +-- Tailwind config (spacing, breakpoints, CSS vars)
    |   |   → Read: .claude/skills/angular-spa/reference/tailwind-v4-config.md
    |   |
    |   +-- Conventions (design principles, form patterns)
    |   |   → Read: .claude/skills/angular-spa/reference/angular-conventions.md
    |   |
    |   +-- Animations (timing, keyframes, reduced motion)
    |       → Read: .claude/skills/angular-spa/reference/animations.md
    |
    +-- Cross-stack (interaction contracts, visual direction)
        |
        +-- Surface/interaction contracts (modals, forms, lists, errors)
        |   → Read: .claude/skills/design-system/reference/interaction-contracts.md
        |
        +-- Visual design direction (anti-patterns, typography philosophy)
            → Read: .claude/skills/frontend-design/reference/frontend-design-principles.md
```

## Design Rules — Hard Policy (Both Stacks)

These are non-negotiable. Violations are caught by hookify rules at write-time.

### Colors
- **NEVER** hardcode hex values (`#3b82f6`, `Color(0xFF...)`)
- **NEVER** use `rgb()`, `rgba()`, `hsl()`, `hsla()` literals
- **Flutter:** Use `Theme.of(context).colorScheme.*`
- **Angular:** Use daisyUI semantic tokens (`bg-primary`, `text-base-content`)

### Spacing
- **NEVER** use raw numeric spacing (`EdgeInsets.all(16)`, `mt-3`)
- **Flutter:** Use `AppSpacing.xs/sm/md/lg/xl/xxl`
- **Angular:** Use Tailwind semantic scale or daisyUI component spacing

### Typography
- **NEVER** use raw font sizes (`TextStyle(fontSize: 14)`, `text-[14px]`)
- **Flutter:** Use `Theme.of(context).textTheme.*`
- **Angular:** Use Tailwind typography scale (`text-sm`, `text-lg`, `text-xl`)

### Forms
- **NEVER** use bare `<input>`, `<select>`, `<textarea>` without framework bindings
- **Angular:** Use daisyUI form classes + reactive form `formControlName`
- **Flutter:** Use shared form field wrapper widgets

### Touch Targets
- **Minimum 48dp** (Flutter) / **44px** (Angular) for all interactive elements

### Inline Styles
- **NEVER** use `style="..."` in Angular templates — use Tailwind utilities or SCSS
- **NEVER** use inline `Style` widgets in Flutter — use theme extensions

## Machine Enforcement

### Hookify Rules (fire on every Write/Edit by Claude)

| Rule | Stack | What It Catches |
|------|-------|-----------------|
| `hookify.design-no-hardcoded-colors-dart` | Flutter | `Color(0xFF...)`, `Colors.blue` |
| `hookify.design-no-raw-spacing-dart` | Flutter | `EdgeInsets.all(16)`, `SizedBox(height: 8)` |
| `hookify.design-no-raw-textstyle-dart` | Flutter | `TextStyle(fontSize: N)` |
| `hookify.design-no-hex-angular` | Angular | `bg-[#...]`, `color: #...`, `rgb()`, `hsl()` |
| `hookify.design-no-raw-spacing-angular` | Angular | `mt-3`, `px-4`, `gap-2` |
| `hookify.design-no-raw-typography-angular` | Angular | `text-[14px]`, `font-[...]`, `font-size: N` |
| `hookify.design-no-raw-form-inputs` | Angular | bare `<input>`, `<select>` without `formControl` |

All rules respect `// ignore-design: [reason]` exception markers.

### Lint Commands

| Command | Scope |
|---------|-------|
| `/lint-design-system` | Orchestrator — runs all checks for detected stack(s) |
| `dart analyze` | Flutter static analysis |
| `ng lint` | Angular static analysis |

### Quality Gate

Before declaring any UI work done, these must pass (from `verification-and-reporting.md`):

- [ ] No hardcoded colors — all colors use theme tokens
- [ ] No raw spacing values — all spacing uses semantic tokens
- [ ] No inline TextStyles — all typography uses theme text styles
- [ ] Touch targets >= 48dp (Flutter) / 44px (Angular)
- [ ] `/lint-design-system` run with zero violations
- [ ] Exception markers (`// ignore-design: [reason]`) reviewed and justified

## Exception Policy

When a design rule must be violated intentionally:

1. Add inline marker: `// ignore-design: [short reason]` or `<!-- ignore-design: [reason] -->`
2. Reason must explain WHY (e.g., "platform-specific iOS styling", "third-party widget constraint")
3. Exceptions are reviewed like code debt
4. Periodically audit: search for `ignore-design` and remove stale exceptions
5. If 5+ exceptions accumulate in one file → the design system may need extending, flag it

## Scope & Rationale

### What We Enforce

| Surface | Status | Rationale |
|---------|--------|-----------|
| **Logged-in app screens** (Flutter + Angular) | ENFORCED | Primary user experience — consistency here drives retention and trust |
| **Shared/reusable components** | ENFORCED | Foundation — drift here cascades everywhere |
| **Theme definitions** | ENFORCED | Single source of truth for tokens |

### What We Defer

| Surface | Status | Rationale |
|---------|--------|-----------|
| **Admin panels / internal tools** | DEFERRED | Lower user impact — enforce when dedicated redesign pass happens |
| **Legacy/migration pages** | DEFERRED | Will be replaced — enforce on new version only |
| **Marketing / landing pages** | DEFERRED | Often need custom creative direction that conflicts with app tokens |
| **Test files** | EXCLUDED | Test code can use raw values for assertion clarity |
| **Generated code** (`*.g.dart`, `*.freezed.dart`) | EXCLUDED | Machine-generated — not human-authored |

### File Path Scope

| Stack | Included | Excluded |
|-------|----------|----------|
| Flutter | `lib/` | `test/`, `*.g.dart`, `*.freezed.dart`, build output |
| Angular | `src/app/` | `node_modules/`, `*.spec.ts`, config files |
