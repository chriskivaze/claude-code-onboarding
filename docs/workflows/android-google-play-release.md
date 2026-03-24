# Android Google Play Release

> **When to use**: Releasing an Android app (AAB) to Google Play internal testing, beta, or production
> **Time estimate**: 30 min – 2 hours depending on track and rollout strategy
> **Prerequisites**: Flutter app builds cleanly (`flutter build appbundle`), Google Play Developer account, `gpd` CLI configured

## Overview

End-to-end Android release workflow using the `gpd` (Google Play Developer) CLI. Covers AAB upload, build lifecycle tracking, beta group management, pre-release health checks, and production rollout with staged delivery.

---

## Skills Reference

Load the relevant skill before each phase:

| Skill | When to load |
|-------|-------------|
| `app-store-optimization` | Phase 0 — store listing, keywords, ASO health score |
| `gpd-cli-usage` | Before any `gpd` command — covers flags, pagination, auth, safety conventions |
| `gpd-id-resolver` | When commands need package names, track names, version codes, product IDs |
| `gpd-build-lifecycle` | Phase 1 — AAB upload and build processing |
| `gpd-betagroups` | Phase 2 — internal/beta tester management |
| `gpd-submission-health` | Phase 3 — pre-release validation |
| `gpd-release-flow` | Phase 4 — track promotion and staged rollout |

---

## Phases

### Phase 0 — Store Listing Optimization (before first release and each major update)

**Skill**: Load `app-store-optimization`
**Purpose**: Optimize keywords, metadata copy, and conversion before promoting to Google Play production
**When to run**: Before first production submission; revisit before each major update or international expansion

