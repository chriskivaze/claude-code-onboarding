# iOS App Store Release

> **When to use**: Releasing an iOS or macOS app to TestFlight (beta) or App Store (production)
> **Time estimate**: 1–3 hours for initial setup; 30 min per subsequent release
> **Prerequisites**: Flutter app builds cleanly (`flutter build ipa`), Apple Developer account, `asc` CLI configured

## Overview

End-to-end iOS release workflow using the `asc` CLI (App Store Connect). Covers code signing setup, TestFlight beta distribution, pre-submission health checks, App Store submission, and post-release crash monitoring.

---

## Skills Reference

Load the relevant skill before each phase:

| Skill | When to load |
|-------|-------------|
| `app-store-optimization` | Phase 0 — store listing, keywords, ASO health score |
| `asc-cli-usage` | Before any `asc` command — covers flags, pagination, auth |
| `asc-id-resolver` | When commands need bundle IDs, app IDs, build IDs |
| `asc-signing-setup` | Phase 1 — code signing configuration |
| `asc-testflight-orchestration` | Phase 2 — TestFlight beta management |
| `asc-submission-health` | Phase 3 — pre-submission validation |
| `asc-release-flow` | Phase 4 — App Store submission |
| `asc-crash-triage` | Phase 5 — post-release monitoring |

---

## Phases

### Phase 0 — Store Listing Optimization (before first release and each major update)

**Skill**: Load `app-store-optimization`
**Purpose**: Optimize keywords, metadata copy, and conversion before submitting to App Store review
**When to run**: Before first App Store submission; revisit before each major update or international expansion

**Steps**:
1. Keyword research — identify high-volume, lower-competition keywords for your category
2. Optimize title (30 chars), subtitle (30 chars), keyword field (100 chars — no spaces after commas, no plurals, no words already in title)
3. Write conversion-focused description (up to 4,000 chars)
4. Competitor analysis — identify keyword gaps and visual asset opportunities vs top 10 apps in category
5. Run ASO health score via `aso_scorer.py` — target ≥ 70/100 before proceeding
6. Plan A/B test for icon and first 2 screenshots via `ab_test_planner.py`
7. Localization — use `localization_helper.py` to assess ROI for additional markets beyond en-US

**Gate**: ASO health score ≥ 70/100; all field character limits validated; keyword field has no duplicates or spaces between commas

---

### Phase 1 — Code Signing Setup (first time only)

**Skill**: Load `asc-signing-setup`
**Covers**: Bundle IDs, capabilities, signing certificates, provisioning profiles

**Steps**:
1. Register bundle ID: `asc bundle-ids create --identifier com.example.app --name "My App"`
2. Enable capabilities (Push Notifications, Sign in with Apple, etc.): `asc bundle-ids capabilities`
3. Create distribution certificate: `asc certificates create --type IOS_DISTRIBUTION`
4. Download certificate and install in Keychain
5. Create provisioning profile: `asc profiles create --bundle-id <id> --certificates <cert-id>`
6. Download profile: `asc profiles download --id <profile-id> --output ./profiles/`
7. Configure Xcode project: update `exportOptions.plist`

**ID resolution** (when you have names but need IDs):
- Skill: `asc-id-resolver`
- Example: `asc bundle-ids list` → find ID by identifier string

**Produces**: Signed `.ipa` ready for upload
**Gate**: `asc certificates list` shows valid distribution certificate; `flutter build ipa --export-options-plist exportOptions.plist` succeeds

---

### Phase 2 — TestFlight Beta Distribution

**Skill**: Load `asc-testflight-orchestration`
**Purpose**: Distribute beta builds to testers before App Store review

**Steps**:
1. Upload build: `asc builds upload --path build/ios/ipa/*.ipa`
2. Wait for processing: `asc builds list --app-id <app-id>` (check `processingState`)
3. Create TestFlight group (if new): `asc beta-groups create --app-id <app-id> --name "QA Team"`
4. Add testers: `asc beta-testers add --group-id <group-id> --email tester@example.com`
5. Set "What to Test" notes: `asc builds test-info update --build-id <build-id> --whats-new "..."`
6. Distribute to group: `asc builds beta-groups add --build-id <build-id> --group-id <group-id>`

**ID resolution** (from `asc-id-resolver` skill):
- App ID: `asc apps list --filter-bundle-id com.example.app`
- Build ID: `asc builds list --app-id <app-id> --filter-version <version>`
- Group ID: `asc beta-groups list --app-id <app-id>`

