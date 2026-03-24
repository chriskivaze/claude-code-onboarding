---
name: flutter-mobile
description: This skill provides patterns and templates for Flutter 3.38 / Dart 3.11 cross-platform mobile development. It should be activated when building Flutter screens, Riverpod providers, Freezed models, or widget tests.
allowed-tools: Bash, Read, Write, Edit
metadata:
  triggers: Flutter, Dart, Riverpod, Flutter widget, Flutter app, mobile development, iOS, Android, cross-platform mobile, Flutter screen
  related-skills: riverpod-patterns, ui-standards-tokens
  domain: frontend
  role: specialist
  scope: implementation
  output-format: code
last-reviewed: "2026-03-14"
---

## Iron Law

**NO FLUTTER UI WITHOUT RUNNING THE MFRI RISK SCORE FIRST — load `references/mfri-scoring.md` before writing any screen**

# Flutter Mobile Development Skill

## Code Conventions
- Use **Riverpod** for state management
- Follow feature-first folder structure: `lib/features/<feature>/`
- Separate `data/`, `domain/`, `presentation/` layers (clean architecture)
- Use `freezed` for immutable models
- Firebase integration via `firebase_core`, `cloud_firestore`, `firebase_auth`

## Quick Scaffold

```bash
flutter create --org com.company --platforms ios,android my_app
cd my_app

# Add core dependencies
flutter pub add flutter_riverpod riverpod_annotation
flutter pub add freezed_annotation json_annotation go_router firebase_core cloud_firestore firebase_auth
flutter pub add dev:riverpod_generator dev:freezed dev:json_serializable dev:build_runner dev:mocktail

# Run code generation
dart run build_runner build --delete-conflicting-outputs
```

## Before Writing Any UI Code

Before creating or modifying any widget, screen, or visual component:

1. **Run MFRI scoring** — read `reference/mfri-scoring.md` and complete the checkpoint template. Score < 3 = stop and redesign. Score 3-5 = add a validation milestone.
2. **Load `ui-standards-tokens` skill** — read `reference/ui-design-tokens.md` for spacing, color, typography, radius tokens
3. **Verify token awareness** — can you name the spacing token (`AppSpacing.md`), color approach (`Theme.of(context).colorScheme`), and typography pattern (`Theme.of(context).textTheme`) you will use?
4. If not → read the reference file before writing any widget code
5. For accessibility: read `reference/ui-accessibility-patterns.md` before adding interactive elements

## Process

1. **Read templates** - Use Read tool on `reference/flutter-templates.md` for all code templates (Freezed models, Riverpod providers, screens, GoRouter, tests, Firebase integration)
2. **Create feature structure** - Build `lib/features/<feature>/data/`, `domain/`, `presentation/` directories
3. **Define models** - Create Freezed data models in `data/models/` with Firestore serialization
4. **Build providers** - Create Riverpod notifiers with `@riverpod` annotation in `presentation/providers/`
5. **Design screens** - Build `ConsumerWidget` screens that watch `AsyncValue<T>` providers
6. **Run codegen** - Execute `dart run build_runner build --delete-conflicting-outputs`
7. **Write tests** - Create widget tests with `ProviderScope` overrides

## Key Patterns

| Pattern | Description |
|---------|-------------|
| `@freezed` models | Immutable data classes with `fromFirestore` factory |
| `@riverpod` providers | Code-generated state notifiers with `AsyncValue` |
| `ConsumerWidget` | Widgets that watch providers via `ref.watch()` |
| `AsyncValue.when()` | Handle loading/error/data states declaratively |
| Clean Architecture | Separate data/domain/presentation layers |
| GoRouter | Declarative routing with path parameters |
| Firebase Auth | Stream-based auth state with `authStateChanges()` |
| Firestore snapshots | Real-time data with `.snapshots()` streams |

## Modern Flutter Architecture (2025/2026)

For architecture patterns, accessibility, performance, UX, and premium polish guidelines:

Read [reference/flutter-architecture-patterns.md](reference/flutter-architecture-patterns.md) — Sealed classes, Result types, Riverpod AsyncNotifier
Read [reference/flutter-performance-ux.md](reference/flutter-performance-ux.md) — Accessibility, performance, haptic feedback, shimmer, animations
Read [reference/flutter-design-polish.md](reference/flutter-design-polish.md) — Glassmorphism, premium cards, dark/light themes, gradients
Read [reference/accessibility-audit-checklist.md](reference/accessibility-audit-checklist.md) — WCAG 2.1 audit checklist (used by `accessibility-auditor` agent)
Read [reference/flutter-security-hardening.md](reference/flutter-security-hardening.md) — Security hardening & privacy compliance (used by `flutter-security-expert` agent)

## Documentation Sources

Before generating code, consult these sources for current syntax and APIs:

| Source | URL / Tool | Purpose |
|--------|-----------|---------|
| Flutter / Dart | `Dart MCP server` | Latest Flutter widgets, Dart syntax, platform APIs |
| Riverpod | `Context7` MCP | Provider types, ref usage, AsyncValue patterns |
| Firebase Firestore | `Firebase MCP server` | Firestore operations, rules validation, auth flows |

## iOS Build, Run & Debug (XcodeBuildMCP)

For iOS-specific workflows, use the `xcodebuild` MCP server instead of raw CLI commands. It provides structured output, error parsing, and debugging that `flutter build ios` via Bash cannot.

### When to Use Which

