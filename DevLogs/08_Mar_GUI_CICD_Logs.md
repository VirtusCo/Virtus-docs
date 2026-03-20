# Porter Robotics -- Developer Log

### **Date:** 08 Mar 2026

### **Engineer:** Claude (AI) + Antony Austin (review)

### **Subsystem:** Flutter GUI Polish, CI/CD Flutter Integration, E-Stop Bug Fix

---

## 1. Summary

Three areas of work completed in this session:

1. **Flutter GUI Performance Optimization** -- reduced RAM and CPU overhead across the entire app
2. **CI/CD Flutter Build Integration** -- added Flutter Linux binary build and verification to GitHub Actions, redesigned release notes template
3. **E-Stop Bug Fix** -- emergency stop button could not be disengaged after activation

**Key results:**
- 23 unused transitive packages removed (google_fonts, cupertino_icons dropped)
- Streaming rebuild rate reduced 75% (50 Hz to 12.5 Hz)
- New `build-flutter-gui` job in `build-release.yml` -- packages `porter-gui-linux-x64-VERSION.tar.gz`
- New `flutter-verify` job in `verify.yml` -- analyze + test + build on every push/PR
- Release notes redesigned with categorized downloads, collapsible Quick Start, conventional commit changelog
- E-stop disengage (long-press) now works correctly
- `version-bump.sh` now syncs Flutter `pubspec.yaml` version
- Version: `0.3.2`

---

## 2. Flutter GUI Performance Optimization

### 2.1 Problem

The GUI was using unnecessary RAM from unused packages and suboptimal patterns:
- `google_fonts` (pulls 23 transitive packages) was unused -- app uses system fonts
- `cupertino_icons` was unused -- app uses Material icons only
- Chat streaming rebuilt widgets at 50 Hz (20ms timer, 2 chars per tick)
- No message list caching -- `UnmodifiableListView` created on every access
- No cap on stored messages -- unlimited memory growth
- All 3 animations lacked `RepaintBoundary` -- full repaints on every frame
- Rosbridge reconnect used fixed 3s interval with no backoff or limit

### 2.2 Changes

**providers.dart:**
- Streaming rate: 20ms/2chars --> 80ms/8chars (75% fewer rebuilds)
- Cached `_cachedMessages` with `_mutated()` invalidation pattern
- 100 message cap (removes oldest when exceeded)
- Health check interval: 10s --> 30s
- Centralized `_mutated()` method for cache invalidation + notify

**chat_screen.dart:**
- `RepaintBoundary` on all 3 animations (pulsing dot, gradient, opacity)
- `AnimatedCrossFade` replaced with `AnimatedSize` (simpler, fewer widgets)
- `addAutomaticKeepAlives: false` on ListView builder
- Smart auto-scroll: only when user is near bottom (within 150px)

**pubspec.yaml:**
- Removed `google_fonts` and `cupertino_icons` -- 23 transitive packages eliminated

**ros_bridge_service.dart:**
- Exponential backoff: 3s initial, 2x growth, 60s max
- Max 20 reconnect attempts (was unlimited)

### 2.3 Verification

- `flutter analyze` -- no issues
- `flutter test` -- 9/9 pass
- `flutter build linux --release` -- 50 MB bundle, success
- 23 packages removed from dependency tree

---

## 3. CI/CD Flutter Integration

### 3.1 build-release.yml Changes

Added **Job 4: `build-flutter-gui`** (between ESP32 firmware and release):
- Runs on `ubuntu-24.04`
- Installs Linux build deps: `cmake`, `ninja-build`, `pkg-config`, `libgtk-3-dev`, `clang`, `lld`
- Uses `subosito/flutter-action@v2` with stable channel + cache
- Runs `flutter pub get`, `flutter analyze`, `flutter test`, `flutter build linux --release`
- Packages bundle as `porter-gui-linux-x64-VERSION.tar.gz`
- Uploads as `flutter-gui` artifact

**Release job updated:**
- `needs` now includes `build-flutter-gui`
- Downloads Flutter GUI artifact alongside Docker image and ESP32 firmware
- FILES array collects GUI tarball first, then Docker, then firmware

