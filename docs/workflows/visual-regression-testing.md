# Visual Regression Testing

> **When to use**: Adding visual regression tests to a new Angular or Flutter project, investigating "looks different on CI than locally", before merging any PR that adds or modifies UI components
> **Time estimate**: 1–2 hours for initial setup; ~15 minutes per PR cycle once established
> **Prerequisites**: Angular or Flutter project with baseline UI; GitHub Actions CI configured

## Overview

Set up automated visual regression CI/CD pipelines and run systematic pre-PR visual verification for Angular and Flutter UIs.

---

## Iron Law (from `skills/ui-visual-validator/SKILL.md`)

> **USE `reality-checker` AGENT FIRST for live browser verdict. This workflow adds CI/CD automation and the 13-item pre-commit checklist — not a replacement for reality-checker.**

---

## Skills

- `ui-visual-validator` — visual regression tooling setup + 13-item checklist + CI/CD integration
- `browser-testing` — Chrome DevTools MCP for live screenshots

## Agents

- `reality-checker` — final APPROVED/NEEDS WORK verdict (always run this)
- `browser-testing` — live browser interaction + screenshot capture

---

## Phases

### Phase 1 — Run reality-checker First

Get baseline live screenshot and binary verdict before setting up regression tooling. Do not proceed to tooling setup until you have a passing baseline.

### Phase 2 — Choose Tool for Project Type

| Project Type | Tool |
|---|---|
| Angular with Storybook | Chromatic |
| Angular E2E | Playwright Visual Comparisons |
| Config-based (no Storybook) | BackstopJS |
| Cross-browser cloud | Percy |
| Flutter | `golden_toolkit` (`flutter test --update-goldens`) |

### Phase 3 — Run 13-Item Mandatory Checklist

Load `ui-visual-validator` skill and run through all 13 items before declaring any UI change complete.

### Phase 4 — Set Up CI/CD Pipeline

Add GitHub Actions workflow for automated screenshot diffs on every PR. See tool-specific configs in `ui-visual-validator` skill reference files.

### Phase 5 — Establish Baseline

Generate reference screenshots on the main branch:

```bash
# BackstopJS
backstop reference

# Flutter
flutter test --update-goldens

# Playwright
npx playwright test --update-snapshots
```

### Phase 6 — Verify

Run comparison — zero regressions required:

```bash
# BackstopJS
backstop test

# Flutter
flutter test

# Playwright
npx playwright test
```

---

## Common Pitfalls

- **Skipping reality-checker and relying only on automated tools** — automated tools miss layout shifts and interaction states
- **Setting mismatch threshold too low (< 0.05%)** — font rendering differences between CI and local cause false failures
- **Not testing both light AND dark mode** (daisyUI `data-theme` switch, Flutter `ThemeMode.dark`)
- **Not establishing baseline on stable main branch** — first run will always fail without reference screenshots
- **Flutter golden tests fail on CI due to font rendering differences** — use `golden_toolkit` `loadAppFonts()` to fix

---

## Related Workflows

- [feature-angular-spa.md](feature-angular-spa.md)
- [feature-flutter-mobile.md](feature-flutter-mobile.md)
- [browser-e2e-testing.md](browser-e2e-testing.md)
- [design-system-compliance.md](design-system-compliance.md)
- [pre-commit-validation.md](pre-commit-validation.md)
