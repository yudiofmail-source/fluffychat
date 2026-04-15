# FluffyChat — Agent Guide

> This file is for AI coding agents. It assumes zero prior knowledge of the project.

## Project Overview

FluffyChat is an open-source, nonprofit Matrix client written in [Flutter](https://flutter.dev). It targets Android, iOS, web, Linux, Windows, and macOS. The app uses the [Famedly Matrix Dart SDK](https://github.com/famedly/matrix-dart-sdk) (`package:matrix`) for all Matrix protocol handling (sync, rooms, encryption, etc.). End-to-end encryption relies on [Vodozemac](https://github.com/famedly/dart-vodozemac), a Rust implementation of the Double Ratchet algorithm, bridged to Dart via `flutter_rust_bridge`.

Key facts:
- **License**: AGPL-3.0
- **Primary language**: Dart (Flutter)
- **Auxiliary language**: Rust (encryption layer)
- **State management**: `Provider` + StatefulWidget controllers
- **Navigation**: `go_router`
- **Localization**: ARB files under `lib/l10n/` (Flutter gen-l10n)

## Key Configuration Files

| File | Purpose |
|------|---------|
| `pubspec.yaml` | Dart dependencies, Flutter assets, version (`2.5.1+3551`) |
| `analysis_options.yaml` | Lint rules (`flutter_lints` + `dart_code_linter` plugins) |
| `.tool_versions.yaml` | Pins Flutter version (`3.41.6`) for CI and local dev |
| `l10n.yaml` | Localization generation config (template: `intl_en.arb`) |
| `licenses.yaml` | License compliance rules (CI-enforced, AGPL-3.0 compatible) |
| `config.sample.json` | Optional runtime web config (homeserver, privacy URL, etc.) |
| `snap/snapcraft.yaml` | Snap package build config for Linux |

## Technology Stack & Runtime Architecture

- **Flutter SDK** (`>=3.11.1 <4.0.0`, pinned to `3.41.6` in `.tool_versions.yaml`).
- **Matrix SDK** (`package:matrix ^6.2.0`) — handles rooms, sync, E2EE, VoIP signaling.
- **Vodozemac** (`flutter_vodozemac`) — Rust-based Olm/Megolm implementation. For web builds, a WASM compile step is required (see `scripts/prepare-web.sh`).
- **Background push** — Android uses a shared isolate for background notification processing (`mainIsolateReceivePort` / `pushIsolatePortName`). The app can start in "detached" background-fetch mode without rendering UI.
- **Database** — `sqflite_common_ffi` + `sqlcipher_flutter_libs` on desktop; platform-specific secure storage on mobile.
- **Notifications** — `flutter_local_notifications`, `unifiedpush`, plus optional Firebase Cloud Messaging via `scripts/add-firebase-messaging.sh`.

## Code Organization

```
lib/
  main.dart              # Entry point; handles isolate ports, Vodozemac init, background/foreground mode
  config/                # Constants, themes, route table, setting keys
  pages/                 # One directory per screen; each screen uses Controller + View pattern
  utils/                 # Helper functions, Dart/Flutter extensions, Matrix SDK extensions, sign-in flows, VoIP
  widgets/               # Reusable UI components, adaptive dialogs, layouts, app shell
  l10n/                  # ARB translation files (intl_*.arb)
```

### Controller / View Separation

Every major page follows a strict split:
- **Controller**: a `StatefulWidget` whose `State` class has a `Controller` suffix. It holds state, `TextEditingController`s, and action methods.
- **View**: a `StatelessWidget` with a `View` suffix. It receives the controller as its only parameter and builds the widget tree.

Example pattern (from `CONTRIBUTING.md`):
```dart
// lib/pages/enter_name/enter_name.dart
class EnterNameController extends State<EnterName> {
  String name = 'Unknown';
  void setNameAction() => setState(() => ...);
  @override
  Widget build(BuildContext context) => EnterNameView(this);
}

// lib/pages/enter_name/enter_name_view.dart
class EnterNameView extends StatelessWidget {
  final EnterNameController controller;
  const EnterNameView(this.controller, {super.key});
  @override
  Widget build(BuildContext context) => Scaffold(...);
}
```

Rules:
- File names are `lower_snake_case`.
- Views must have a `View` suffix; controllers must have a `Controller` suffix.
- Action methods must be typed methods (not inline lambdas) and should have DartDoc comments.
- Any non-trivial logic belongs in `lib/utils/` as a helper function, not in the view.

## Build & Run Commands

Prerequisites: Flutter and Rust toolchains installed.

### General
```bash
flutter pub get
flutter run                    # Debug run on connected device/emulator
flutter test                   # Run unit/widget tests
```

### Platform-specific builds
```bash
# Android
flutter build apk

# iOS / iPadOS (macOS + Xcode required)
./scripts/build-ios.sh

# Web (requires Rust + wasm compile)
./scripts/prepare-web.sh
flutter build web --release

# Linux (install deps first; see README)
flutter build linux --release

# Windows / macOS
flutter build windows --release
flutter build macos --release
```

### Web-specific setup
`scripts/prepare-web.sh` clones the Vodozemac repository, compiles the Rust code to WASM, downloads `native_imaging` web assets, and compiles `web/native_executor.dart` to JS. You must rerun this after updating `flutter_vodozemac` or `native_imaging`.

## Testing

### Unit / Widget Tests
Located in `test/`. Run with:
```bash
flutter test
```

Notable test: `test/command_hint_test.dart` verifies that every Matrix SDK command has a matching `commandHint_*` translation in `lib/l10n/intl_en.arb`.

### Integration Tests
Located in `integration_test/`. Tests run against a real Synapse homeserver spun up via Docker.

Prerequisite: Docker installed locally.

```bash
./scripts/prepare_integration_test.sh   # Starts Synapse and seeds test users
flutter test integration_test/mobile_test.dart
```

The integration test suite covers login/logout, basic messaging, and chat archiving flows. You must rerun the preparation script before every integration test run because it resets the homeserver state.

## CI / CD & Deployment

All automation is in `.github/workflows/`.

- **`integrate.yaml`** — Pull-request gating. Steps include:
  - Conventional commit check
  - Locale config generation check (`scripts/generate-locale-config.sh`)
  - `dart format lib/ test/ --set-exit-if-changed`
  - License compliance check (`dart run license_checker`)
  - `flutter analyze`
  - Unused dependency/code/file/l10n checks via `dart_code_linter`
  - Commented-out code scan (rejects `//` lines ending in `;`)
  - `flutter test`
  - Debug builds for APK, Web, Linux (x64 + arm64), and iOS
  - Integration tests on an Android API 34 emulator

- **`release.yaml`** — Triggered on GitHub Releases. Builds and uploads:
  - Web tarball + deploys to GitHub Pages (`fluffychat.im`)
  - Android APK
  - Linux tarballs (x64 / arm64)
  - Deploys Android App Bundle to Google Play via Fastlane (`android/fastlane/Fastfile`)
  - Builds and pushes a Docker image to GHCR

- **`snap/snapcraft.yaml`** — Produces a Snap package for Linux distribution.

## Development Conventions

1. **Formatting**: run `flutter format lib` (or ensure your IDE does). CI rejects unformatted code.
2. **Linting**: zero Dart errors or warnings allowed. `analysis_options.yaml` enables strict rules including `avoid_print`, `prefer_final_locals`, `prefer_single_quotes`, `always_declare_return_types`, and Flutter-specific rules.
3. **Commits**:
   - Sign your commits.
   - Use [Conventional Commits](https://www.conventionalcommits.org) format.
   - Prefer **one commit per PR**.
   - Rebase instead of merging.
4. **PR scope**: one change per PR. For large changes, open an issue and ask maintainers for approval before coding.
5. **Dependencies**:
   - Keep `pubspec.lock` in sync (CI checks this).
   - Avoid `dependency_overrides` unless unavoidable; if used, link the upstream issue and explain when it can be removed.
6. **Translations**: new UI strings go into `lib/l10n/intl_en.arb`. CI runs `translations_cleaner` to detect unused terms.
7. **Communication Language**: 与用户沟通时请使用中文。

## Security Considerations

- **Encryption**: E2EE is handled by the `matrix` SDK + Vodozemac. Do not attempt to implement custom cryptographic primitives.
- **App Lock PIN**: Stored via `flutter_secure_storage` (key: `chat.fluffy.app_lock`).
- **Database**: SQLCipher is used where supported.
- **License compliance**: Only licenses compatible with AGPL-3.0 are permitted (see `licenses.yaml`). CI will fail if a problematic dependency is introduced.
- **Push isolates**: The Android background push isolate communicates with the main isolate via named `SendPort`/`ReceivePort` pairs. Be careful when modifying `main.dart` isolate logic.
- **Web OIDC**: The web entry point sanitizes the URL hash to avoid routing issues (`#` → `#?`).

## Useful Scripts

| Script | What it does |
|--------|--------------|
| `scripts/prepare-web.sh` | Compiles Vodozemac WASM and fetches web-native imaging assets |
| `scripts/prepare_integration_test.sh` | Starts a Dockerized Synapse for integration tests |
| `scripts/add-firebase-messaging.sh` | Patches the Android project to enable Firebase Cloud Messaging |
| `scripts/build-ios.sh` | Builds an iOS IPA with optional ID rotation and automatic device install |
| `scripts/generate-locale-config.sh` | Regenerates locale-related config files |
| `scripts/generate_command_hints_glue.sh` | Helper to regenerate Matrix command hint translation keys |

## When Making Changes

- If you add a new page, create a folder under `lib/pages/` with a controller (`*.dart`) and a view (`*_view.dart`).
- If you add a new string, add it to `lib/l10n/intl_en.arb` and run `flutter gen-l10n` (or let the IDE/CI handle it).
- If you change the version in `pubspec.yaml`, also bump the build number in `snap/snapcraft.yaml`.
- If you touch encryption, Vodozemac, or push-isolate code, run **both** unit tests and integration tests locally.
