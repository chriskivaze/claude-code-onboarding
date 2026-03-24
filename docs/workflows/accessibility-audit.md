# Accessibility Audit

> **When to use**: After building any Angular or Flutter UI component — before PR review, or when auditing for WCAG compliance
> **Time estimate**: 30–60 min automated setup; 1–2 hours for a full manual audit session
> **Prerequisites**: Angular or Flutter UI written; `test/accessibility/` directory created
> **Skill**: `accessibility-audit`

## Overview

4-phase audit pipeline: automated scan → manual checks → remediation → CI/CD gate. Dispatches `accessibility-auditor` agent for WCAG 2.1 AA compliance and `ui-standards-expert` for design token + interaction contract verification. Works for both Angular 21.x and Flutter 3.38.

---

## When to Use vs. Related Workflows

| Situation | Workflow |
|-----------|----------|
| UI component just built — quick a11y pass | This workflow (Phase 1 + 2 only) |
| Full feature audit before PR | This workflow (all phases) |
| Design token violations | [`design-system-compliance.md`](design-system-compliance.md) |
| Security audit | [`security-audit.md`](security-audit.md) |
| Full pre-merge gate | [`pr-shipping.md`](pr-shipping.md) |
| E2E browser testing | [`browser-e2e-testing.md`](browser-e2e-testing.md) |

---

## Phase 1 — Automated Scan

**Skill**: Load `accessibility-audit` (`.claude/skills/accessibility-audit/SKILL.md`)

### Angular — axe-core + pa11y

```bash
# Install (one-time setup)
npm install --save-dev @axe-core/playwright axe-core jest-axe @testing-library/angular

# Run axe component tests
npm run test:a11y

# Run pa11y against running dev server (port 4200)
npx ng serve &
npx pa11y-ci --config .pa11yci.json
```

**Reference**: `.claude/skills/accessibility-audit/reference/angular-a11y-automated.md`

What axe catches automatically:
- Missing alt text on images
- Icon-only buttons without `aria-label`
- Form inputs without labels
- Color contrast violations (WCAG 1.4.3)
- Missing heading structure (no h1, skipped levels)
- Missing `lang` attribute on `<html>`
- Keyboard traps

### Flutter — flutter_test SemanticsController

```bash
# Run accessibility widget tests
flutter test test/accessibility/ --reporter=expanded
```

**Reference**: `.claude/skills/accessibility-audit/reference/flutter-a11y-automated.md`

What flutter_test catches:
- Interactive elements without `Semantics(label:)`
- Missing button semantics
- Text overflow at 2x font scale
- Touch target sizes below 48dp
- Focus order via `FocusTraversalGroup`

**Gate**: Zero automated violations before proceeding to Phase 2.

---

## Phase 2 — Agent: `accessibility-auditor`

**Agent**: `accessibility-auditor` (WCAG 2.1 AA compliance)

Dispatch after automated scan passes. The agent reads your component files and cross-references against:
- `.claude/skills/angular-spa/reference/accessibility-checklist.md` (Angular)
- `.claude/skills/flutter-mobile/reference/accessibility-audit-checklist.md` (Flutter)

**Angular findings the agent checks:**

```html
<!-- ❌ CRITICAL — icon button without label -->
<button class="btn btn-circle"><mat-icon>delete</mat-icon></button>

<!-- ✅ FIX — aria-label on icon-only button -->
<button class="btn btn-circle" aria-label="Delete order ORD-001">
  <mat-icon aria-hidden="true">delete</mat-icon>
</button>

<!-- ❌ CRITICAL — form input without label -->
<input type="email" placeholder="Email" />

<!-- ✅ FIX — associated label -->
<label for="email">Email address</label>
<input id="email" type="email" aria-required="true" />

<!-- ❌ IMPORTANT — no live region for dynamic content -->
<div>{{ orderStatus() }}</div>

<!-- ✅ FIX — live region -->
<div aria-live="polite" aria-atomic="true">{{ orderStatus() }}</div>
```

**Flutter findings the agent checks:**

```dart
// ❌ CRITICAL — icon without semantic label
IconButton(icon: Icon(Icons.delete), onPressed: _deleteOrder)

// ✅ FIX — Semantics widget
Semantics(
  label: 'Delete order ORD-001',
  button: true,
  child: IconButton(icon: Icon(Icons.delete), onPressed: _deleteOrder),
)

// ❌ IMPORTANT — fixed height clips at 200% text scale
SizedBox(height: 40, child: Text(title))

// ✅ FIX — flexible height
ConstrainedBox(
  constraints: BoxConstraints(minHeight: 40),
  child: Text(title),
)
```

**Color contrast**: >= 4.5:1 for normal text, >= 3:1 for large text (18px/18sp+)

**Touch targets**: >= 48dp (Flutter) / >= 44px (Angular)

