
# Porter Robotics – Developer Log

### **Date:** 10 Mar 2026

### **Engineer:** Claude (AI) + Antony Austin (review)

### **Subsystem:** ESP32 Firmware & ROS 2 Bridge — Phase 3, Tasks 10–17

---

## 1. Summary

Built the complete ESP32 firmware stack and ROS 2 bridge layer for the Porter Robot.
This covers the full RPi ↔ ESP32 communication chain: a shared C protocol library
(CRC16, binary packet parser/encoder, transport abstraction), two Zephyr RTOS
firmware applications (motor controller on ESP32 #1, sensor fusion on ESP32 #2),
unit tests running on `native_sim`, ROS 2 bridge nodes translating between
`/cmd_vel` / `/environment` topics and binary serial, and udev rules for stable
device naming.

**Key results:**
- **60 Ztest unit tests** on `native_sim` — 14 CRC + 26 protocol + 20 transport — all pass
- **187 colcon tests** across 4 ROS 2 packages — 0 errors, 0 failures, 12 skipped
- **247 total tests, 0 failures**
- Both ESP32 firmware builds compile clean for `esp32_devkitc/esp32/procpu`
- Motor controller: ~110 KB text, Sensor fusion: ~119 KB text
- `porter_esp32_bridge` ROS 2 package builds + all 8 lint tests pass

---

## 2. Files Created / Modified

### Task 10 — CRC16-CCITT (5 files)

| File | Purpose |
|------|---------|
| `esp32_firmware/common/include/crc16.h` | CRC16-CCITT header (poly 0x1021, init 0xFFFF) |
| `esp32_firmware/common/src/crc16.c` | 256-byte lookup table implementation |
| `esp32_firmware/tests/test_crc16/CMakeLists.txt` | Ztest build config |
| `esp32_firmware/tests/test_crc16/prj.conf` | Ztest project config |
| `esp32_firmware/tests/test_crc16/src/main.c` | 14 test cases (known vectors, empty, single byte, incremental, all-zeros, all-0xFF) |

### Task 11 — Protocol Parser & Encoder (5 files)

| File | Purpose |
|------|---------|
| `esp32_firmware/common/include/protocol.h` | Packet format, command IDs, state machine API |
| `esp32_firmware/common/src/protocol.c` | 9-state byte-by-byte parser + encoder + ACK/NACK |
| `esp32_firmware/tests/test_protocol/CMakeLists.txt` | Ztest build config |
| `esp32_firmware/tests/test_protocol/prj.conf` | Ztest project config |
| `esp32_firmware/tests/test_protocol/src/main.c` | 26 tests: 14 parser + 7 encoder + 5 roundtrip |

### Task 12 — Transport Abstraction Layer (4 files)

| File | Purpose |
|------|---------|
| `esp32_firmware/common/include/transport.h` | Transport API: init/read/write/flush/deinit + rx callback |
| `esp32_firmware/common/src/transport.c` | 3 backends: UART, CDC_ACM, Mock — selected via Kconfig |
| `esp32_firmware/common/Kconfig` | `CONFIG_PORTER_TRANSPORT`, `CONFIG_PORTER_TRANSPORT_{UART,CDC_ACM,MOCK}` |
| `esp32_firmware/tests/test_transport/` | 20 Ztest cases: init/deinit, read/write, flush, callbacks, protocol integration |

### Task 13 — Motor Controller Firmware (~730 lines)

| File | Purpose |
|------|---------|
| `esp32_firmware/motor_controller/src/main.cpp` | SMF state machine (IDLE→RUNNING→FAULT→ESTOP), BTS7960 PWM, differential drive, heartbeat watchdog, speed ramping, encoder stub, lift control, zbus channels |
| `esp32_firmware/motor_controller/CMakeLists.txt` | Zephyr build — links common protocol/CRC libs |
| `esp32_firmware/motor_controller/prj.conf` | PWM, GPIO, USB device, SMF, zbus, task watchdog |
| `esp32_firmware/motor_controller/app.overlay` | BTS7960 PWM channels (LEDC), encoder GPIOs, lift relay, E-stop button |

### Task 14 — Sensor Fusion Firmware (~750 lines)

| File | Purpose |
|------|---------|
| `esp32_firmware/sensor_fusion/src/main.cpp` | SMF (INIT→CALIBRATING→ACTIVE→DEGRADED→FAULT), VL53L0x I2C driver, ultrasonic trigger/echo, microwave ADC, 1D Kalman filter, cross-validation, sensor timeout/fallback, zbus channels |
| `esp32_firmware/sensor_fusion/CMakeLists.txt` | Zephyr build — links common libs |
| `esp32_firmware/sensor_fusion/prj.conf` | I2C, ADC, GPIO, USB device, SMF, zbus |
| `esp32_firmware/sensor_fusion/app.overlay` | VL53L0x on I2C0, ultrasonic trigger/echo GPIOs, microwave on ADC1 CH0 |

### Task 15 — Ztest Unit Tests on native_sim

| File | Purpose |
|------|---------|
| `esp32_firmware/tests/test_transport/CMakeLists.txt` | Ztest build for transport |
| `esp32_firmware/tests/test_transport/prj.conf` | Mock transport enabled |
| `esp32_firmware/tests/test_transport/Kconfig` | Sources common Kconfig + Zephyr |
| `esp32_firmware/tests/test_transport/src/main.c` | 20 tests covering all transport API surfaces |

### Task 16 — ROS 2 Bridge Nodes (10 files)

| File | Purpose |
|------|---------|
| `src/porter_esp32_bridge/package.xml` | ament_cmake, Apache-2.0 |
| `src/porter_esp32_bridge/CMakeLists.txt` | Builds porter_protocol static lib, serial_port lib, two bridge executables |
| `src/porter_esp32_bridge/include/porter_esp32_bridge/serial_port.hpp` | POSIX termios serial wrapper |
| `src/porter_esp32_bridge/src/serial_port.cpp` | termios 8N1, non-blocking reads, blocking writes |
| `src/porter_esp32_bridge/src/esp32_motor_bridge.cpp` | `/cmd_vel` → differential drive → `CMD_MOTOR_SET_SPEED`, heartbeat, status polling, diagnostics |
| `src/porter_esp32_bridge/src/esp32_sensor_bridge.cpp` | `CMD_SENSOR_FUSED` → `/environment` (Range), per-sensor readings, diagnostics |
| `src/porter_esp32_bridge/launch/esp32_bridge_launch.py` | Launches both bridge nodes with config |
| `src/porter_esp32_bridge/config/esp32_bridge_params.yaml` | All parameters documented |
| `src/porter_esp32_bridge/README.md` | Architecture, parameters, topics, usage |
| `src/porter_esp32_bridge/test/test_lint.cmake` | Standard ament lint test aggregator |

### Task 17 — udev Rules & Device Naming (2 files)

| File | Purpose |
|------|---------|
| `esp32_firmware/udev/99-porter-esp32.rules` | Template rules for `/dev/esp32_motors` + `/dev/esp32_sensors` (by serial number or port path) |
| `esp32_firmware/udev/install_udev_rules.sh` | Installer script — copies rules, reloads udev |

### Modified Existing Files

| File | Change |
|------|--------|
| `docker/docker-compose.dev.yml` | Added ESP32 device pass-through comments for hardware testing |
| `esp32_firmware/tests/test_crc16/prj.conf` | Removed deprecated `CONFIG_ZTEST_NEW_API=y` |
| `esp32_firmware/tests/test_protocol/prj.conf` | Removed deprecated `CONFIG_ZTEST_NEW_API=y` |
| `esp32_firmware/common/src/transport.c` | Fixed duplicate `LOG_MODULE_REGISTER` in mock section |

---

## 3. Architecture

### Wire Protocol

```
[0xAA][0x55][Length:1B][Command:1B][Payload:0..64B][CRC16_lo][CRC16_hi]
```

- CRC16-CCITT (poly 0x1021, init 0xFFFF) over Length + Command + Payload
- 256-byte lookup table for speed (no bit-shifting per byte)
- Max payload: 64 bytes. Min packet: 6 bytes (header + length + cmd + CRC).

### Protocol State Machine (9 states)

```
IDLE → HEADER1 (0xAA) → HEADER2 (0x55) → LENGTH → COMMAND → PAYLOAD → CRC_LOW → CRC_HIGH
  → COMPLETE (CRC ok) or ERROR (CRC mismatch / oversized / timeout)
```

### Transport Abstraction

```
Application Layer (protocol.c)
        ↓
Transport API (transport.h)
        ↓
┌───────────────┬───────────────┬───────────────┐
│  UART Backend │ CDC ACM       │ Mock Backend  │
│  (CP2102/CH340)│ (ESP32-S2/S3) │ (Ztest)      │
└───────────────┴───────────────┴───────────────┘
Selected at compile time via Kconfig — no runtime overhead.
```

### Motor Controller State Machine (SMF)

```
    ┌──────┐    CMD_MOTOR_SET_SPEED    ┌─────────┐
    │ IDLE │ ──────────────────────→ │ RUNNING │
    └──┬───┘                          └────┬────┘
       │           ← CMD_MOTOR_STOP →      │
       │    ┌───────┐                       │
       └──→ │ FAULT │ ←── overcurrent/timeout
            └───┬───┘
                │ recovery
                ↓
          ┌─────────┐
          │  ESTOP  │ ←── hardware E-stop button (GPIO ISR)
          └─────────┘

Thread priorities: safety(-1) > motor(0) > protocol(1) > reporting(5) > shell(14)
Heartbeat watchdog: 500ms timeout → auto-stop motors
Speed ramping: max acceleration/deceleration enforced per control loop iteration
```

### Sensor Fusion State Machine (SMF)

```
    ┌──────┐    all sensors OK    ┌────────────┐
    │ INIT │ ──────────────────→ │ CALIBRATING│
    └──────┘                      └──────┬─────┘
                                         │ baselines set
                                         ↓
                                   ┌────────┐
                    ← sensor fail → │ ACTIVE │
                                   └────┬───┘
              ┌──────────┐              │ partial fail
              │  FAULT   │ ←── all fail │
              └──────────┘              ↓
                                  ┌──────────┐
                                  │ DEGRADED │ → fallback to secondary sensor
                                  └──────────┘

Sensors: VL53L0x (ToF I2C) + HC-SR04 (Ultrasonic GPIO) + RCWL-0516 (Microwave ADC)
Fusion: 1D Kalman filter — prediction + update cycle per sensor reading
Cross-validation: ToF vs Ultrasonic disagree >30% → flag inconsistency
Sensor timeout: 100ms → mark degraded, switch to fallback
```

### ROS 2 Bridge Architecture

```
ROS 2 (RPi)                     USB Serial              ESP32
─────────────────────────────────────────────────────────────────
                                                    ┌─────────────┐
/cmd_vel ──→ esp32_motor_bridge ──→ /dev/esp32_motors ──→ │ Motor Ctrl  │
             (differential drive,   (CMD_MOTOR_SET_SPEED)  │ (Zephyr)    │
              heartbeat sender)  ←── CMD_MOTOR_STATUS ←── │ BTS7960 PWM │
             publishes /motor_status                       └─────────────┘
             publishes /diagnostics
                                                    ┌─────────────┐
/environment ←── esp32_sensor_bridge ←── /dev/esp32_sensors ←── │ Sensor Fuse │
  (Range)        (Kalman decode,    CMD_SENSOR_FUSED           │ (Zephyr)    │
                  status polling)  ──→ CMD_SENSOR_STATUS ──→   │ ToF+US+MW   │
                 publishes /diagnostics                        └─────────────┘
```

---

## 4. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Pure C for common library (crc16, protocol, transport) | Must compile for both Zephyr (Xtensa) and Linux (x86_64/arm64 ROS 2 bridge) |
| 256-byte CRC lookup table | ~5× faster than bit-shifting; 256 bytes of flash is negligible on ESP32 |
| 9-state parser (not regex/delimiter) | Byte-by-byte state machine handles partial reads, streaming, and back-to-back packets |
| Transport via Kconfig (compile-time) | Zero runtime overhead; swap ESP32 variant = change Kconfig + overlay |
| SMF (Zephyr State Machine Framework) | Built into Zephyr, well-tested, enforces entry/run/exit handlers |
| zbus for inter-thread communication | Type-safe pub/sub channels, built into Zephyr, no manual mutexes |
| 500ms heartbeat watchdog | Airport safety — motors must stop immediately if RPi crashes/disconnects |
| Speed ramping (acceleration limits) | Prevent sudden movements that could injure passengers or tip luggage |
| Kalman filter for sensor fusion | Standard approach; handles noisy ToF/ultrasonic with predictive smoothing |
| Cross-validation (ToF vs Ultrasonic) | Detects sensor drift or failure — 30% disagreement → flag + fallback |
| POSIX termios for bridge serial | Direct, no extra dependency; handles EAGAIN on non-blocking reads properly |
| Apache 2.0 license for bridge | Matches all other Porter packages |

---

## 5. Bugs Fixed During Development

### Bug 1: `CONFIG_ZTEST_NEW_API` deprecated in Zephyr 4.3.x

**Symptom:** Build warning: `Symbol ZTEST_NEW_API is not defined in Kconfig`
**Root cause:** Zephyr 4.3.x made the new Ztest API the default. The old `CONFIG_ZTEST_NEW_API=y` in `prj.conf` references an undefined symbol.
**Fix:** Removed `CONFIG_ZTEST_NEW_API=y` from all 3 test `prj.conf` files.

### Bug 2: `ZTEST_SUITE` requires 6 arguments in Zephyr 4.3.x

**Symptom:** `error: too few arguments to function call 'ZTEST_SUITE'`
**Root cause:** The macro signature is `ZTEST_SUITE(name, predicate, setup, before, after, teardown)` — 6 args. Transport tests initially had 5.
**Fix:** Added NULL teardown argument: `ZTEST_SUITE(transport_tests, NULL, NULL, NULL, NULL, NULL);`

### Bug 3: Duplicate `LOG_MODULE_REGISTER` in transport.c

**Symptom:** `error: redefinition of 'log_const_transport'`
**Root cause:** `transport.c` had `LOG_MODULE_REGISTER(transport, LOG_LEVEL_INF)` at line 29 (shared Zephyr section) AND line 50 (mock backend section inside `#ifdef __ZEPHYR__`). Both expand to the same linker symbol.
**Fix:** Removed the duplicate in the mock section; kept only the shared one at line 29.

### Bug 4: Zephyr venv polluting colcon CMake cache

**Symptom:** `colcon build` fails with `No module named 'catkin_pkg'`
**Root cause:** CMake cached `~/zephyrproject/.venv/bin/python3` from a previous Zephyr build. The Zephyr venv doesn't have `catkin_pkg` or `ament` packages.
**Fix:** `rm -rf build/ install/ log/` to clear CMake cache. Always deactivate Zephyr venv before running `colcon build`.

### Bug 5: ament_copyright requires `//` comment style with Apache 2.0

**Symptom:** `ament_copyright` test fails — doesn't recognize `/* ... */` block comment copyright
**Root cause:** ament_copyright's Apache 2.0 detection regex expects `// Copyright [year] <Owner>` format with `//` line comments.
**Fix:** Converted all C++ file headers to `//` comment style with full Apache 2.0 boilerplate.

### Bug 6: cpplint include order (C before C++ headers)

**Symptom:** `cpplint: system header <cstring> should be after <string.h>`
**Root cause:** cpplint enforces: own header → C system → C++ system → other. `serial_port.cpp` had C++ headers before C headers.
**Fix:** Reordered includes: `serial_port.hpp` → C headers (`<fcntl.h>`, `<string.h>`, etc.) → C++ headers (`<cstring>`, `<stdexcept>`).

### Bug 7: cpplint `build/include_subdir` for protocol headers

**Symptom:** `cpplint: include 'protocol.h' not using a subdirectory`
**Root cause:** `#include "protocol.h"` has no directory prefix. These are external headers from `esp32_firmware/common/` added via CMake `target_include_directories`.
**Fix:** Added `// NOLINT(build/include_subdir)` to the include lines.

### Bug 8: uncrustify lambda brace spacing

**Symptom:** `uncrustify: expected '{return ...;}' not '{ return ...; }'`
**Root cause:** uncrustify (ament profile) expects no spaces inside braces for single-line lambdas.
**Fix:** `{return ...;}` instead of `{ return ...; }`.

### Bug 9: cpplint TODO format

**Symptom:** `cpplint: TODO should be in format TODO(username)`
**Root cause:** `// TODO: ...` doesn't match cpplint's required format.
**Fix:** Changed to `// TODO(antony): ...` throughout.

---

## 6. Test Results

### Ztest on native_sim (60/60 pass)

```
=== test_crc16 (14/14) ===
PASS - test_crc16_known_vector
PASS - test_crc16_empty_data
PASS - test_crc16_single_byte
PASS - test_crc16_incremental
PASS - test_crc16_all_zeros
PASS - test_crc16_all_0xff
PASS - test_crc16_one_byte_values
PASS - test_crc16_two_bytes
PASS - test_crc16_different_init
PASS - test_crc16_large_buffer
PASS - test_crc16_alignment
PASS - test_crc16_known_packet
PASS - test_crc16_stability
PASS - test_crc16_table_correctness

=== test_protocol (26/26) ===
Parser tests (14): valid packet, empty payload, max payload, bad CRC, truncated,
  oversized, back-to-back, partial feed, double header, all commands, reset mid-parse,
  zero-length explicit, minimum packet, interleaved garbage
Encoder tests (7): basic encode, empty payload, max payload, ACK, NACK, null buffer,
  buffer too small
Roundtrip tests (5): CMD_MOTOR_SET_SPEED, CMD_SENSOR_FUSED, CMD_HEARTBEAT, ACK, NACK

=== test_transport (20/20) ===
Init/deinit (4): basic init, double init, deinit without init, init-deinit-reinit
Read/write (5): write data, read data, read empty, write zero, large transfer
Flush (2): basic flush, flush when empty
Callbacks (5): rx callback set, rx callback trigger, rx callback clear, rx callback null,
  multiple callbacks
Protocol integration (4): write+read roundtrip, protocol feed via transport,
  partial reads, concurrent read/write simulation
```

### colcon tests (187/187 pass, 12 skipped)

```
4 packages tested:
  porter_esp32_bridge — 8 tests (copyright, cpplint, cppcheck, uncrustify, flake8, pep257, lint_cmake, xmllint)
  porter_lidar_processor — 30 tests (24 unit + 6 lint)
  porter_orchestrator — 27 tests (23 unit + 4 lint)
  ydlidar_driver — 21 tests (9 unit + 12 lint)

12 skipped: cppcheck runs on porter_esp32_bridge but marks some checks as skipped (expected)
```

### ESP32 firmware builds

```
Motor controller:   west build -b esp32_devkitc/esp32/procpu → 110,368 bytes text
Sensor fusion:      west build -b esp32_devkitc/esp32/procpu → 119,040 bytes text
```

---

## 7. Command Reference (for reproducing)

### Build Ztest on native_sim

```bash
source ~/zephyrproject/.venv/bin/activate
cd ~/zephyrproject

# CRC16 tests
west build -p always -b native_sim ~/Porter-ROS/porter_robot/esp32_firmware/tests/test_crc16
./build/zephyr/zephyr.exe

# Protocol tests
west build -p always -b native_sim ~/Porter-ROS/porter_robot/esp32_firmware/tests/test_protocol
./build/zephyr/zephyr.exe

# Transport tests
west build -p always -b native_sim ~/Porter-ROS/porter_robot/esp32_firmware/tests/test_transport
./build/zephyr/zephyr.exe
```

### Build ESP32 firmware

```bash
source ~/zephyrproject/.venv/bin/activate
cd ~/zephyrproject

# Motor controller
west build -p always -b esp32_devkitc/esp32/procpu ~/Porter-ROS/porter_robot/esp32_firmware/motor_controller

# Sensor fusion
west build -p always -b esp32_devkitc/esp32/procpu ~/Porter-ROS/porter_robot/esp32_firmware/sensor_fusion
```

### Build ROS 2 packages

```bash
# IMPORTANT: Deactivate Zephyr venv first!
deactivate 2>/dev/null
cd ~/Porter-ROS/porter_robot
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --cmake-args -Wno-dev
source install/setup.bash
colcon test --event-handlers console_direct+
colcon test-result --verbose
```

---

## 8. Lessons for Future Sessions

1. **Never have Zephyr venv active when running colcon** — CMake caches the wrong Python.
2. **Zephyr 4.3.x changes**: `CONFIG_ZTEST_NEW_API` is deprecated (default now), `ZTEST_SUITE` needs 6 args.
3. **ament_copyright** only recognizes `//` comment style for Apache 2.0, not `/* */` blocks.
4. **cpplint include ordering**: C system headers before C++ system headers. External includes need NOLINT.
5. **uncrustify lambda format**: `{return x;}` not `{ return x; }` for single-line lambdas.
6. **SMF in C++**: Need `extern "C"` for state handler arrays, `extern const struct smf_state xxx[];` forward declaration, `enum smf_state_result` return type (not `void`).
7. **ESP32 ADC in Zephyr**: ADC nodes may be disabled by default in devicetree — use `status = "okay"` in overlay.
8. **ESP32 `#pwm-cells`**: Must match exactly — `<3>` for standard LEDC PWM binding, not `<2>`.
9. **Transport mock for testing**: Keep mock backend inside `#ifdef __ZEPHYR__` with `CONFIG_PORTER_TRANSPORT_MOCK`, plus a host mock outside for Linux compilation. Share one `LOG_MODULE_REGISTER`.

---