**Steps**:
1. Keyword research — identify keywords to embed in title and description (Google Play has no separate keyword field — all keywords must appear in title or description)
2. Optimize title (50 chars) and short description (80 chars) with primary keywords front-loaded
3. Write conversion-focused full description (up to 4,000 chars) with keywords distributed naturally
4. Competitor analysis — identify keyword gaps vs top 10 apps in category
5. Run ASO health score via `aso_scorer.py` — target ≥ 70/100 before proceeding to production
6. Plan A/B test for icon and feature graphic via `ab_test_planner.py`
7. Localization — use `localization_helper.py` to prioritize markets (pt-BR is typically high ROI for Android given Play Store's strength in Brazil)

**Gate**: ASO health score ≥ 70/100; all field character limits validated; keywords appear naturally in title and description

---

### Phase 1 — Build Upload and Processing

**Skill**: Load `gpd-build-lifecycle`
**Purpose**: Upload AAB and track processing status

**Build**:
```
flutter build appbundle --release
```

**Upload** (from `gpd-build-lifecycle` skill):
```
gpd publish uploads create \
  --package-name com.example.app \
  --path build/app/outputs/bundle/release/app-release.aab
```

**Track processing**:
```
gpd publish edits validate --package-name com.example.app --edit-id <edit-id>
```

**Edit lifecycle** (critical pattern from skill):
- Every change creates an "edit" — a staging area
- Edits must be **committed** or they expire in 7 days
- Never abandon an edit with good content — commit it

**ID resolution** (from `gpd-id-resolver` skill):
- Package name: literal string (e.g., `com.example.app`)
- Track name: `internal`, `alpha`, `beta`, `production`
- Version code: integer from `versionCode` in `build.gradle`

**Produces**: AAB uploaded and processed
**Gate**: Build shows `ACTIVE` processing state; `gpd publish edits validate` returns clean

---

### Phase 2 — Beta Testing

**Skill**: Load `gpd-betagroups`
**Purpose**: Distribute to internal testers before production

**Internal track** (immediate, no review):
```
gpd publish tracks update \
  --package-name com.example.app \
  --track internal \
  --version-codes <version-code>
```

**Manage testers**:
```
# List beta groups
gpd testers list --package-name com.example.app

# Add tester
gpd testers create --package-name com.example.app --email tester@example.com

# List testers in group
gpd testers list --package-name com.example.app --track alpha
```

**Tracks in order**: `internal` (no review) → `alpha` (closed testing) → `beta` (open testing) → `production`

**Produces**: Build available to internal testers
**Gate**: At least 3 internal testers confirm app works on their devices

---

### Phase 3 — Pre-Release Health Check

**Skill**: Load `gpd-submission-health`
**Purpose**: Validate everything is ready before production

**Preflight validation** (from `gpd-submission-health` skill):
- Store listing complete (title, short description, full description per locale)
- Screenshots at required resolutions (phone, 7" tablet, 10" tablet)
- Feature graphic present (1024x500 JPEG/PNG)
- Privacy policy URL set
- Content rating questionnaire completed
- Target API level meets Google Play requirements (current year's API level)
- App signing configured (Play App Signing enabled)
- `android:debuggable` set to `false` in release build

**Commands**:
```
gpd publish listings list --package-name com.example.app
gpd publish details get --package-name com.example.app
```

**Produces**: Validation report
**Gate**: All required fields present; no missing screenshots or metadata

---

### Phase 4 — Production Release with Staged Rollout

**Skill**: Load `gpd-release-flow`
**Purpose**: Promote to production with controlled rollout

**Staged rollout pattern** (from `gpd-release-flow` skill):
```
# Start at 10% rollout
gpd publish tracks update \
  --package-name com.example.app \
  --track production \
  --version-codes <version-code> \
  --user-fraction 0.1

# Commit the edit
gpd publish edits commit --package-name com.example.app --edit-id <edit-id>
```

**Rollout progression** (monitor between each step):
- 10% → 25% → 50% → 100%
- Monitor crash rate and ANR rate in Play Console after each step
- Rollback if crash rate increases >0.5% above baseline

**Halt rollout**:
```
gpd publish tracks update \
  --package-name com.example.app \
  --track production \
  --version-codes <version-code> \
  --user-fraction 0  # halts rollout
```

**Track promotion** (promoting alpha/beta to production):
```
gpd publish tracks promote \
  --package-name com.example.app \
  --from-track beta \
  --to-track production \
  --user-fraction 0.1
```

**Produces**: App live on Google Play at staged percentage
**Gate**: Crash rate and ANR rate stable at each rollout step

---

## Quick Reference

| Phase | Skill to Load | Key Action | Gate |
|-------|--------------|------------|------|
| 0 — ASO | `app-store-optimization` | `aso_scorer.py` + `keyword_analyzer.py` | ASO score ≥ 70/100 |
| 1 — Upload | `gpd-build-lifecycle` | `gpd publish uploads create` | Build `ACTIVE` |
| 2 — Beta | `gpd-betagroups` | `gpd publish tracks update --track internal` | Testers confirm |
| 3 — Preflight | `gpd-submission-health` | `gpd publish listings list` | All checks pass |
| 4 — Production | `gpd-release-flow` | `gpd publish tracks update --user-fraction 0.1` | Crash rate stable |

---

## Common Pitfalls

- **Forgetting to commit the edit** — edits expire in 7 days; all changes are lost if not committed
- **Jumping to 100% rollout** — staged rollout catches regressions before affecting all users; always start at 10%
- **Wrong version code** — must increment for every upload; reusing version codes causes rejection
- **`android:debuggable=true` in release** — Google Play flags this; build with `--release` flag, never `--debug`
- **Missing target API level** — Google Play requires apps target the latest major Android API by specific dates
- **Edit conflict** — only one edit can be open at a time; check for open edits before creating a new one: `gpd publish edits list`

## Related Workflows

- [`feature-flutter-mobile.md`](feature-flutter-mobile.md) — build the feature before this
- [`ios-app-store-release.md`](ios-app-store-release.md) — parallel iOS release
- [`security-audit.md`](security-audit.md) — run before first Play Store submission