| Task | Use This | Not This |
|------|----------|----------|
| Build iOS for simulator | `build_sim` (MCP) | `flutter build ios --simulator` (Bash) |
| Build + install + launch on simulator | `build_run_sim` (MCP) | `flutter run` (Bash) |
| Build for physical device | `build_device` (MCP) | `flutter build ios` (Bash) |
| Debug a crash or inspect state | `debug_attach_sim` → `debug_variables` (MCP) | Manual Xcode debugging |
| Capture runtime logs | `start_sim_log_cap` / `stop_sim_log_cap` (MCP) | Reading console manually |
| Check signing/scheme config | `show_build_settings` / `list_schemes` (MCP) | `xcodebuild -showBuildSettings` (Bash) |
| UI interaction (tap, swipe, type) | `tap`, `swipe`, `type_text` (MCP) | Not available via Bash |
| Build Android | `flutter build apk` (Bash) | N/A — XcodeBuildMCP is iOS/macOS only |

### iOS Debug Workflow

```
1. discover_projs → find ios/Runner.xcworkspace
2. list_schemes → identify the Runner scheme
3. build_run_sim → build, install, and launch on simulator
4. start_sim_log_cap → begin capturing logs
5. [reproduce the issue]
6. debug_attach_sim → attach LLDB debugger
7. debug_breakpoint_add → set breakpoint at suspect line
8. debug_variables / debug_stack → inspect state
9. stop_sim_log_cap → get captured logs
```

### Gotchas

- **Flutter projects**: The Xcode project is at `ios/Runner.xcworkspace`, not the project root
- **First build**: Run `flutter build ios` once first to generate the Xcode project and Pods
- **Signing**: `show_build_settings` exposes signing config — use this to diagnose code signing failures
- **Scheme name**: Flutter apps use the `Runner` scheme by default

## E2E Testing (Maestro MCP)

For cross-platform E2E testing, use the `maestro` MCP server. It runs test flows on both iOS simulator and Android emulator from natural language prompts.

### When to Use Which Test Tool

| Test Type | Tool | When |
|-----------|------|------|
| Widget/unit tests | `flutter test` (Bash) or `dart-mcp-server` `run_tests` | Every feature — fast, isolated, no device needed |
| iOS build + quick visual check | `xcodebuild` MCP (`build_run_sim`, `screenshot`) | Spot-checking a screen during development |
| Repeatable E2E flows | `maestro` MCP | Login, checkout, CRUD journeys — saved and rerunnable |
| Cross-platform E2E | `maestro` MCP | Same YAML flow runs on iOS simulator AND Android emulator |
| CI/CD regression suite | `maestro test test/e2e/` (Bash) | Pre-merge gate in GitHub Actions |

### E2E Workflow

```
1. Build and launch app (XcodeBuildMCP for iOS, flutter run for Android)
2. Describe the test flow in natural language
3. Claude generates YAML via Maestro MCP and runs it
4. Claude reports pass/fail per step
5. Save generated YAML to test/e2e/<flow-name>.yaml for reuse
```

### Flow File Conventions

- Save flows to `test/e2e/` at the project root
- Name files by user journey: `login-flow.yaml`, `create-workout-flow.yaml`
- Use `appId` matching your app's bundle ID
- One flow per file — keep flows focused on a single journey

### Gotchas

- **App must be running**: Maestro interacts with a live app — build and launch first
- **Accessibility labels**: Maestro finds elements by text and accessibility labels — ensure `Semantics` widgets have labels
- **Timing**: Use `waitForAnimationToEnd` or `extendedWaitUntil` for slow transitions, not hardcoded sleeps
- **Cross-platform element names**: iOS and Android may render text differently — use `id` attributes for reliable cross-platform selectors

## Common Commands

```bash
flutter run                          # Run on connected device/emulator
flutter test                         # Run tests
flutter build apk                    # Build Android APK
flutter build ios                    # Build iOS (also generates Xcode project)
flutter pub get                      # Install dependencies
flutter clean                        # Clean build artifacts
dart run build_runner build --delete-conflicting-outputs  # Run code generation
flutter analyze                      # Static analysis
```

## Error Handling

**Build runner fails**: Delete `.dart_tool/build/`, run `flutter clean`, retry codegen

**Missing generated files**: Ensure `part` directives match filename (e.g., `part 'user_model.g.dart';`)

**Provider not found**: Run `dart run build_runner build`, import generated `.g.dart` file

**Firestore Timestamp errors**: Use `(data['createdAt'] as Timestamp).toDate()` in `fromFirestore`

**Hot reload breaks state**: Restart app fully when changing provider signatures

**AsyncValue stuck loading**: Check repository returns data, use `AsyncValue.guard()` to catch errors

## Hard Prohibitions

- No `Navigator.push` with raw strings — use GoRouter type-safe routes exclusively
- No returning null on async error — use `AsyncValue.error`, never null/empty fallbacks
- No `Color(0x...)` or `Colors.*` — use `Theme.of(context).colorScheme.*`
- No raw `EdgeInsets` with numeric values — use `AppSpacing.*` tokens
- No inline `TextStyle(fontSize: ...)` — use `Theme.of(context).textTheme.*`

## Post-Code Review

After writing Dart code, dispatch these reviewer agents:
- `riverpod-reviewer` — state management, provider types, AsyncValue handling
- `flutter-security-expert` — secure storage, certificate pinning, data protection
- `accessibility-auditor` — WCAG 2.1 compliance, Semantics widgets, touch targets

## Templates Reference

For all code templates (pubspec.yaml, Freezed models, Riverpod providers, screen widgets, GoRouter config, widget tests, Firebase integration, repository patterns):

Read `reference/flutter-templates.md`
