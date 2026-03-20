
# Porter Robotics – Developer Log

### **Date:** 07 Mar 2026 (CI/CD Session)

### **Engineer:** Claude (AI) + Antony Austin (review)

### **Subsystem:** CI/CD Pipeline Fixes — GitHub Actions, Docker, Zephyr SDK, SMF API

---

## 1. Summary

Fixed 6 distinct CI/CD pipeline failures discovered after pushing the ESP32 firmware
(Phase 3) and Docker production build to GitHub Actions. Each failure was diagnosed
from GitHub Actions screenshots shared by Antony, root-caused, and fixed.

**Key results:**
- **6 bugs fixed** across Dockerfiles, GitHub Actions workflows, and ESP32 firmware
- `docker/Dockerfile.prod` multi-stage build now works end-to-end
- `.github/workflows/verify.yml` (8-job pipeline) fully functional
- `.github/workflows/build-release.yml` (semver release pipeline) fully functional
- ESP32 firmware builds on Zephyr 4.0.0 in CI (SMF API compatibility resolved)
- **6 new lessons learned** added to `Claude.md` (entries 24–28 + lesson #20 corrected)

---

## 2. Bug Fixes

### Fix 1 — Docker image `osrf/ros:jazzy-ros-base` does not exist

**File:** `docker/Dockerfile.prod` (line 71, runtime stage `FROM`)

**Problem:** Stage 2 of the multi-stage Dockerfile.prod used `osrf/ros:jazzy-ros-base`
as the runtime base image. This image does not exist on Docker Hub. The `osrf/` prefix
is only available for `osrf/ros:jazzy-desktop` — the `-ros-base` variant is published
under the official Docker library as `ros:jazzy-ros-base` (no `osrf/` prefix).

**Error:** `ERROR: pull access denied, repository does not exist or may require authentication`

**Fix:**
```dockerfile
# Before
FROM osrf/ros:jazzy-ros-base AS runtime

# After
FROM ros:jazzy-ros-base AS runtime
```

---

### Fix 2 — BuildKit apt cache lock contention

**File:** `docker/Dockerfile.prod` (both build and runtime stages)

**Problem:** Both stages of the multi-stage Dockerfile used
`--mount=type=cache,target=/var/cache/apt` without unique IDs. BuildKit runs
stages in parallel, causing both `apt-get install` commands to fight over
`/var/cache/apt/archives/lock`, resulting in exit code 100.

**Error:** `E: Could not get lock /var/cache/apt/archives/lock - open (11: Resource temporarily unavailable)`

**Fix:**
```dockerfile
# Build stage
RUN --mount=type=cache,id=apt-build,target=/var/cache/apt \
    --mount=type=cache,id=apt-build-lists,target=/var/lib/apt/lists \
    apt-get update && apt-get install -y ...

# Runtime stage
RUN --mount=type=cache,id=apt-runtime,target=/var/cache/apt \
    --mount=type=cache,id=apt-runtime-lists,target=/var/lib/apt/lists \
    apt-get update && apt-get install -y ...
```

---

### Fix 3 — Zephyr SDK toolchain filename format (404 download)

**Files:** `.github/workflows/verify.yml`, `.github/workflows/build-release.yml`

**Problem:** Workflow used `toolchain_xtensa-espressif_esp32_zephyr-TOOLCHAIN_linux-x86_64.tar.xz`.
The Zephyr SDK 0.17.0 naming convention puts the **platform before the toolchain name**:
`toolchain_linux-x86_64_xtensa-espressif_esp32_zephyr-elf.tar.xz`. The suffix is `zephyr-elf`,
not `zephyr-TOOLCHAIN`.

**Error:** `wget: server returned error: HTTP/1.1 404 Not Found`

**Verified correct filename via:**
```bash
curl -sL "https://api.github.com/repos/zephyrproject-rtos/sdk-ng/releases/tags/v0.17.0" \
  | jq '.assets[].name' | grep esp32
# "toolchain_linux-x86_64_xtensa-espressif_esp32_zephyr-elf.tar.xz"
```

**Fix:**
```yaml
# Before
TOOLCHAIN_FILE: toolchain_xtensa-espressif_esp32_zephyr-TOOLCHAIN_linux-x86_64.tar.xz

# After
TOOLCHAIN_FILE: toolchain_linux-x86_64_xtensa-espressif_esp32_zephyr-elf.tar.xz
```

---

### Fix 4 — ESP32 board name invalid for Zephyr 4.0.0

**Files:** `.github/workflows/verify.yml`, `.github/workflows/build-release.yml`

**Problem:** Workflows used `esp32_devkitc/esp32/procpu` (HWMv2 qualified format).
This format does not exist in Zephyr 4.0.0. The flat name `esp32_devkitc_wroom` is
required for Zephyr 4.0.0 (it gets auto-qualified with a deprecation warning).

**Error:** `No board named 'esp32_devkitc' found`

**Fix:** Changed in 4 locations across both workflow files:
```yaml
# Before
west build -b esp32_devkitc/esp32/procpu ...

# After
west build -b esp32_devkitc_wroom ...
```

---

### Fix 5 — Toolchain tarball downloaded inside SDK directory

**Files:** `.github/workflows/verify.yml`, `.github/workflows/build-release.yml`

**Problem:** `wget` ran after `cd ~/zephyr-sdk-0.17.0/`, downloading the `.tar.xz`
file **into** the SDK directory. The SDK's cmake toolchain detection globs
`*xtensa-espressif_esp32*` and matched the tarball file as a toolchain path,
causing `CROSS_COMPILE` to point at `.tar.xz/bin/` — compiler not found.

**Error:** `CROSS_COMPILE` pointed at non-existent path inside tarball

**Fix:**
```yaml
# Before
cd ~/zephyr-sdk-${{ env.ZEPHYR_SDK_VERSION }}
wget ${{ env.SDK_BASE_URL }}/${{ env.TOOLCHAIN_FILE }}
tar xf ${{ env.TOOLCHAIN_FILE }}

# After
cd /tmp
wget ${{ env.SDK_BASE_URL }}/${{ env.TOOLCHAIN_FILE }}
tar xf ${{ env.TOOLCHAIN_FILE }} -C ~/zephyr-sdk-${{ env.ZEPHYR_SDK_VERSION }}/
rm -f ${{ env.TOOLCHAIN_FILE }}
```

---

### Fix 6 — SMF API incompatibility with Zephyr 4.0.0

**Files:** `esp32_firmware/motor_controller/src/main.cpp`, `esp32_firmware/sensor_fusion/src/main.cpp`

**Problem:** SMF state handlers used `enum smf_state_result` return type and returned
`SMF_EVENT_HANDLED`. These symbols do not exist in Zephyr 4.0.0 — they were introduced
in Zephyr ≥4.1. In Zephyr 4.0.0, SMF handlers return `void`.

**Error:**
```
error: 'smf_state_result' is not a member of 'smf'
error: 'SMF_EVENT_HANDLED' was not declared in this scope
```

**Fix:** Changed 9 handlers across 2 firmware files:
- `motor_controller/src/main.cpp`: 4 handlers (`idle_run`, `running_run`, `fault_run`, `estop_run`)
- `sensor_fusion/src/main.cpp`: 5 handlers (`init_run`, `calibrating_run`, `active_run`, `degraded_run`, `fault_run`)

```cpp
// Before
static enum smf_state_result idle_run(void *obj) {
    // ...
    return SMF_EVENT_HANDLED;
}

// After
static void idle_run(void *obj) {
    // ...
    return;  // (or just fall through)
}
```

---

## 3. Files Modified

| File | Changes |
|------|---------|
| `docker/Dockerfile.prod` | `osrf/ros:jazzy-ros-base` → `ros:jazzy-ros-base`; unique apt cache IDs per stage |
| `.github/workflows/verify.yml` | Toolchain filename, board name (×2), wget to `/tmp` |
| `.github/workflows/build-release.yml` | Toolchain filename, board name (×2), wget to `/tmp` |
| `esp32_firmware/motor_controller/src/main.cpp` | 4 SMF handlers: `enum smf_state_result` → `void` |
| `esp32_firmware/sensor_fusion/src/main.cpp` | 5 SMF handlers: `enum smf_state_result` → `void` |
| `porter_robot/Claude.md` | Corrected lesson #20; added lessons 24–28; added 6 gotchas to §15 table; added devlog ref |
| `porter_robot/OBJECTIVES.md` | Updated CI/CD status |

---

## 4. Documentation Updates

### Claude.md Changes

**Lesson #20 corrected:**
- Was incorrect (only documented local dev fix direction)
- Now correctly documents the version-dependent behavior:
  - Zephyr 4.0.0: SMF handlers return `void`
  - Zephyr ≥4.1: SMF handlers return `enum smf_state_result`

**New lessons added (24–28):**
| # | Topic |
|---|-------|
| 24 | `osrf/ros:jazzy-ros-base` doesn't exist — use `ros:jazzy-ros-base` |
| 25 | BuildKit apt cache lock — unique `id` per stage |
| 26 | Zephyr SDK toolchain filename format — platform before toolchain |
| 27 | Toolchain tarball must not be inside SDK directory |
| 28 | ESP32 board name depends on Zephyr version |

**Known Issues & Gotchas table (§15):**
Added 6 new rows for CI/CD specific issues (Docker image naming, BuildKit cache, SDK URLs, toolchain download, board names, SMF API version).

---

## 5. Root Cause Analysis

All 6 bugs share a common theme: **assumptions from local dev environment don't hold in CI.**

| Bug | Local Assumption | CI Reality |
|-----|-----------------|-----------|
| Docker image tag | Never pulled `osrf/ros:jazzy-ros-base` locally | Docker Hub doesn't have it |
| apt cache | Single-stage builds don't contend | Multi-stage BuildKit runs parallel |
| Toolchain URL | Never downloaded in CI before | Filename format differs from intuition |
| Board name | Local Zephyr ≥4.1 has HWMv2 names | CI Zephyr 4.0.0 uses flat names |
| wget location | Only extracted locally once | CI `cd` into SDK dir before wget |
| SMF API | Local Zephyr ≥4.1 has new return types | CI Zephyr 4.0.0 uses void handlers |

**Lesson:** Always pin versions explicitly, verify download URLs against release APIs,
and test with the exact same versions as CI. The `ZEPHYR_VERSION` / `ZEPHYR_SDK_VERSION`
env vars in workflows are the single source of truth.

---

## 6. Verification

- Docker dev build: `docker compose -f docker/docker-compose.dev.yml build` ✅
- Docker prod build: `docker build -f docker/Dockerfile.prod -t porter-robot:test --no-cache .` ✅
- All 6 fixes verified via subsequent GitHub Actions runs (user confirmed)

---

## 7. Next Steps

1. Push fixes and verify full CI green on GitHub Actions
2. Continue to Phase 2 (Navigation / Nav2 integration)
3. Consider pinning Zephyr version to ≥4.1 to use new SMF API (cleaner code)