---

## Phase 3 — Manual Testing

**Reference**: `.claude/skills/accessibility-audit/reference/manual-testing-checklist.md`

Run in this order — each takes 5–10 minutes:

### 3a. Keyboard (Both Platforms)

1. Tab through the entire UI — every interactive element must receive focus
2. Activate each element with Enter or Space
3. Open any modal — verify focus traps inside; Esc closes and returns focus to trigger
4. No keyboard trap anywhere

### 3b. Screen Reader

| Platform | Tool | Shortcut |
|----------|------|---------|
| macOS / iOS | VoiceOver | Cmd+F5 |
| Windows | NVDA | Free download |
| Android | TalkBack | Hold both volume keys |

Navigate by headings -> navigate to forms -> trigger errors -> confirm announcements.

### 3c. Visual

```bash
# In Chrome DevTools -> Rendering tab:
# - Enable "Emulate CSS media feature prefers-color-scheme"
# - Enable "Emulate CSS media feature prefers-contrast: more"
# Zoom to 200% — verify no overflow
# Enable grayscale — verify all information still conveyed
```

### 3d. Cognitive (New — not in existing checklists)

- Every error message says what went wrong AND how to fix it
- Destructive actions (delete, cancel) have a confirmation step
- No form loses user input without warning
- Navigation order is consistent across all screens

**Gate**: No Critical findings. Important findings documented with file:line and fix.

---

## Phase 4 — Remediation and Re-test

For each Critical or Important finding:

1. Apply exact code fix (file:line)
2. Re-run automated scan: `npm run test:a11y` or `flutter test test/accessibility/`
3. Confirm finding no longer appears
4. Add regression test to prevent recurrence

**Regression test pattern (Angular)**:
```typescript
it('delete button has aria-label', async () => {
  const { container } = await render(OrderListComponent);
  const results = await axe(container, {
    runOnly: { type: 'rule', values: ['button-name'] },
  });
  expect(results).toHaveNoViolations();
});
```

**Regression test pattern (Flutter)**:
```dart
testWidgets('delete button has semantic label', (tester) async {
  final handle = tester.ensureSemantics();
  await tester.pumpWidget(MaterialApp(home: OrderListScreen()));
  expect(
    tester.getSemantics(find.byTooltip('Delete')),
    matchesSemantics(label: 'Delete order', isButton: true),
  );
  handle.dispose();
});
```

---

## Phase 5 — CI/CD Gate

Add automated a11y tests to the existing CI pipeline.

**Reference**: `.claude/skills/accessibility-audit/reference/cicd-integration.md`

```yaml
# Add to existing .github/workflows/ci.yml
- name: Run accessibility tests
  run: npm run test:a11y  # Angular
  # or: flutter test test/accessibility/  # Flutter
```

**Gate rule**: `--threshold 0` (pa11y) and `expect(violations).toEqual([])` (axe). Any violation fails the PR. No exceptions without explicit justification in PR description.

---

## Quick Reference

| Phase | Action | Tool | Gate |
|-------|--------|------|------|
| 1 — Automated | axe-core + pa11y (Angular) / flutter_test (Flutter) | CLI | Zero violations |
| 2 — Agent | WCAG 2.1 AA code review | `accessibility-auditor` | PASS |
| 3 — Manual | Keyboard, screen reader, visual, cognitive | Human + DevTools | No Critical findings |
| 4 — Remediate | Fix findings, add regression tests | Code + tests | All Critical/Important resolved |
| 5 — CI/CD | Automated gate on every PR | GitHub Actions | Build green |

---

## Common Pitfalls

- **Axe passes but screen reader fails** — axe catches structural violations; screen reader testing catches announcement timing and reading order
- **Fixing the symptom not the cause** — adding `aria-label` to a `<div>` when it should be a `<button>`
- **Decorative images missed** — `<img>` without `alt=""` fails axe; Flutter `Image` without `ExcludeSemantics` adds noise to screen readers
- **Focus not returning after modal close** — must explicitly return focus to the trigger element
- **Cognitive check skipped** — automated tools cannot catch unhelpful error messages or missing confirmation dialogs
- **Re-test skipped after fix** — mark finding as resolved only after re-running the automated scan

## Related Workflows

- [`design-system-compliance.md`](design-system-compliance.md) — token enforcement + accessibility combined gate
- [`feature-angular-spa.md`](feature-angular-spa.md) — Phase 7 runs `accessibility-auditor` as part of feature workflow
- [`feature-flutter-mobile.md`](feature-flutter-mobile.md) — Phase 5 runs `accessibility-auditor` as part of feature workflow
- [`browser-e2e-testing.md`](browser-e2e-testing.md) — E2E flows include keyboard navigation as part of user journey
- [`clean-code-review.md`](clean-code-review.md) — code quality baseline before a11y review
