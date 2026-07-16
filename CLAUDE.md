# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

SwitchGo SDK demo app — a reference integration for the SwitchGo hardware diagnostics & control SDK (`com.rhizo.switchgo`), which communicates with TemiGoPro smart delivery robot MCUs over USB HID. The app controls doors, ambient/interior lighting, turn signals, battery indicators, reads sensor states, and performs MCU firmware upgrades.

The SDK itself is distributed as `app/libs/SwitchGoLibrary_1.1.8.aar` and auto-initializes via `ContentProvider`.

## Build & test commands

```bash
# Build debug APK
./gradlew assembleDebug

# Build release APK
./gradlew assembleRelease

# Run unit tests (JVM, no Android dependency)
./gradlew test

# Run a single unit test class
./gradlew test --tests "com.example.switchgo.ExampleUnitTest"

# Run instrumented tests (requires connected device/emulator)
./gradlew connectedAndroidTest

# Clean build
./gradlew clean
```

APK output: `app/build/outputs/apk/<variant>/SwitchGo-<version>.apk`

## Architecture

**Single-module Android app** (`app/`) with MVVM using `AndroidViewModel` and view binding.

### Layer map

| Layer | Location | Role |
|-------|----------|------|
| UI | `MainActivity.kt` | Button-driven control panel; observes ViewModel StateFlows |
| ViewModel | `viewmodel/SwitchGoViewModel.kt` | All business logic; wraps SDK calls, manages door-toggle state, log buffer, firmware update lifecycle |
| USB | `usb/UsbPermissionHandler.kt` | BroadcastReceiver-based USB permission request/response for the HID device |
| Parser | `data/parser/StatusParser.kt` | Pure-function hex protocol parser — no Android deps, fully unit-testable |
| Models | `data/model/SwitchStates.kt` | `SwitchStates` data class + sealed classes/enums for door motors, limits, battery, firmware update state |

### Key SDK integration points

- **SDK entry**: `SwitchGo.getInstance(application)` — returns the singleton; SDK auto-inits via `ContentProvider`
- **Connection lifecycle**: `startAutoConnect()` / `stopAutoConnect()` called in `onResume`/`onPause`; connection state observed via `switchGo.isConnected` (a `Flow<Boolean>`)
- **SDK namespace**: `com.rhizo.switchgo` — the AAR is a closed-source binary; all hardware I/O goes through it
- **MCU update callback**: `McuUpdateCallback` interface with `onUpdateStart`, `onProgress(len)`, `onFail(str)`, `onSuccess`

### Data flow

```
USB HID device → SwitchGo SDK (AAR) → ViewModel (coroutine + StateFlow) → MainActivity (collectLatest) → UI
                                                                           → StatusParser → SwitchStates model
```

### MCU protocol (StatusParser)

The MCU returns a space-delimited hex string (26+ bytes). The parser skips a 5-byte prefix and extracts door motors, open/close limits, space panels, battery SoC, auxiliary connection, and resistance alarms from fixed byte offsets. See `StatusParser.parse()` doc comment for the full offset table.

## Dependencies

- **Kotlin** 2.0.0, **AGP** 8.5.0, **Java** 17 target
- **Coroutines** 1.7.3 for async SDK calls and Flow
- **Lifecycle ViewModel** 2.6.2 for `AndroidViewModel`
- **utilcodex** (com.blankj) for general Android utilities
- **Material** 1.10.0 + ConstraintLayout for UI
- Version catalog at `gradle/libs.versions.toml`

## Firmware files

Firmware binaries (`SwitchGoTopApp.bin`, `SwitchGoBottomApp.bin`) are expected in `app/src/main/assets/`. On first launch, the ViewModel copies them to internal storage. Firmware update buttons are hidden behind a 5-tap easter egg on a transparent trigger area.

## Commit conventions

- Never add `Co-Authored-By: Claude <noreply@anthropic.com>` or similar auto-generated signatures to commit messages.
