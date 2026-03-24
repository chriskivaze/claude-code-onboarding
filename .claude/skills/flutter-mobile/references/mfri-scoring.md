# MFRI (Mobile Feature Risk Index) Scoring

Load this file before writing any new Flutter screen. Score the feature on 4 axes and use the total to decide how much upfront design is needed.

## Scoring Axes

| Axis | 1 — Low | 2 — Medium | 3 — High |
|------|---------|-----------|---------|
| **State Complexity** | Static/read-only display | Single provider, simple async | Multiple providers, cross-feature state |
| **Navigation Depth** | Single screen | Modal or tab child | Deep nested route with params |
| **Data Mutation Risk** | No writes | Local writes only | Firestore/backend writes |
| **Platform Sensitivity** | No platform-specific code | Minor (permissions) | Camera, biometrics, notifications |

## Interpretation

| Total Score | Action |
|-------------|--------|
| 4–5 | Proceed directly — simple screen, minimal design needed |
| 6–8 | Sketch component tree and provider structure before coding |
| 9–12 | Plan full Riverpod provider hierarchy, error states, and offline behavior before writing any widget code |

## Required Before Coding (Score ≥ 6)

- [ ] Provider hierarchy mapped (what watches what)
- [ ] Loading, error, and empty states designed
- [ ] Navigation entry/exit points defined
- [ ] Offline behavior decided (cache, block, or degrade)

## Quick Checklist (All Screens)

- [ ] `ConsumerWidget` or `ConsumerStatefulWidget` — never plain `StatefulWidget` with Riverpod
- [ ] `AsyncValue.when()` handles loading/error/data states
- [ ] No `ref.read()` inside `build()` — use `ref.watch()`
- [ ] Error states show a user-visible message, never a silent empty screen
- [ ] Touch targets ≥ 48dp for all interactive elements
