# UI/UX Design System Selection

> **When to use**: Starting a new UI feature, redesigning an existing screen, choosing a visual direction, or needing data-backed UX guidelines
> **Time estimate**: 15–30 minutes for design direction; implementation time varies by screen complexity
> **Prerequisites**: `ui-ux-pro-max` skill available; Python 3.x with BM25 dependencies installed

## Overview

Use the design intelligence database to select style, color palette, typography, and UX guidelines before writing Angular or Flutter UI code.

---

## Iron Law (from `skills/ui-ux-pro-max/SKILL.md`)

> **RUN `--design-system` SCRIPT FIRST — never start UI implementation without a validated design direction. Load `frontend-design` skill afterward to apply DFII scoring.**

---

## Skills

- `ui-ux-pro-max` — design knowledge database + BM25 search
- `frontend-design` — DFII scoring, Design Thinking Phase, code implementation

## Agents

- `frontend-design` agent — builds distinctive UI code from the selected design direction
- `ui-standards-expert` agent — verifies token compliance after implementation

---

## Phases

### Phase 1 — Analyze Requirements

Extract:
- Product type (SaaS / health / fintech / etc.)
- Style keywords (e.g., "minimal", "bold", "warm", "corporate")
- Industry vertical
- Workspace stack: Angular → `html-tailwind`, Flutter → `flutter`

### Phase 2 — Run Design System Generation

```bash
python3 .claude/skills/ui-ux-pro-max/scripts/search.py "<keywords>" --design-system -p "Project Name" [-f markdown]
```

Gets style + palette + typography + anti-patterns in one command. This is the primary entry point — do not skip it.

### Phase 3 — Supplement If Needed

Domain searches for additional chart types, UX guidelines, or alternative palettes:

```bash
python3 .claude/skills/ui-ux-pro-max/scripts/search.py "<domain keywords>" --domain ux-guidelines
```

### Phase 4 — Apply DFII Scoring

Load `frontend-design` skill. Score the recommended style with the DFII formula. Minimum score of **8** required to proceed to implementation.

### Phase 5 — Implement

Use `frontend-design` agent with the chosen stack:
- **Angular**: daisyUI semantic tokens (`btn-primary`, `bg-base-100`, etc.)
- **Flutter**: `ThemeData` + `AppSpacing` + `colorScheme` from `Theme.of(context)`

### Phase 6 — Verify

1. Run `/lint-design-system` — zero violations required
2. Run `reality-checker` agent — live screenshot + binary APPROVED/NEEDS WORK verdict

---

## Common Pitfalls

- **Running `--domain` searches before `--design-system`** — domain search gives fragments; `--design-system` gives synthesized recommendations
- **For Angular: mapping hex palette colors directly as Tailwind arbitrary values** instead of mapping to daisyUI semantic tokens
- **Skipping DFII scoring** — a visually appealing palette can fail implementation feasibility
- **Not checking `--stack` flag**: defaulting to `html-tailwind` works for Angular but misses Flutter-specific patterns

---

## Related Workflows

- [feature-angular-spa.md](feature-angular-spa.md)
- [feature-flutter-mobile.md](feature-flutter-mobile.md)
- [tailwind-v4-patterns.md](tailwind-v4-patterns.md)
- [design-system-compliance.md](design-system-compliance.md)