**Release notes redesigned:**
- Clean metadata table (version, branch, linked commit hash, date, Zephyr)
- Three categorized download sections: GUI Application, Docker Image, ESP32 Firmware
- File sizes displayed in download tables
- Collapsible `<details>` Quick Start sections (GUI, Docker, ESP32 Flash)
- Conventional commit changelog with type labels (`[feat]`, `[fix]`, `[docs]`, etc.)
- No emojis

Pipeline is now 5 jobs: version --> build-ros2-docker + build-esp32-firmware + build-flutter-gui --> release

### 3.2 verify.yml Changes

Added **Job 4: `flutter-verify`** (between ros2-lint and esp32-tests):
- Same Flutter setup (subosito/flutter-action@v2 with cache)
- Installs Linux build deps
- Runs `flutter pub get`, `flutter analyze`, `flutter test`, `flutter build linux --release`
- Step summary reports pass/fail

**Verification gate updated:**
- Now checks 8 jobs (was 7): added Flutter GUI
- Summary table includes Flutter GUI row
- Gate message: "All 8 verification jobs passed."

Pipeline is now 9 jobs: version --> ros2-build-test + ros2-lint + flutter-verify + esp32-tests --> esp32-firmware --> docker-verify --> integration --> verification-gate

### 3.3 version-bump.sh

Added Flutter pubspec.yaml sync:
- After updating `package.xml` files, also updates `src/porter_gui/pubspec.yaml` version field
- Pattern: `version: X.Y.Z+1` (build number stays at 1)

---

## 4. E-Stop Bug Fix

### 4.1 Problem

When the emergency stop button was engaged (tapped), it turned red and showed "E-STOP ON" -- but long-pressing to disengage did nothing. The button was permanently stuck in engaged state.

### 4.2 Root Cause

Flutter's `GestureDetector` requires competing gesture recognizers to properly disambiguate long press from tap. When engaged:
- `onTap` was set to `null` (no tap handler registered)
- `onLongPress` was set to show the disengage dialog

With `onTap: null`, no tap recognizer entered the gesture arena. Without a competing tap recognizer, the long press recognizer has nothing to win against and Flutter's gesture disambiguator never triggers the long press callback.

### 4.3 Fix

Changed `onTap: null` to `onTap: () {}` (no-op) when engaged. This registers a tap recognizer that enters the arena, allowing the long press recognizer to properly compete and fire after the long-press delay.

**Files changed:**
- `lib/main.dart` -- `_FloatingEStop` widget (floating button on all screens)
- `lib/screens/follow_me_screen.dart` -- `_EmergencyStopButton` widget (large button on Follow Me screen)

Both instances had the same bug and received the same fix.

---

## 5. Files Changed

| File | Change |
|------|--------|
| `src/porter_gui/lib/providers/providers.dart` | Streaming rate, message caching, health interval, message cap |
| `src/porter_gui/lib/screens/chat_screen.dart` | RepaintBoundary, AnimatedSize, smart auto-scroll |
| `src/porter_gui/lib/services/ros_bridge_service.dart` | Exponential backoff, max reconnect attempts |
| `src/porter_gui/pubspec.yaml` | Removed google_fonts + cupertino_icons, version 0.3.2 |
| `src/porter_gui/lib/main.dart` | E-stop onTap fix |
| `src/porter_gui/lib/screens/follow_me_screen.dart` | E-stop onTap fix |
| `.github/workflows/build-release.yml` | Flutter build job, release notes redesign, GUI artifact |
| `.github/workflows/verify.yml` | Flutter verify job, 8-job gate |
| `version-bump.sh` | Flutter pubspec.yaml sync |

---

## 6. Test Results

| Suite | Tests | Status |
|-------|-------|--------|
| Flutter analyze | -- | No issues |
| Flutter test | 9/9 | Pass |
| Flutter build linux --release | -- | Success (50 MB) |
| YAML validation (both workflows) | -- | Valid |
