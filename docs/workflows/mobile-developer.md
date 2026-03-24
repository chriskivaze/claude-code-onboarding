# React Native & Native Mobile Development

> **When to use**: Building React Native apps, native Swift/Kotlin modules, or mobile CI/CD pipelines
> **Prerequisites**: Approved spec in `docs/specs/`, approved plan in `docs/plans/`
> **NOT for Flutter work** — use `feature-flutter-mobile.md` instead

## Overview

Mobile development workflow for React Native, native modules, and Fastlane/Codemagic/EAS CI/CD. Uses Context7 MCP for framework docs and existing asc-*/gpd-* skills for store deployment.

---

## Phases

### Phase 0 — Mobile Design Thinking (MANDATORY)

**Skill**: Load `mobile-design` skill
**Steps:**
1. Read `skills/mobile-design/reference/touch-psychology.md`
2. Read `skills/mobile-design/reference/mobile-performance.md`
3. Complete MFRI scoring
4. Fill Mobile Checkpoint — score must be >= 3 to proceed

**Gate**: Mobile Checkpoint complete; MFRI >= 3

---

### Phase 1 — Load Skill and Documentation

**Skill**: Load `mobile-developer` skill
**MCP**: Use `Context7` MCP for React Native, Expo, Swift, Kotlin docs
**Query Context7** before writing any code — confirm current API signatures

**Decision: What are you building?**
| Task | Path |
|------|------|
| New React Native app | Use Expo Router (`create-expo-app`) |
| RN feature in existing app | Follow existing navigation pattern |
| Native Swift module | Platform channel pattern from skill |
| Native Kotlin module | Platform channel pattern from skill |
| Fastlane/Codemagic CI/CD | Load skill CI/CD section |

**Gate**: Skill loaded, Context7 queried for current API

---

### Phase 2 — Implement

**React Native component build order:**
1. TypeScript types / interfaces
2. Data layer (API calls, offline storage)
3. State management (Zustand / Redux Toolkit / React Query)
4. UI components (React Native + StyleSheet)
5. Navigation (Expo Router or React Navigation)
6. Tests (Jest unit + Detox integration)

**Performance gates (from mobile-design skill):**
- Lists: FlatList with `getItemLayout` and `keyExtractor` (stable IDs)
- Animations: `useNativeDriver: true` or Reanimated
- Memoization: `React.memo` + `useCallback` for renderItem

**Gate**: `jest --coverage` passes, `eslint` clean

---

### Phase 3 — Native Module (if needed)

**For Swift/SwiftUI native module:**
1. Create `ios/NativeModuleName/` directory
2. Implement `RCTBridgeModule` (old arch) or Turbo Module spec (new arch)
3. Export to React Native via `@ReactMethod`
4. Test on iOS simulator via xcodebuild MCP

**For Kotlin/Compose native module:**
1. Create `android/app/src/main/java/.../NativeModule.kt`
2. Implement `ReactContextBaseJavaModule`
3. Export via `@ReactMethod`
4. Test on Android emulator

**Gate**: Native module tests pass on both platforms

---

### Phase 4 — CI/CD Setup

**Fastlane:**
```bash
cd ios && bundle exec fastlane beta    # TestFlight upload
cd ios && bundle exec fastlane deploy  # App Store upload
```

**EAS (Expo):**
```bash
eas build --platform ios --profile production
eas submit --platform ios
eas update --branch production --message "Release v1.0"
```

**For App Store deployment**: load `asc-release-flow` skill
**For Google Play deployment**: load `gpd-release-flow` skill

**Gate**: CI build passes, artifacts uploaded

---

### Phase 5 — Security & Review

**Agent**: `security-reviewer` — OWASP MASVS compliance
**Checks**:
- Sensitive data NOT in AsyncStorage — use `react-native-keychain` or `expo-secure-store`
- Certificate pinning configured
- No hardcoded secrets or API keys
- Code obfuscation enabled for production

**Agent**: `code-reviewer` — general code quality

**Gate**: Zero CRITICAL security findings

---

## Quick Reference

| Phase | What to Run | Gate |
|-------|-------------|------|
| 0 — Design | Mobile Checkpoint + MFRI | Score >= 3 |
| 1 — Skill | Load mobile-developer + Context7 MCP | Docs verified |
| 2 — Implement | Build components, tests | Jest passes |
| 3 — Native (opt) | Swift/Kotlin module | Simulator tests pass |
| 4 — CI/CD | Fastlane or EAS | Build artifact |
| 5 — Review | security-reviewer + code-reviewer | Zero CRITICAL |

---

## Related Workflows

- [`feature-flutter-mobile.md`](feature-flutter-mobile.md) — Flutter work (use instead of this for Flutter)
- [`ios-app-store-release.md`](ios-app-store-release.md) — App Store submission
- [`android-google-play-release.md`](android-google-play-release.md) — Google Play submission