**Produces**: Build distributed to beta testers
**Gate**: Testers receive TestFlight invite email; build shows as "Testing" in App Store Connect

---

### Phase 3 — Pre-Submission Health Check

**Skill**: Load `asc-submission-health`
**Purpose**: Validate everything is ready before submitting for App Store review

**Preflight checks** (from `asc-submission-health` skill):
- App version string is updated
- Build number is incremented (unique per version)
- Privacy manifest (`PrivacyInfo.xcprivacy`) present if using tracked APIs
- Export compliance declarations complete
- App Store screenshots at correct resolutions (6.7" and 6.5" for iPhone, 12.9" for iPad)
- App description, keywords, support URL present
- Age rating configured
- In-app purchases approved (if applicable)

**Commands**:
```
asc apps versions list --app-id <app-id>
asc builds review-submissions list
```

**Produces**: Validation report with pass/fail per requirement
**Gate**: All required items pass; no missing metadata

---

### Phase 4 — App Store Submission

**Skill**: Load `asc-release-flow`
**Purpose**: Submit to App Store review

**Steps**:
1. Create new App Store version: `asc apps versions create --app-id <app-id> --version 1.2.0 --platform IOS`
2. Set release notes (per locale): `asc apps versions localizations update --version-id <id> --whats-new "Bug fixes and performance improvements"`
3. Attach build to version: `asc apps versions builds add --version-id <id> --build-id <build-id>`
4. Submit for review: `asc review-submissions create --app-id <app-id> --version-id <id>`
5. Monitor review status: `asc review-submissions list --app-id <app-id>`

**Review states**: `WAITING_FOR_REVIEW` → `IN_REVIEW` → `APPROVED` or `REJECTED`

**If rejected**: Check Resolution Center in App Store Connect; address the specific guideline violation

**Produces**: App in App Store Review queue
**Gate**: Status reaches `APPROVED`

---

### Phase 5 — Post-Release Monitoring

**Skill**: Load `asc-crash-triage`
**Purpose**: Monitor crashes and performance after release

**Commands**:
```
asc diagnostics list --app-id <app-id>       # Crash reports
asc feedback list --app-id <app-id>           # Beta feedback
asc power-performance-metrics list            # Performance data
```

**Triage process**:
1. Check crash reports daily for first week post-release
2. Group crashes by signal/exception type
3. Cross-reference with build version
4. Prioritize by affected users count
5. File `bugfix/` PR for CRITICAL crashes

**Gate**: No P0 crashes affecting >1% of users within 48h of release

---

## Quick Reference

| Phase | Skill to Load | Key Action | Gate |
|-------|--------------|------------|------|
| 0 — ASO | `app-store-optimization` | `aso_scorer.py` + `keyword_analyzer.py` | ASO score ≥ 70/100 |
| 1 — Signing | `asc-signing-setup` | `asc certificates create` | `flutter build ipa` succeeds |
| 2 — TestFlight | `asc-testflight-orchestration` | `asc builds upload` | Testers receive invite |
| 3 — Preflight | `asc-submission-health` | `asc apps versions list` | All checks pass |
| 4 — Submit | `asc-release-flow` | `asc review-submissions create` | Status: APPROVED |
| 5 — Monitor | `asc-crash-triage` | `asc diagnostics list` | No P0 crashes at 48h |

---

## Common Pitfalls

- **Expired certificates** — distribution certificates are valid 1 year; `asc-signing-setup` covers rotation
- **Build number not incremented** — App Store Connect rejects duplicate build numbers; increment even for patches
- **Missing privacy manifest** — required for apps using certain Apple APIs (location, contacts, etc.) since iOS 17
- **Screenshots wrong resolution** — App Store Connect rejects submissions with wrong screenshot sizes; check the required sizes in `asc-submission-health` skill
- **Not waiting for build processing** — upload succeeds but build isn't available for 5–20 minutes; poll `processingState`
- **asc ID confusion** — app ID ≠ bundle ID ≠ build ID; use `asc-id-resolver` skill to get the right identifier

## Related Workflows

- [`feature-flutter-mobile.md`](feature-flutter-mobile.md) — build the feature before this
- [`android-google-play-release.md`](android-google-play-release.md) — parallel Android release
- [`security-audit.md`](security-audit.md) — run before first App Store submission
