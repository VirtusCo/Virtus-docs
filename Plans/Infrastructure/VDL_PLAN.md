# Virtus Message Definition Library (VDL)
### Detailed Development Plan
**Project:** Single source of truth for all custom ROS 2 message, service, and action types across the entire Virtusco stack  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026  
**Build Priority:** #1 — Build before Nav2, Fleet Monitor, or any new ROS 2 package

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [Why This Must Be Built First](#2-why-this-must-be-built-first)
3. [Repository Structure](#3-repository-structure)
4. [Message Definitions](#4-message-definitions)
5. [Service Definitions](#5-service-definitions)
6. [Action Definitions](#6-action-definitions)
7. [Versioning Strategy](#7-versioning-strategy)
8. [Cross-Stack Usage Map](#8-cross-stack-usage-map)
9. [Build & Integration](#9-build--integration)
10. [Documentation Standards](#10-documentation-standards)
11. [CI/CD Pipeline](#11-cicd-pipeline)
12. [Migration from Current State](#12-migration-from-current-state)
13. [Phased Build Plan](#13-phased-build-plan)
14. [Tech Stack Summary](#14-tech-stack-summary)
15. [Risk Register](#15-risk-register)

---

## 1. Overview & Vision

### What VDL Is

A standalone ROS 2 package — `virtus_msgs` — that defines every custom message, service, and action type used anywhere in the Virtusco stack. It is the **contract layer** between all components.

Every package that needs a custom Virtusco type declares a dependency on `virtus_msgs` in its `package.xml`. No package ever defines its own custom Virtus types inline.

### What VDL Is Not

- Not a runtime library — it generates no executable code
- Not a configuration file — it does not store parameters
- Not specific to any platform — it compiles on Ubuntu/Linux, Windows (WSL2), and macOS

### The Problem It Solves

Right now, custom message types are either:
- Scattered across individual packages (each package defines what it needs)
- Implicitly agreed on via convention (everyone just knows the `SensorFusion` struct layout)
- Duplicated in the Flutter GUI as Dart classes maintained separately from the ROS definitions

When the sensor fusion struct gains a new field (e.g., `obstacle_class`), you currently need to update:
- The Zephyr firmware struct in `esp32_firmware/common/protocol.h`
- The ROS 2 bridge decoder in `porter_esp32_bridge`
- The Flutter GUI Dart model class
- Any Python scripts that parse the topic
- The Hardware Dashboard extension's telemetry schema
- The ROS 2 Studio extension's topic registry

With VDL, you update the `.msg` file once. Everything that depends on it recompiles. Anything that doesn't recompile and match the new schema fails loudly at build time — not silently at runtime.

---

## 2. Why This Must Be Built First

### The Cost of Waiting

Every new package you add before VDL exists creates technical debt:
- `porter_nav2_integration` will need `OrchestratorState` — it'll either import it from `porter_orchestrator` (wrong) or define it again (worse)
- Fleet Monitor backend will need `RobotStatus` — without VDL it will be hand-written JSON with no schema enforcement
- AI Studio will produce model outputs that need to become ROS 2 messages — without VDL there's no canonical type to target

The longer you wait, the more packages need to be refactored when VDL arrives. Since this is a 1–2 day task, there is no justification for waiting.

### Dependency Graph

```
virtus_msgs  (VDL)
    ├── porter_esp32_bridge      depends on: BridgeFrame, SensorFusion
    ├── porter_orchestrator      depends on: OrchestratorState, PassengerCommand
    ├── porter_lidar_processor   depends on: (uses std ROS types — no VDL dep needed)
    ├── porter_ai_assistant      depends on: PassengerCommand, AIResponse
    ├── porter_nav2_integration  depends on: OrchestratorState, NavigationGoal (future)
    ├── Flutter GUI              depends on: all (via rosbridge JSON serialization)
    ├── Hardware Dashboard ext   depends on: SensorFusion, RobotStatus, BridgeFrame
    ├── ROS 2 Studio ext         depends on: all (for topic registry)
    └── Fleet Monitor backend    depends on: RobotStatus, FleetHeartbeat, IncidentReport
```

---

## 3. Repository Structure

VDL lives as a standalone ROS 2 package inside the Porter-ROS monorepo:

```
Porter-ROS/
└── src/
    └── virtus_msgs/                    ← The VDL package
        ├── package.xml                 ← ROS 2 package manifest
        ├── CMakeLists.txt              ← rosidl_generate_interfaces()
        ├── README.md                   ← Usage guide + field documentation
        ├── CHANGELOG.md                ← Version history + breaking changes
        │
        ├── msg/                        ← Message definitions
        │   ├── SensorFusion.msg
        │   ├── OrchestratorState.msg
        │   ├── BridgeFrame.msg
        │   ├── BridgeFrameRaw.msg
        │   ├── RobotStatus.msg
        │   ├── PassengerCommand.msg
        │   ├── AIResponse.msg
        │   ├── PowerTelemetry.msg
        │   ├── MotorTelemetry.msg
        │   ├── FleetHeartbeat.msg
        │   └── IncidentEvent.msg
        │
        ├── srv/                        ← Service definitions
        │   ├── GetFlightInfo.srv
        │   ├── NavigateTo.srv
        │   ├── ManualOverride.srv
        │   ├── GetRobotStatus.srv
        │   └── UpdateConfig.srv
        │
        └── action/                     ← Action definitions
            └── EscortPassenger.action
```

### package.xml

```xml
<?xml version="1.0"?>
<package format="3">
  <name>virtus_msgs</name>
  <version>1.0.0</version>
  <description>
    Virtusco custom ROS 2 message, service, and action definitions.
    Single source of truth for all Virtus stack interfaces.
    See CHANGELOG.md before making breaking changes.
  </description>
  <maintainer email="virtusco.tech@gmail.com">Antony Austin</maintainer>
  <license>MIT</license>

  <buildtool_depend>ament_cmake</buildtool_depend>
  <buildtool_depend>rosidl_default_generators</buildtool_depend>

  <depend>std_msgs</depend>
  <depend>geometry_msgs</depend>
  <depend>builtin_interfaces</depend>

  <exec_depend>rosidl_default_runtime</exec_depend>
  <member_of_group>rosidl_interface_packages</member_of_group>
</package>
```

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.8)
project(virtus_msgs)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(builtin_interfaces REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  # Messages
  "msg/SensorFusion.msg"
  "msg/OrchestratorState.msg"
  "msg/BridgeFrame.msg"
  "msg/BridgeFrameRaw.msg"
  "msg/RobotStatus.msg"
  "msg/PassengerCommand.msg"
  "msg/AIResponse.msg"
  "msg/PowerTelemetry.msg"
  "msg/MotorTelemetry.msg"
  "msg/FleetHeartbeat.msg"
  "msg/IncidentEvent.msg"
  # Services
  "srv/GetFlightInfo.srv"
  "srv/NavigateTo.srv"
  "srv/ManualOverride.srv"
  "srv/GetRobotStatus.srv"
  "srv/UpdateConfig.srv"
  # Actions
  "action/EscortPassenger.action"
  DEPENDENCIES std_msgs geometry_msgs builtin_interfaces
)

ament_package()
```

---

## 4. Message Definitions

### SensorFusion.msg
Published by ESP32 #2 bridge on `/sensor_fusion` at 50Hz.

```
# virtus_msgs/SensorFusion
# ESP32 #2 sensor fusion output — Kalman-filtered obstacle estimate
# Published at 50Hz on /sensor_fusion
# Source: porter_esp32_bridge, decoded from BridgeFrame type 0x02

builtin_interfaces/Time stamp
string frame_id                  # Always "base_link"

# Raw sensor readings (before fusion)
uint16 tof_mm                    # VL53L0X Time-of-Flight distance (mm). 0 = no reading.
uint16 ultrasonic_cm             # HC-SR04 ultrasonic distance (cm). 0 = no reading.
bool   microwave_detected        # RCWL-0516 motion presence (true = motion detected)

# Kalman filter output
float32 kalman_estimate_cm       # Fused distance estimate (cm)
float32 kalman_variance          # Estimate uncertainty — low = confident, high = uncertain
                                 # Variance > 10.0 indicates sensor disagreement

# Obstacle classification
uint8 obstacle_class             # 0=CLEAR, 1=NEAR(<50cm), 2=MEDIUM(50-150cm), 3=FAR(>150cm)
uint8 OBSTACLE_CLEAR   = 0
uint8 OBSTACLE_NEAR    = 1
uint8 OBSTACLE_MEDIUM  = 2
uint8 OBSTACLE_FAR     = 3

# Health flags (bitmask)
uint8 health_flags               # Bit 0: ToF OK, Bit 1: Ultrasonic OK, Bit 2: Microwave OK
                                 # All 3 bits set = all sensors healthy
```

### OrchestratorState.msg
Published by `porter_orchestrator` on `/orchestrator/state` at 5Hz.

```
# virtus_msgs/OrchestratorState
# Porter orchestrator FSM state — published at 5Hz on /orchestrator/state

builtin_interfaces/Time stamp

# Current FSM state
uint8 state
uint8 STATE_BOOT                = 0
uint8 STATE_HEALTH_CHECK        = 1
uint8 STATE_IDLE                = 2
uint8 STATE_PASSENGER_DETECTED  = 3
uint8 STATE_FOLLOWING           = 4
uint8 STATE_NAVIGATING          = 5
uint8 STATE_OBSTACLE_AVOIDANCE  = 6
uint8 STATE_ERROR               = 7
uint8 STATE_RECOVERY            = 8

# Transition info
string last_trigger              # Event that caused last transition e.g. "passenger_detected"
builtin_interfaces/Time state_entered_at  # When current state was entered
uint32 state_duration_ms         # How long in current state (ms)

# Active task description (human-readable, for logging and Fleet Monitor)
string current_task              # e.g. "Escorting passenger to Gate C5" or ""

# Recovery info (populated when state = STATE_RECOVERY)
uint8 recovery_attempt           # 0 if not in recovery
uint8 recovery_max_attempts      # Configured max (default: 3)

# Error info (populated when state = STATE_ERROR)
string error_code                # Machine-readable error code e.g. "STUCK_TIMEOUT"
string error_detail              # Human-readable detail
```

### BridgeFrame.msg
Published by `porter_esp32_bridge` on `/esp32_bridge/rx` and `/esp32_bridge/tx`.

```
# virtus_msgs/BridgeFrame
# Decoded ESP32 bridge protocol frame
# Raw binary: [0xAA][MSG_TYPE:1B][LENGTH:2B LE][PAYLOAD:NB][CRC16:2B LE]

builtin_interfaces/Time stamp
uint8  source                    # 0=ESP32_1 (motor), 1=ESP32_2 (sensors)
uint8  msg_type                  # Frame type constants below
bool   crc_valid                 # False if CRC16-CCITT check failed

# Frame type constants
uint8 TYPE_MOTOR_CMD     = 1     # RPi → ESP32 #1
uint8 TYPE_SENSOR_DATA   = 2     # ESP32 #2 → RPi
uint8 TYPE_MOTOR_STATUS  = 3     # ESP32 #1 → RPi
uint8 TYPE_HEARTBEAT     = 4     # Either direction
uint8 TYPE_ERROR_FRAME   = 255   # Error report

# Payload (decoded per msg_type — only relevant fields populated)
# TYPE_MOTOR_CMD
int16  motor_left_speed          # RPWM duty % (-100 to +100, negative = reverse)
int16  motor_right_speed
uint8  lift_cmd                  # 0=STOP, 1=UP, 2=DOWN

# TYPE_SENSOR_DATA (see SensorFusion.msg for field definitions)
uint16 tof_mm
uint16 ultrasonic_cm
bool   microwave_detected
float32 kalman_cm

# TYPE_MOTOR_STATUS
uint16 left_current_ma
uint16 right_current_ma
bool   left_en
bool   right_en

# TYPE_HEARTBEAT
uint32 uptime_ms
uint8  error_flags

# TYPE_ERROR_FRAME
uint8  error_code
uint16 error_context
```

### RobotStatus.msg
Full robot health snapshot — used by Fleet Monitor and Hardware Dashboard.

```
# virtus_msgs/RobotStatus
# Complete robot status snapshot — published at 1Hz on /robot/status

builtin_interfaces/Time stamp
string robot_id                  # e.g. "virtus-cial-001"
string robot_name                # e.g. "CIAL Porter #1"

# Position
geometry_msgs/Pose2D pose        # x, y in meters (map frame), theta in radians
string current_zone              # Semantic zone name e.g. "Gate B"

# State (mirrors OrchestratorState for convenience)
uint8  fsm_state
string current_task

# Hardware health
float32 battery_voltage          # Volts
uint8   battery_pct              # 0-100
float32 rail_12v                 # Volts
float32 rail_5v
float32 rail_33v
uint16  motor_left_ma            # milliamps
uint16  motor_right_ma

# Compute health
uint8   cpu_pct
uint8   ram_pct
float32 cpu_temp_c
float32 disk_free_gb

# Software versions
string ros_pkg_version           # semver e.g. "0.4.2"
string firmware_version          # semver e.g. "1.2.0"
string ai_model_version          # e.g. "qwen2.5-1.5b-virtus-v4"

# Session metrics (today since midnight)
uint32 tasks_completed
float32 distance_m
uint16 errors_today
uint32 uptime_today_s
```

### PassengerCommand.msg
LLM-parsed passenger intent — output of `porter_ai_assistant`.

```
# virtus_msgs/PassengerCommand
# Structured command parsed from passenger speech/text by Virtue AI
# Published on /ai_assistant/command

builtin_interfaces/Time stamp
string session_id                # Passenger session identifier

# Intent classification
uint8  intent
uint8  INTENT_NAVIGATE     = 1   # Go to a location
uint8  INTENT_FOLLOW       = 2   # Follow the passenger
uint8  INTENT_STOP         = 3   # Stop movement
uint8  INTENT_WAIT         = 4   # Wait at current location
uint8  INTENT_INFO_QUERY   = 5   # Passenger asked a question (no movement)
uint8  INTENT_WEIGH        = 6   # Weigh luggage
uint8  INTENT_ASSIST       = 7   # Call human assistance

# Navigation target (populated for INTENT_NAVIGATE)
string destination_id            # Semantic ID e.g. "gate_c5", "checkin_zone_b"
string destination_label         # Human-readable e.g. "Gate C5"

# Confidence
float32 confidence               # 0.0-1.0 — how certain the AI is about this intent

# Original utterance (for logging)
string raw_text
string language                  # ISO 639-1: "en", "ml", "hi", "ta"
```

### AIResponse.msg
Virtue AI text response — for display in Flutter GUI.

```
# virtus_msgs/AIResponse
# Virtue AI text response to passenger
# Published on /ai_assistant/response

builtin_interfaces/Time stamp
string session_id
string text                      # Response text in passenger's language
string language                  # ISO 639-1 language code
float32 inference_time_ms        # Time to generate response
string model_version             # Which model/adapter produced this
bool   used_tool                 # True if a ROS 2 tool was called
string tool_called               # Tool name if used_tool is true
```

### PowerTelemetry.msg
Power rail telemetry from the power management board.

```
# virtus_msgs/PowerTelemetry
# Power management board telemetry — published at 10Hz on /hardware/power

builtin_interfaces/Time stamp
float32 v12                      # 12V rail voltage
float32 v5                       # 5V rail voltage
float32 v33                      # 3.3V rail voltage
uint16  i12_ma                   # 12V rail current (mA)
uint16  i5_ma                    # 5V rail current (mA)
bool[4] relay_states             # K1-K4 relay states (true = energized)
```

### MotorTelemetry.msg
BTS7960 motor driver telemetry.

```
# virtus_msgs/MotorTelemetry
# Motor driver telemetry — published at 10Hz on /hardware/motors

builtin_interfaces/Time stamp
# Left motor (BTS7960 #1)
uint8   left_rpwm_pct            # RPWM duty cycle 0-100%
uint8   left_lpwm_pct            # LPWM duty cycle 0-100%
bool    left_en                  # Enable pin state
uint16  left_current_ma
float32 left_temp_est_c          # Estimated die temperature

# Right motor (BTS7960 #2)
uint8   right_rpwm_pct
uint8   right_lpwm_pct
bool    right_en
uint16  right_current_ma
float32 right_temp_est_c
```

### FleetHeartbeat.msg
Heartbeat sent from robot to fleet backend every 30s.

```
# virtus_msgs/FleetHeartbeat
# Robot → fleet backend heartbeat
# Sent every 30s to confirm robot is alive and report summary status

builtin_interfaces/Time stamp
string robot_id
uint8  fsm_state                 # Current FSM state (mirrors OrchestratorState constants)
uint8  battery_pct
float32 pos_x                    # Map position
float32 pos_y
bool   is_serving                # True if actively serving a passenger
uint32 tasks_today               # Tasks completed today
```

### IncidentEvent.msg
Single event in the 60-second pre-incident rolling buffer.

```
# virtus_msgs/IncidentEvent
# Single event in incident recording buffer
# Published on /incident/events — consumed by incident_logger

builtin_interfaces/Time stamp
uint8  severity                  # 0=DEBUG, 1=INFO, 2=WARN, 3=ERROR, 4=FATAL
string source_node               # ROS 2 node name that generated this event
string event_type                # Machine-readable type e.g. "FSM_TRANSITION", "SENSOR_FAULT"
string description               # Human-readable description
string payload_json              # Optional JSON with structured event data
```

---

## 5. Service Definitions

### GetFlightInfo.srv
```
# Request
string flight_number             # e.g. "AI 6E 421"
string airport_iata              # e.g. "COK" (Kochi)
---
# Response
bool   found
string airline
string origin
string destination
string gate                      # e.g. "C5"
string terminal
string status                    # "ON_TIME", "DELAYED_30", "BOARDING", "DEPARTED"
int32  delay_minutes             # 0 if on time
string baggage_belt              # e.g. "Belt 3"
```

### NavigateTo.srv
```
# Request
string destination_id            # Semantic ID from airport map
string session_id                # Passenger session (for logging)
bool   escort_mode               # True = wait for passenger, False = go alone
---
# Response
bool   accepted
string reason                    # If not accepted: why
float32 estimated_time_s        # ETA in seconds
```

### ManualOverride.srv
```
# Request
uint8  command
uint8  CMD_STOP          = 1
uint8  CMD_RESUME        = 2
uint8  CMD_RETURN_BASE   = 3
uint8  CMD_ESTOP         = 4
string operator_id               # Who issued the override (for audit log)
string reason
---
# Response
bool   accepted
uint8  previous_state
uint8  new_state
```

### GetRobotStatus.srv
```
# Request
string robot_id                  # "" = this robot
---
# Response
virtus_msgs/RobotStatus status
bool   found
```

### UpdateConfig.srv
```
# Request
string config_key
string config_value_json         # JSON-serialized new value
string updated_by                # Who requested the change
---
# Response
bool   accepted
string previous_value_json
string validation_error          # Empty if accepted
```

---

## 6. Action Definitions

### EscortPassenger.action
Long-running action for full passenger escort task.

```
# Goal
string passenger_id
string destination_id
string session_id
bool   carry_luggage             # True if luggage is loaded on robot
---
# Result
bool   success
string failure_reason            # Empty if success
float32 total_distance_m
float32 total_time_s
string final_state               # FSM state at completion
---
# Feedback (published during escort)
float32 progress_pct             # 0.0-100.0
string  current_phase            # "LOADING", "NAVIGATING", "ARRIVED", "WAITING_PASSENGER"
float32 distance_remaining_m
float32 eta_s
geometry_msgs/Pose2D current_pose
```

---

## 7. Versioning Strategy

### Semantic Versioning

`virtus_msgs` follows semver strictly:

| Change | Version bump | Action required |
|---|---|---|
| Add new field to existing msg | Minor | All consumers recompile |
| Add new msg/srv/action | Minor | No existing consumers affected |
| Remove field from existing msg | **Major** | All consumers must update |
| Rename field | **Major** | All consumers must update |
| Change field type | **Major** | All consumers must update |
| Add constant to existing msg | Minor | Safe |

### Breaking Change Process

Major version bumps must follow this process:
1. Add new field alongside old field (both exist) in a Minor release
2. Mark old field as `# DEPRECATED since v1.x — use new_field_name instead`
3. Give all consumers one sprint to migrate
4. Remove old field in the next Major release

**No breaking changes are ever pushed without this process.** The `CHANGELOG.md` documents every change.

### Version Pinning in Downstream Packages

```xml
<!-- In porter_orchestrator/package.xml -->
<depend version_gte="1.0.0" version_lt="2.0.0">virtus_msgs</depend>
```

This ensures a Major version bump fails CI immediately, forcing a conscious migration.

---

## 8. Cross-Stack Usage Map

| Message | Publisher | Subscribers | Frequency |
|---|---|---|---|
| `SensorFusion` | `porter_esp32_bridge` | `porter_orchestrator`, `Hardware Dashboard`, `ROS 2 Studio` | 50Hz |
| `OrchestratorState` | `porter_orchestrator` | `porter_esp32_bridge`, `porter_ai_assistant`, `Fleet Monitor`, `ROS 2 Studio` | 5Hz |
| `BridgeFrame` | `porter_esp32_bridge` | `ROS 2 Studio Bridge Debugger` | 50Hz |
| `RobotStatus` | `porter_telemetry` (new) | `Fleet Monitor`, `Hardware Dashboard`, `Suite Dashboard` | 1Hz |
| `PassengerCommand` | `porter_ai_assistant` | `porter_orchestrator`, `porter_nav2_integration` | On-demand |
| `AIResponse` | `porter_ai_assistant` | `Flutter GUI` (via rosbridge) | On-demand |
| `PowerTelemetry` | `porter_telemetry` (new) | `Hardware Dashboard`, `Fleet Monitor` | 10Hz |
| `MotorTelemetry` | `porter_telemetry` (new) | `Hardware Dashboard` | 10Hz |
| `FleetHeartbeat` | `porter_telemetry` (new) | `Fleet Monitor backend` | 0.033Hz (30s) |
| `IncidentEvent` | All nodes | `incident_logger` | On-demand |

---

## 9. Build & Integration

### Adding virtus_msgs as a Dependency

Any ROS 2 package that uses VDL types adds this to its `package.xml`:

```xml
<depend>virtus_msgs</depend>
```

And in Python code:

```python
from virtus_msgs.msg import SensorFusion, OrchestratorState
from virtus_msgs.srv import NavigateTo
from virtus_msgs.action import EscortPassenger
```

And in C++:

```cpp
#include "virtus_msgs/msg/sensor_fusion.hpp"
#include "virtus_msgs/srv/navigate_to.hpp"
```

### Flutter GUI Integration

The Flutter GUI uses rosbridge JSON serialization. VDL types automatically serialize to JSON when published via rosbridge. The Dart model classes are generated from the `.msg` files using a code generation script:

```bash
# scripts/generate_dart_models.py
# Reads all .msg files → generates lib/models/virtus_msgs/*.dart
python3 scripts/generate_dart_models.py src/virtus_msgs/msg/ porter_gui/lib/models/
```

This ensures the Dart models and ROS 2 messages are always in sync.

---

## 10. Documentation Standards

Every field in every `.msg`, `.srv`, and `.action` file must have:
1. A `# comment` explaining what it represents
2. Units (mm, cm, m, mA, V, %, ms, s)
3. Value range or valid values
4. Which node publishes / which nodes consume (for `.msg` files)
5. A `# DEPRECATED` marker if the field is being phased out

This is enforced by a CI linting script that checks every field has at least one comment.

---

## 11. CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/virtus_msgs.yml
name: virtus_msgs CI

on:
  push:
    paths: ['src/virtus_msgs/**']
  pull_request:
    paths: ['src/virtus_msgs/**']

jobs:
  build-and-validate:
    runs-on: ubuntu-latest
    container: ros:jazzy
    steps:
      - uses: actions/checkout@v4

      - name: Build virtus_msgs
        run: |
          source /opt/ros/jazzy/setup.bash
          colcon build --packages-select virtus_msgs

      - name: Lint message fields (all fields commented)
        run: python3 scripts/lint_msg_fields.py src/virtus_msgs/

      - name: Check version bump on change
        run: python3 scripts/check_version_bump.py src/virtus_msgs/

      - name: Build all dependent packages
        run: |
          source /opt/ros/jazzy/setup.bash
          colcon build --packages-up-to porter_orchestrator porter_esp32_bridge porter_ai_assistant

      - name: Run dependent package tests
        run: |
          source /opt/ros/jazzy/setup.bash
          colcon test --packages-up-to porter_orchestrator porter_esp32_bridge
```

### lint_msg_fields.py

Checks every field in every `.msg` file has an inline comment:

```python
import os, sys, re

def check_msg_file(path):
    errors = []
    with open(path) as f:
        for i, line in enumerate(f, 1):
            line = line.rstrip()
            if not line or line.startswith('#'):
                continue  # blank line or comment-only line
            if '#' not in line:
                errors.append(f"{path}:{i}: Field missing comment: '{line}'")
    return errors
```

---

## 12. Migration from Current State

### Step 1 — Audit (Day 1, 2 hours)
Run `grep -r "msg_type\|SensorFusion\|OrchestratorState\|BridgeFrame" src/ --include="*.py" --include="*.cpp" --include="*.h"` to find all places in the codebase that currently reference these types informally.

### Step 2 — Create virtus_msgs package (Day 1, 3 hours)
Write all `.msg`, `.srv`, `.action` files. Create `package.xml` and `CMakeLists.txt`. Build and verify with `colcon build --packages-select virtus_msgs`.

### Step 3 — Migrate porter_esp32_bridge (Day 1, 2 hours)
Add `virtus_msgs` dependency. Replace any inline struct definitions with VDL imports. Rebuild and test.

### Step 4 — Migrate porter_orchestrator (Day 2, 2 hours)
Same process. The FSM state machine's state publishing should use `OrchestratorState.msg`.

### Step 5 — Migrate porter_ai_assistant (Day 2, 2 hours)
Replace `PassengerCommand` and `AIResponse` inline definitions.

### Step 6 — Generate Dart models for Flutter GUI (Day 2, 1 hour)
Run code generator. Verify Flutter compiles.

### Step 7 — Update ROS 2 Studio and Hardware Dashboard extensions (Day 3, half day)
The topic registries in these extensions reference type strings. Update to use VDL type strings.

**Total migration effort: ~2.5 days**

---

## 13. Phased Build Plan

### Phase 1 — Core Messages (Day 1)
- Create package structure + `package.xml` + `CMakeLists.txt`
- Implement: `SensorFusion`, `OrchestratorState`, `BridgeFrame`, `PassengerCommand`, `AIResponse`
- Build passes with `colcon build --packages-select virtus_msgs`
- **Deliverable:** Core message types compile cleanly

### Phase 2 — Telemetry + Fleet Messages (Day 1)
- Implement: `RobotStatus`, `PowerTelemetry`, `MotorTelemetry`, `FleetHeartbeat`, `IncidentEvent`, `BridgeFrameRaw`
- **Deliverable:** All message types defined

### Phase 3 — Services + Actions (Day 2)
- Implement all 5 services + `EscortPassenger` action
- **Deliverable:** Full interface library compiles

### Phase 4 — Migration (Days 2–3)
- Migrate all 3 existing packages to import from virtus_msgs
- Generate Dart models for Flutter GUI
- **Deliverable:** Zero inline message definitions remain in the codebase

### Phase 5 — CI + Linting (Day 3)
- GitHub Actions workflow
- `lint_msg_fields.py` check
- `check_version_bump.py` check
- **Deliverable:** Automated enforcement of VDL standards

---

## 14. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| Interface format | ROS 2 IDL (.msg/.srv/.action) | Native ROS 2, generates Python + C++ bindings automatically |
| Build system | ament_cmake + rosidl | Standard ROS 2 toolchain |
| Versioning | semver (package.xml `<version>`) | Standard, enforced by CI |
| Dart codegen | Custom Python script | rosbridge uses JSON — need matching Dart models |
| Field linting | Custom Python CI script | Enforce documentation standards |
| Dependency pinning | `version_gte` / `version_lt` in package.xml | Breaks CI on unexpected major version changes |

---

## 15. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Breaking change deployed without migration window | Medium | High | `check_version_bump.py` in CI blocks merges that bump major without CHANGELOG entry |
| Dart code generation drifts from .msg files | Medium | Medium | Dart codegen script runs in CI — build fails if generated files differ from committed files |
| Field added without comment fails linting | High | Low | `lint_msg_fields.py` catches this; easy fix |
| Flutter rosbridge JSON field naming mismatch | Medium | High | Integration test in CI: publish from ROS 2, receive in mock Flutter client, assert all fields |
| Package renamed from virtus_msgs to something else | Low | High | Once published and depended on, never rename — document this as a hard rule |
