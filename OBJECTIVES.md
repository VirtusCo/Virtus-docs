# OBJECTIVES.md — Porter Robot Project Goals & Context

> **Read this file for WHY we're building and WHERE we're going.**
> Read `Claude.md` for HOW to build and WHAT rules to follow.
> Read `COMPANY.md` for WHO we are and Claude's role in the team.
>
> Last updated: 08 Mar 2026 · VirtusCo Engineering

---

## 1. Company

| Field | Value |
|-------|-------|
| **Name** | VirtusCo |
| **Website** | [virtusco.in](https://virtusco.in) |
| **Location** | Thripunithara, Kerala, India |
| **Mission** | Democratizing robotics — making automation accessible to businesses of all sizes |
| **Stage** | Pre-seed / Pre-funding — building prototype to raise investment |
| **Email** | virtusco.tech@gmail.com |

### Founding Team (5 members)

| Name | Role |
|------|------|
| **Antony Austin** | Founder — Robotics engineer (autonomous systems, embedded, AI). Primary software dev. |
| **A. Azeem Kouther** | Founder — Robotics engineer (mechanical systems, hardware) |
| **Allen George Thomas** | Founder — Product designer (UX, CAD models) |
| **Alwin George Thomas** | Founder — Financial strategist (investment planning, revenue modelling) |
| **Danush Krishna** | Founder — Robotics engineer (hardware, embedded systems) |

### GitHub / Social

| Platform | Link |
|----------|------|
| GitHub (company) | [github.com/VirtusCo](https://github.com/VirtusCo) |
| GitHub (this repo) | [github.com/austin207/Porter-ROS](https://github.com/austin207/Porter-ROS) |
| LinkedIn | [linkedin.com/company/virtusco](https://www.linkedin.com/company/virtusco/) |
| X / Twitter | [x.com/VirtuscoTech](https://x.com/VirtuscoTech) |

---

## 2. Product Vision

### What is Porter?

An **autonomous luggage-carrying robot** designed primarily for **airports**. Passengers interact via a touchscreen display. Porter follows them through the terminal, carrying their luggage autonomously while providing flight info, wayfinding, and check-in assistance.

### Target Markets (in rollout order)

1. **Airports** (primary — prototype target)
2. Cruise ships
3. Hotels & resorts
4. Restaurants
5. Any environment where luggage/goods need autonomous transport

### Key Product Features

| Feature | Description |
|---------|-------------|
| **120 kg payload capacity** | Robust chassis for multiple heavy bags, stable while manoeuvring |
| **Autonomous navigation** | LIDAR + sensor fusion + Nav2 for complex indoor layouts, crowd avoidance |
| **Adjustable lifting mechanism** | Smart lift system for various luggage sizes; ground ↔ platform ↔ counter transfer |
| **Interactive display** | Touchscreen: check-in, boarding gates, flight status, airport maps, travel assistant |
| **Airport-system integration** | Ticket & boarding pass scanning for gate-aware navigation |
| **Built-in weighing scale** | Baggage compliance check |
| **ESD-safe design** | Electrostatic discharge protection for airport environments |
| **Dual-mode operation** | Fixed-path autonomous + semi-autonomous |

### Business Model

Robot-as-a-Service (RaaS) to airports/venues. Revenue from deployment, maintenance, and partnerships. Investor materials available at [virtusco.in/investor](https://virtusco.in/investor).

---

## 3. Hardware Architecture

### Compute Hierarchy

```
┌─────────────────────────────────────────────────────┐
│          Raspberry Pi 4 / 5 (Master)                │
│                                                     │
│  • ROS 2 Jazzy (Docker)                             │
│  • YDLIDAR driver (direct serial connection)        │
│  • Nav2 navigation stack                            │
│  • Sensor fusion consumer                           │
│  • Motor command publisher                          │
│  • Interactive display (touchscreen)                │
│  • OTA update manager                               │
│                                                     │
│         ┌──── Serial/USB ────┐  ┌── Serial/USB ──┐  │
│         ▼                    │  ▼                │  │
│  ┌─────────────┐      ┌─────────────┐            │  │
│  │  ESP32 #1   │      │  ESP32 #2   │            │  │
│  │  (Motors)   │      │  (Sensors)  │            │  │
│  │             │      │             │            │  │
│  │ Zephyr RTOS │      │ Zephyr RTOS │            │  │
│  │             │      │             │            │  │
│  │ • Motor PWM │      │ • ToF       │            │  │
│  │ • Encoders  │      │ • Ultrasonic│            │  │
│  │   (future)  │      │ • Microwave │            │  │
│  │ • Lift ctrl │      │ • Sensor    │            │  │
│  │             │      │   fusion    │            │  │
│  └─────────────┘      └─────────────┘            │  │
└─────────────────────────────────────────────────────┘
```

### LIDAR

| Field | Prototype | Production |
|-------|-----------|------------|
| **Model** | YDLIDAR X4 Pro 360° | TBD (higher-end, decided later) |
| **Interface** | Serial USB → RPi directly | Same |
| **Baudrate** | 128000 | Model-dependent |
| **Driver** | Custom ROS 2 Jazzy C++ driver | Same codebase, parameterised for model |

> **Note:** The 25 Feb devlog identified hardware as S2PRO / YD-47 via `tri_test`. The prototype target is the **YDLIDAR X4 Pro 360°**. The driver must be model-agnostic via parameters so swapping models is a config change, not a code change.

### ESP32 #1 — Motor Controller

| Field | Value |
|-------|-------|
| **Firmware** | Zephyr RTOS |
| **Controls** | 2× 12V DC 100RPM 40Kgcm motors via 2× BTS7960 dual H-bridge drivers, lifting mechanism |
| **Drive type** | Differential drive (2 powered wheels + casters) |
| **Encoders** | Not installed yet — code must support future encoder feedback for odometry |
| **Receives from RPi** | Velocity commands (linear x, angular z), lift commands |
| **Sends to RPi** | Motor status, encoder data (future), lift position |

### ESP32 #2 — Sensor Fusion Unit

| Field | Value |
|-------|-------|
| **Firmware** | Zephyr RTOS |
| **Sensors** | Time-of-Flight (ToF), Ultrasonic, Microwave |
| **On-board processing** | Kalman filter — fuses ToF + Ultrasonic + Microwave into unified obstacle estimate |
| **Sends to RPi** | Fused environment/obstacle data (distances, obstacle classification) |
| **Receives from RPi** | Configuration commands (thresholds, enable/disable sensors) |

### RPi ↔ ESP32 Communication Protocol

**Requirements:** Fast, reliable, not complex.

**Recommended: USB CDC Serial** (top pick for this project)

| Protocol | Speed | Complexity | Reliability | Verdict |
|----------|-------|-----------|-------------|---------|
| **USB CDC Serial** | ~1 Mbps | Low (plug-and-play, `/dev/ttyACM*`) | High (flow control, CRC) | ✅ **Best fit** |
| UART (direct) | 115200–921600 baud | Low | Good (no flow control by default) | ✅ Good backup |
| SPI | 10+ Mbps | High (wiring, CS lines, ISR) | High | ❌ Over-engineered for this data rate |
| I2C | 400 kbps–1 Mbps | Medium (addressing, pull-ups) | Medium (no CRC, clock stretching issues) | ❌ Too slow, fragile |
| WiFi/UDP | Variable | Medium | Low (packet loss, latency jitter) | ❌ Unreliable for motor control |

**Why USB CDC Serial wins:**
- Each ESP32 appears as `/dev/ttyACM0`, `/dev/ttyACM1` — Docker can pass these through
- ~1 Mbps is more than enough for cmd_vel (50 Hz × 20 bytes) + sensor data (50 Hz × 100 bytes)
- Built-in flow control, no GPIO wiring beyond the USB cable
- Hot-pluggable — supports recovery if ESP32 resets
- Zephyr has native USB CDC ACM support
- udev rules can give stable names: `/dev/esp32_motors`, `/dev/esp32_sensors`

**Message protocol (custom binary, to be designed):**
```
[HEADER: 2B] [MSG_ID: 1B] [LENGTH: 2B] [PAYLOAD: NB] [CRC16: 2B]
```

### Cross-Compilation Targets

| Target | Architecture | Board | Priority |
|--------|-------------|-------|----------|
| Dev PC | amd64 | Any x86_64 Linux | Primary (daily dev) |
| Raspberry Pi 5 | arm64 / aarch64 | RPi 5 | Primary (prototype robot) |
| Jetson (Nano/Orin) | arm64 / aarch64 | NVIDIA Jetson | Secondary (future upgrade) |
| Latte Panda | amd64 | LP Delta/Sigma | Tertiary (alternative SBC) |

Cross-compilation via Docker multi-arch builds (`docker buildx`) + QEMU or native cross-toolchains.

### Interactive Display

| Option | Overhead | Pros | Cons |
|--------|----------|------|------|
| **Flutter (SELECTED)** | Low (~50 MB) | Native ARM64, Material Design 3, cross-platform, hot-reload | Needs Flutter SDK for dev |
| HDMI direct (Qt/GTK app) | Low | Native performance, no network stack | Harder to develop, needs display server |
| Web UI (localhost browser) | Medium | Easy to develop (HTML/CSS/JS), hot-reload | Browser overhead (~200MB RAM) |

**Decision: Flutter.** Cross-compiles to Linux x64 (dev) and ARM64 (RPi). ~50 MB release bundle. Material Design 3 UI. Direct HTTP to AI server (no rosbridge needed for chat). Rosbridge for ROS 2 topics (sensors, nav, motors). Single codebase for future Android/iOS kiosk mode.

---

## 4. Software Architecture

### Layer Stack

```
┌─────────────────────────────────────────┐
│           Interactive Display           │  ← Touchscreen UI (web or native)
├─────────────────────────────────────────┤
│         Porter Orchestrator             │  ← Python: state machine, boot seq
│         (porter_orchestrator)           │     health monitoring, recovery
├─────────────────────────────────────────┤
│            Nav2 Stack                   │  ← Path planning, obstacle avoidance
│           (nav2_config)                 │     costmaps, controller server
├─────────────────────────────────────────┤
│      Porter LIDAR Processor             │  ← Python: filtering, smoothing,
│     (porter_lidar_processor)            │     ROI cropping, downsampling
├────────────────┬────────────────────────┤
│  YDLIDAR Driver │  ESP32 Bridge Nodes   │  ← C++: serial driver, LaserScan pub
│ (ydlidar_driver)│ (porter_esp32_bridge) │     Python/C++: motor cmd, sensor sub
├────────────────┴────────────────────────┤
│          Docker + OTA Layer             │  ← Container management, updates
├─────────────────────────────────────────┤
│     Raspberry Pi / Linux / Zephyr       │  ← OS + firmware
└─────────────────────────────────────────┘
```

### Data Flow

```
YDLIDAR ──serial──► ydlidar_driver ──/scan──► porter_lidar_processor ──/scan/processed──► Nav2
                                                                                          │
ESP32 #2 ──USB──► esp32_sensor_bridge ──/environment──► Nav2 costmap (obstacle layer)     │
                                                                                          │
                                        Nav2 ──/cmd_vel──► esp32_motor_bridge ──USB──► ESP32 #1
```

---

## 5. Current Status (Updated 09 Mar 2026)

### What Exists

| Item | Status |
|------|--------|
| Repo structure (`src/` with all package dirs) | ✅ On GitHub, `prototype` branch |
| Docker dev environment (Dockerfile.dev, compose) | ✅ Functional (known COPY glob bug) |
| Docker prod environment (Dockerfile.prod, multi-stage) | ✅ Functional — `ros:jazzy-ros-base` runtime, unique BuildKit cache IDs |
| `Claude.md` (full instruction set, 18 sections, 17 tasks, 28 lessons) | ✅ Complete |
| `skills/` (16 ROS 2 Jazzy reference files + 12 Zephyr) | ✅ Complete |
| 25 Feb devlog (HW investigation) | ✅ Complete |
| 07 Mar devlog (Tasks 1–6, hardware test, bug fixes) | ✅ Complete |
| 07 Mar CI/CD devlog (6 CI/CD pipeline fixes) | ✅ Complete |
| 10 Mar devlog (Tasks 10–17, ESP32 Phase 3 complete) | ✅ Complete |
| 09 Mar Tool-Use Fix devlog (retrain, compact prompt, GUI humanization) | ✅ Complete |
| LIDAR hardware confirmed working (SDK `tri_test`) | ✅ Verified |
| **`ydlidar_driver` C++ package (Tasks 1–4)** | ✅ Complete — builds, publishes `/scan`, diagnostics, health monitor. 9 tests pass. Hardware validated. |
| **`porter_lidar_processor` Python package (Task 5)** | ✅ Complete — 6-stage filter pipeline, `/scan` → `/scan/processed`. 24 tests pass. |
| **`porter_orchestrator` Python package (Task 6)** | ✅ Complete — 9-state FSM, health monitor, boot grace period. 23 tests pass. Hardware validated — reaches READY in ~2s. |
| **ESP32 common library (Tasks 10–12)** | ✅ Complete — CRC16-CCITT, binary protocol parser/encoder, transport abstraction (UART/CDC_ACM/Mock via Kconfig). 60 Ztest pass on native_sim. |
| **Motor controller firmware (Task 13)** | ✅ Complete — SMF state machine, BTS7960 PWM, differential drive, heartbeat watchdog, speed ramping. Builds for esp32_devkitc (110 KB). |
| **Sensor fusion firmware (Task 14)** | ✅ Complete — SMF, VL53L0x ToF + HC-SR04 ultrasonic + RCWL-0516 microwave, 1D Kalman filter, cross-validation. Builds for esp32_devkitc (119 KB). |
| **`porter_esp32_bridge` C++ package (Task 16)** | ✅ Complete — motor bridge (/cmd_vel → ESP32) + sensor bridge (ESP32 → /environment). 8 lint tests pass. |
| **udev rules (Task 17)** | ✅ Complete — template rules + installer for /dev/esp32_motors, /dev/esp32_sensors. |
| **`porter_ai_assistant` Python package (Task 18a–18f)** | ✅ Complete — 12K dataset, Qwen 2.5 1.5B Instruct, QLoRA fine-tuned (conv eval_loss=0.1365/95.5% acc, tool_use eval_loss=0.0513/98.15% acc after retrain with compact prompt). DPO reinforcement learning on top of SFT (conv eval_loss=2.25e-10, tool_use eval_loss=1.34e-6). GGUF Q4_K_M 940 MB merged (4 variants: SFT+DPO × conv+tool_use). Benchmarks: DPO conv 1172ms/100% <2s/42.7 tok/s, DPO tool_use 724ms/100% <2s/41.4 tok/s, 100% tool compliance. AI persona: **Virtue**. Humanized GUI tool responses. 57 tests pass. |
| **Conversation orchestrator** | ✅ Complete — custom lightweight orchestrator (no LangChain). ToolExecutor (14 tools), ConversationOrchestrator (session memory, tool call loop), orchestrator_node.py (ROS 2). 35 tests pass. |
| **Total test suite** | ✅ 398 tests (244 colcon + 60 Ztest + 85 AI + 9 Flutter), 0 failures |
| **Flutter GUI** | ✅ Complete — 6 screens (Chat, Flights, Map, Status, Follow, E-Stop). AI chat via HTTP. Performance optimized (streaming batching, RepaintBoundary, 100-msg cap, exponential backoff). E-stop disengage fixed. `flutter analyze` clean, 9/9 tests, 50 MB release. |
| **AI HTTP server** | ✅ Standalone `ai_server.py` — POST /api/chat with multi-turn history, GET /api/health, /api/status. No ROS 2 dependency for GUI↔AI. |
| **SSE streaming** | ✅ Real-time token streaming via Server-Sent Events. POST /api/chat/stream. Flutter GUI consumes SSE for character-by-character reveal. |
| **RAG knowledge base** | ✅ 5 JSON files (41 docs): facilities, terminals, transport, dining, services. TF-IDF + keyword boosting retriever (<1ms, zero deps). 30 tests. |
| **CPU/SLAM coexistence** | ✅ `n_threads=2` (reserves 2 cores for SLAM), `n_ctx=1024` (saves 28 MB), Docker `cpus:2.0`/`mem_limit:2g`, `PORTER_NICE=10`. RPi 4 memory budget: ~2 GB total (fits 4 GB). |
| **CI/CD Pipeline** | ✅ GitHub Actions: `verify.yml` (9 jobs incl. Flutter), `build-release.yml` (5 jobs incl. Flutter GUI build). Semver, conventional commit changelog, categorized release downloads. |

### What Does NOT Exist Yet

| Item | Status |
|------|--------|
| Docker improvements (Task 7) | ✅ Fixed COPY paths, multi-stage prod, entrypoint, 3 compose services |
| CI/CD pipeline (Task 8) | ✅ GitHub Actions verify.yml (9 jobs incl. Flutter) + build-release.yml (5 jobs incl. Flutter GUI build). Semver, conventional commits, categorized releases. |
| Documentation (Task 9) | ✅ Complete — README, CHANGES.md, Driver Porting Guide, Contribution Guide |
| ESP32 Zephyr firmware (Phase 3) | ✅ Complete — all F1-F7 + B1-B2 tasks done. 60 Ztest + 187 colcon = 247 tests, 0 failures |
| AI Assistant fine-tuning (Task 18c) | ✅ QLoRA on RTX 5070 — Qwen conv eval_loss=0.1365 (95.5% acc), tool_use eval_loss=0.0513 (98.15% acc, retrained with compact prompt). DPO RL: conv eval_loss=2.25e-10, tool_use eval_loss=1.34e-6 |
| AI Assistant quantization (Task 18d) | ✅ Merged LoRA + Q4_K_M GGUF 940 MB each — 4 variants: SFT conv, SFT tool_use, DPO conv, DPO tool_use |
| AI Assistant integration/benchmarks (Task 18f) | ✅ DPO conv 1172ms/100% <2s/42.7 tok/s, DPO tool_use 724ms/100% <2s/41.4 tok/s, 100% tool compliance. GUI displays humanized tool responses. |
| Conversation orchestrator | ✅ Custom lightweight orchestrator (replaced LangChain plan). ToolExecutor + ConversationOrchestrator + ROS 2 node. 35 tests. |
| Nav2 configuration | ❌ |
| URDF / robot model | ❌ |
| Display UI | ✅ Complete — Flutter GUI, 6 screens, AI chat integrated, perf optimized, CI/CD builds Linux binary |
| OTA update system | ❌ |
| AI DPO reinforcement learning | ✅ Complete — DPO on synthetic preference data (1500 tool_use + 2000 conv pairs). Conv ~4% faster, tool_use 100% compliance. See DevLogs/09_Mar_DPO_Training_Logs.md |
| AI quality upgrade (Phase 4.6) | ❌ Planned — RAG + output guardrails on top of Qwen 2.5 1.5B |
| AI production hardening (Phase 4.7) | ❌ Planned — context window fix, per-adapter temperature, tool stubs → real, adapter pre-loading, entity extraction, output guardrails, request queue, real airport KB |

---

## 6. Objectives & Timeline

### PHASE 1 — LIDAR Driver (NOW → 2 weeks — deadline ~14 Mar 2026)

> **Goal:** Production-grade C++ LIDAR driver publishing `/scan` in ROS 2 Jazzy.
> **Owner:** Claude (AI) writes all code. Antony reviews, tests on hardware, directs.

| Week | Deliverables |
|------|-------------|
| **Week 1** | Task 1 (package skeleton) + Task 2 (SDK adapter/parser) + Task 3 (LaserScan pub) + Task 7 (Docker fixes) |
| **Week 2** | Task 4 (diagnostics) + Task 5 (lidar processor) + Task 8 (tests) + Task 9 (docs) |

**Definition of Done:**
- [x] `ydlidar_driver` builds clean with `colcon build` ✅ (7 Mar 2026)
- [x] Publishes valid `sensor_msgs/LaserScan` on `/scan` from real YDLIDAR X4 Pro hardware ✅ (7 Mar 2026)
- [x] Diagnostics on `/diagnostics` — health, scan rate, error count ✅ (7 Mar 2026)
- [x] `porter_lidar_processor` filters and republishes on `/scan/processed` ✅ (8 Mar 2026)
- [x] Unit tests pass (GTest for C++, pytest for Python) ✅ 99 tests, 0 failures
- [x] Lint passes (ament_cpplint, ament_flake8) ✅
- [x] Docker dev image builds and runs correctly ✅ (7 Mar 2026)
- [x] Driver is model-agnostic — swapping LIDAR model = config change only ✅
- [x] README with parameters, supported models, quick-start ✅ (7 Mar 2026)

### PHASE 2 — Navigation (weeks 3–4)

> **Goal:** Robot navigates autonomously using Nav2 + LIDAR.

- [ ] Nav2 bringup with LIDAR scan input
- [ ] SLAM (slam_toolbox) for map building
- [ ] Localisation (AMCL) on known map
- [ ] Basic waypoint navigation demo
- [ ] Obstacle avoidance validated

### PHASE 3 — ESP32 Integration (weeks 4–5)

> **Goal:** Motor control and sensor fusion via ESP32 bridge nodes.
> **Hardware:** ESP32-WROOM (current) — firmware must be ESP-agnostic (WROOM/S2/S3 via Kconfig).
> **Communication:** UART through USB-UART bridge (WROOM) or native USB CDC ACM (S2/S3) — abstracted via transport layer.

#### Firmware Tasks (Zephyr RTOS — `esp32_firmware/`)

| Task | Description | Location | Status |
|------|-------------|----------|--------|
| **F1** | CRC16-CCITT implementation | `common/src/crc16.c` | ✅ 256-byte lookup, 14 Ztest pass |
| **F2** | Protocol parser & encoder (byte-by-byte state machine) | `common/src/protocol.c` | ✅ 9-state parser, encoder, ACK/NACK. 26 Ztest pass |
| **F3** | Transport abstraction layer (UART vs CDC ACM via Kconfig) | `common/src/transport.c` | ✅ 3 backends via Kconfig. 20 Ztest pass |
| **F4** | Motor controller firmware (SMF, PWM, differential drive, heartbeat watchdog, speed ramping) | `motor_controller/src/` | ✅ ~730 lines, builds clean (110 KB) |
| **F5** | Sensor fusion firmware (SMF, ToF I2C, Ultrasonic GPIO, Microwave ADC, Kalman filter) | `sensor_fusion/src/` | ✅ ~750 lines, builds clean (119 KB) |
| **F6** | Ztest unit tests (CRC, protocol, transport — `native_sim`) | `tests/` | ✅ 60/60 pass on native_sim |
| **F7** | udev rules for stable device names (`/dev/esp32_motors`, `/dev/esp32_sensors`) | `esp32_firmware/udev/` | ✅ Template rules + installer |

#### ROS 2 Bridge Tasks (on RPi — `src/porter_esp32_bridge/`)

| Task | Description | Status |
|------|-------------|--------|
| **B1** | `esp32_motor_bridge` node — subscribes `/cmd_vel`, encodes binary protocol, sends to ESP32 #1, publishes motor status | ✅ Differential drive, heartbeat, diagnostics. 8 lint tests pass |
| **B2** | `esp32_sensor_bridge` node — receives fused sensor data from ESP32 #2, publishes `/environment`, sends config commands | ✅ Range pub, status polling, diagnostics. 8 lint tests pass |

#### Definition of Done (Phase 3):
- [x] CRC16 + protocol parser passes Ztest on `native_sim` ✅ (10 Mar 2026 — 60/60 pass)
- [x] Transport layer builds for both UART and CDC ACM targets ✅ (10 Mar 2026 — 3 backends via Kconfig)
- [x] Motor controller firmware builds for `esp32_devkitc` — motors respond to RPi commands ✅ (10 Mar 2026 — 110 KB)
- [x] Sensor fusion firmware builds — fused obstacle data flows to RPi ✅ (10 Mar 2026 — 119 KB)
- [x] Bridge nodes build with `colcon build` and connect ROS 2 topics to ESP32 serial ✅ (10 Mar 2026 — all lint pass)
- [x] Heartbeat watchdog: motors stop if RPi silent for 500ms ✅ (implemented in motor_controller + motor_bridge)
- [x] Speed ramping: no instant full-speed transitions ✅ (implemented in motor_controller)
- [x] udev rules give stable `/dev/esp32_*` device names ✅ (10 Mar 2026 — template + installer)

### PHASE 4 — Orchestration & State Machine (week 5) — ✅ COMPLETED EARLY (10 Mar 2026)

> **Goal:** Full system bringup, health monitoring, recovery.
> **Completed as Task 6 during Phase 1.** State machine and health monitor built, tested (23 tests), and hardware validated.

- [x] `porter_orchestrator` state machine (INITIALISING → DRIVER_STARTING → HEALTH_CHECK → PROCESSOR_STARTING → READY ⇄ DEGRADED → ERROR → RECOVERY → SHUTDOWN)
- [x] Boot sequence: driver → health check → processing → ready (with 8s boot grace period)
- [x] Health monitoring: LIDAR dropout, scan rate deviation, invalid point ratio → recovery actions
- [ ] Service endpoints: restart driver, reconfigure, emergency stop (partial — restart via state transitions, full service API deferred)
- [x] `lidar_health_monitor` node: subscribes `/diagnostics` + `/scan`, publishes `/porter/health_status`

### PHASE 4.5 — AI Assistant (Qwen 2.5 1.5B — between Phase 4 & 5)

> **Goal:** On-device LLM for smart airport passenger assistant on RPi 4/5.
> **Model:** Qwen 2.5 1.5B Instruct — fine-tuned with QLoRA for airport domain. (Switched from Gemma 3 270M IT on 8 Mar 2026 due to quality limitations.)
> **Source:** `Qwen/Qwen2.5-1.5B-Instruct-GGUF` (Q4_K_M ~1.0 GB).
> **Base model for fine-tuning:** `Qwen/Qwen2.5-1.5B-Instruct` on HuggingFace.
> **Deployment:** Quantised GGUF on RPi 4/5 via llama-cpp-python.
> **Training hardware:** RTX 5070 (8 GB VRAM, Blackwell sm_120), PyTorch nightly cu128.

- [x] Collect/curate airport Q&A training dataset — 12K examples (7K conversational + 3K tool-use, 14 categories + 13 tool types), train/eval splits ✅ (7 Mar 2026)
- [x] Model selection & baseline — initially Gemma 3 270M IT (253 MB Q4_K_M), switched to Qwen 2.5 1.5B Instruct (~1.0 GB Q4_K_M) ✅ (7–8 Mar 2026)
- [x] Fine-tune with QLoRA adapter on airport domain — Qwen: conv eval_loss=0.1365 (95.5% token acc, 56 min), tool_use eval_loss≈0.0 (100% token acc, 67 min) ✅ (8 Mar 2026)
- [x] Quantise model (GGUF Q4_K_M) for RPi inference — merged LoRA + quantised to 940 MB each (conv + tool_use) ✅ (8 Mar 2026)
- [x] Build ROS 2 service node (`porter_ai_assistant`) — `~/query`, `~/get_status`, `/porter/ai_query`, `/porter/ai_response`. 22 tests pass ✅ (8 Mar 2026)
- [x] Benchmark: conv 1436ms/80% <2s, tool_use 1724ms/70% <2s, 1678 MB RSS, ~39 tok/s (RTX 5070 dev machine) ✅ (8 Mar 2026)
- [x] AI persona renamed to **Virtue** (Porter = robot product name) ✅ (8 Mar 2026)
- [x] Conversation orchestrator for GUI ↔ AI ↔ tools integration ✅ (8 Mar 2026) — custom lightweight (no LangChain). ToolExecutor (14 tools), ConversationOrchestrator (session memory, query classification, tool call loop), orchestrator_node.py (ROS 2 wrapper). 35 tests pass.
- [x] Multi-turn conversation history — server API accepts `history` array, GUI sends last 6 turns, system prompts grounded with explicit capability list ✅ (8 Mar 2026)
- [x] DPO reinforcement learning — synthetic preference data, conv eval_loss=2.25e-10, tool_use eval_loss=1.34e-6, 4 GGUF variants exported, DPO conv 4% faster ✅ (9 Mar 2026)
- [ ] Multi-language support (English primary, expandable)

#### Phase 4.5 Known Limitations (Gemma 3 270M — resolved by switching to Qwen 2.5 1.5B)

Observed during GUI integration testing with Gemma 3 270M (8 Mar 2026). Switching to Qwen 2.5 1.5B is expected to resolve most of these:

| # | Issue | Example | Root Cause |
|---|-------|---------|------------|
| 1 | **Mixed-language output** | "WiFi está disponible... ant到时候" (Spanish + Chinese) | 270M model can't reliably suppress non-English tokens from multilingual pretraining |
| 2 | **Fabricated specifics** | "100 meters ahead", "Level 2 near the escalator", "Terminal 3, about 8 minutes" | Model has no real airport layout data — hallucinated locations |
| 3 | **Incorrect topic grounding** | Asked "restroom?" → got "shower facilities"; asked "whats your cost" → got bag purchasing advice | 270M parameter budget too small for fine-grained concept separation |
| 4 | **Identity instability** | Calls itself "Virtus", "CiaoBlade" instead of "Virtue"; mentions "LIDBot" (non-existent) | 270M can't reliably anchor its own identity — system prompt gets overridden by pretraining noise |
| 5 | **No profanity handling** | "what the fuck" → interpreted as lost luggage complaint, gave fabricated steps | No profanity filter or redirect in system prompt or post-processing |
| 6 | **Off-topic requests not rejected** | "write c++" → generic greeting instead of "I can only help with airport questions" | Model tries to be helpful rather than staying in airport domain scope |
| 7 | **Hallucinated capabilities** | "Generate reports on flight status", "LIDBot" | Model invents features not listed in system prompt capabilities |

**Verdict:** 270M was viable for proof-of-concept but too small for production. **Switched to Qwen 2.5 1.5B Instruct (8 Mar 2026)** — 5.6× more parameters, native tool calling, strong English grounding.

### PHASE 4.6 — AI Quality Upgrade (RAG + Output Guardrails) ✅ PARTIALLY COMPLETE

> **Goal:** Add RAG for real airport data and output guardrails on top of Qwen 2.5 1.5B.
> **Model:** Qwen 2.5 1.5B Instruct — selected 8 Mar 2026, fine-tuned, DPO-trained, deployed. Model upgrade is complete.
> **Remaining:** Output guardrails (moved to Phase 4.7 task 20g).

#### Model Selection ✅ COMPLETE

**Qwen 2.5 1.5B Instruct** (Q4_K_M ~1.0 GB) — selected 8 Mar 2026. Community gold standard sub-2B model. Strong instruction following, native tool calling, ChatML format. Fits RPi 4 (~1.1 GB RSS + ~1.5 GB for stack = ~2.6 GB of 4 GB). RPi 5 (8 GB) — ample headroom. Replaced Gemma 3 270M (too small: language mixing, hallucinations, identity instability).

#### RAG (Retrieval-Augmented Generation) ✅ COMPLETE

5 JSON files, 41 documents. TF-IDF + keyword boost retriever (<1ms, zero deps). Integrated into orchestrator context building. Generic airport template — real airport KB deferred to Phase 4.7 task 20j.

| Task | Status |
|------|--------|
| 19d — Airport knowledge base (RAG) | ✅ 41 docs, 5 categories |
| 19e — RAG query pipeline | ✅ TF-IDF + keyword boost, integrated into orchestrator |
| 19f — Output guardrails | ❌ Moved to Phase 4.7 task 20g |
| 19g — Integration + benchmarks | ✅ Partial — benchmarks done, A/B vs 270M done |

#### Definition of Done (Phase 4.6):
- [x] Qwen 2.5 1.5B Instruct deployed — inference <2s on dev machine
- [x] QLoRA fine-tuned with 12K dataset — conv 95.5% acc, tool_use 98.15% acc
- [x] Airport KB with 41 entries (generic template, customisable per venue)
- [x] RAG pipeline injects real data into context
- [ ] Language guardrail: 0% non-English output → moved to Phase 4.7 task 20g
- [x] Hallucination rate reduced vs 270M (Qwen 1.5B is significantly better)
- [x] Benchmarks: DPO conv 1172ms/42.7 tok/s, tool_use 724ms/41.4 tok/s

### PHASE 4.7 — AI Production Hardening (code-review findings, 10 Mar 2026)

> **Goal:** Fix critical bugs and reliability gaps identified by code review before RPi deployment.
> **Source:** Code review of inference_engine.py, orchestrator.py, ai_server.py, tool_executor.py, rag_retriever.py.
> **Owner:** Claude writes all fixes. Antony reviews and tests on RPi 5.

#### Critical Fixes (must be done before any RPi deployment or demo)

| # | Task | Problem Found | Fix |
|---|------|---------------|-----|
| **20a** | **Context window fix** | `n_ctx=1024` is too small: system prompt (~478 tok) + RAG (~300 tok) + history (~200 tok) + query (~30 tok) = ~1008 tokens, leaving ~16 tokens for response. Model is silently truncated. | Raise `n_ctx` to 2048 (costs ~28 MB, RPi 5 has 8 GB headroom). Also inject tool schemas only when `classify_query()` returns `tool_use`, not for conversational queries (saves 478 tokens in conversational path). |
| **20b** | **Per-adapter temperature** | `temperature=0.7` applied to both adapters. Tool-use at 0.7 causes random JSON variation, breaking `parse_tool_call()`. High variance on structured output is the primary cause of tool-call failures post-deployment. | Separate temperature per adapter: conversational=0.7 (current), tool_use=0.15. Add `adapter_temperature` map to `ModelConfig`. |
| **20c** | **Tool stubs → honest fallbacks** | All 14 `ToolExecutor` tools return hardcoded mock data. Passenger asking "What's the status of AI302?" gets fabricated gate/time. This is a passenger safety/trust issue for an airport product. | (1) Wire `escort_passenger` → publish Nav2 goal to `/navigate_to_pose`. (2) Wire `weigh_luggage` → subscribe to hardware scale topic. (3) Wire `call_assistance` → publish staff notification topic. (4) For tools requiring external APIs (`get_flight_status`, `translate_text`) return honest fallback: "I don't have live data — please check the departures board." |
| **20d** | **Adapter switch latency** | `switch_adapter()` does full model unload + reload. On RPi this takes 10–30 seconds. A passenger asking a conversational question then a tool question freezes the robot for ~30s. | Pre-load both `conversational` and `tool_use` merged GGUF models at startup as two `Llama` instances. Route queries by classification without reloading. If RPi 4 RAM is insufficient, keep single adapter and only switch on explicit request (not per-query). |

#### High Priority Fixes

| # | Task | Problem Found | Fix |
|---|------|---------------|-----|
| **20e** | **Entity extraction into session context** | `Session.context` dict exists but is never populated. If passenger says "my flight is AI302", this is not remembered. Next turn asking "what's my gate?" has no context. | After each inference result, scan response + user query for flight numbers (`[A-Z]{2}\d+`), gate references (`gate [A-Z]\d+`), terminal numbers, and write into `session.context`. Prepend extracted context to RAG context string on subsequent turns. |
| **20f** | **Health endpoint bypasses inference lock** | `_engine_lock` blocks both inference and `/api/health`. During a 2s inference, Flutter's 30s health poller gets blocked. | Health endpoint reads cached `_engine.health` dataclass directly without acquiring the lock. Stats are updated after each inference, so reads are safe without locking. |
| **20g** | **Output guardrails** | No language filter, profanity redirect, or off-topic rejection implemented (listed as pending since Phase 4.6 Strategy C). | (1) **Language filter**: after inference, detect non-ASCII/non-English tokens → retry once with "Respond in English only." appended. (2) **Off-topic rejection**: before inference, check query against non-airport keyword set (code, math, recipes) → return template response without calling model. (3) **Profanity redirect**: detect profanity in user input → return "I understand your frustration. How can I help you with your journey?" without inference. |

#### Medium Priority Fixes

| # | Task | Problem Found | Fix |
|---|------|---------------|-----|
| **20h** | **Model warmup on startup** | First inference after `load_engine()` is 2–3× slower (llama.cpp JIT compilation + KV cache cold). Looks broken in demos or for first passenger. | Run a synthetic `"Hello"` query and discard the result during `load_engine()`. Adds ~5s to startup but every real query runs at full speed. |
| **20i** | **Request queue for concurrent users** | With `threading.Lock`, a second passenger's query hangs silently until the first inference completes (up to 5s). No feedback given. | Add a FIFO queue with depth 3. If queue full, return `{"status": "busy"}` immediately. For SSE clients, send `event: waiting` so Flutter shows a spinner instead of a frozen screen. |
| **20j** | **Real airport KB for deployment** | The 41 KB documents use generic T1/T2/T3 template data. When deployed at a specific airport (e.g., Cochin International), these are factually wrong — RAG injects wrong gate/floor/terminal info. Wrong facts are worse than no facts. | Add `kb_profile` field to `assistant_params.yaml`. Ship a real Cochin Airport KB file for first deployment. Keep generic template as fallback. KB update must not require retraining — JSON file swap only. |
| **20k** | **Background session cleanup** | `cleanup_expired_sessions()` only runs when `process_query()` is called. Idle robot accumulates stale sessions in memory. | Start a `threading.Timer` in `ConversationOrchestrator.__init__()` that calls `cleanup_expired_sessions()` every 300s. Reschedules itself on each run. |

#### Definition of Done (Phase 4.7):

- [ ] `n_ctx` raised to 2048; tool schemas injected only for tool_use adapter queries
- [ ] tool_use adapter runs at temperature 0.15; conversational stays at 0.7
- [ ] `escort_passenger` publishes to Nav2; `call_assistance` publishes notification; API-dependent tools return honest fallbacks
- [ ] Both adapters pre-loaded at startup; no mid-conversation model reload
- [ ] Entity extraction populates `session.context` with flight numbers and gate refs
- [ ] `/api/health` responds without acquiring inference lock
- [ ] Language filter + off-topic rejection + profanity redirect active
- [ ] Warmup query run on startup; first real query at full speed
- [ ] Request queue returns `event: waiting` for concurrent users
- [ ] Cochin Airport KB file exists and is loaded via `kb_profile` param
- [ ] Background session cleanup timer running

---

### PHASE 5 — Display & UX (week 6) — IN PROGRESS

> **Goal:** Interactive touchscreen for passenger interaction.
> **Stack:** Flutter (Dart) — cross-compiles to Linux x64/ARM64 for RPi. ~50 MB release bundle.
> **AI integration:** GUI connects directly to `ai_server.py` HTTP API (port 8085) with rosbridge fallback.
> **Target display:** 7-inch 800×480 touchscreen on RPi.

- [x] Flutter GUI skeleton — 6 screens: AI Chat, Flights, Airport Map, System Status, Follow-Me, Emergency Stop ✅ (8 Mar 2026)
- [x] AI chat integration — direct HTTP to `ai_server.py`, multi-turn conversation history, AI Online/Offline badge ✅ (8 Mar 2026)
- [x] `flutter analyze` → No issues, `flutter test` → 9/9, `flutter build linux --release` → ✓ ✅ (8 Mar 2026)
- [x] Polish chat UX — auto-scroll to latest message, streaming character batching (80ms/8chars), RepaintBoundary caching, 100-msg cap ✅ (8 Mar 2026)
- [ ] Flight info screen — display flight data (mock data for prototype, real API integration later)
- [ ] Airport map screen — interactive SVG/image map with pinch-zoom
- [ ] System status screen — battery, LIDAR health, motor status, sensor data from ROS 2 topics
- [ ] Follow-me mode — initiate/stop autonomous following from touchscreen
- [x] Emergency stop — prominent hardware/software E-stop accessible from any screen, long-press disengage ✅ (8 Mar 2026)
- [ ] Cross-compile for ARM64 (RPi) via `flutter build linux --release` on RPi or cross-build
- [ ] ROS 2 ↔ GUI communication via rosbridge for non-AI topics (motor, sensors, nav)
- [x] CI/CD Flutter build — `build-flutter-gui` job in build-release.yml, `flutter-verify` in verify.yml ✅ (8 Mar 2026)
- [x] Performance optimization — streaming rebuild rate reduced 75%, exponential backoff on rosbridge, unused deps removed ✅ (8 Mar 2026)

### PHASE 6 — Cross-Compilation & Deployment (weeks 6–7)

> **Goal:** Build on dev PC, deploy to RPi via Docker.

- [ ] Docker multi-arch builds (amd64 + arm64) via `docker buildx`
- [ ] Production Dockerfile (minimal `ros:jazzy-ros-base`)
- [ ] Docker on RPi running full stack
- [ ] Validated on physical Raspberry Pi 5

### PHASE 7 — OTA & Security (weeks 7–8)

> **Goal:** Field-updatable, airport-grade security.

**OTA Updates:**
- [ ] Docker container OTA for ROS 2 stack on RPi (pull new image, restart)
- [ ] ESP32 Zephyr firmware OTA (MCUboot + DFU over USB or BLE)
- [ ] Rollback mechanism if update fails
- [ ] Version tracking and update audit log

**Security (requires research — airport-grade):**
- [ ] DDS Security (authentication, access control, encryption) — `sros2` toolkit
- [ ] Encrypted RPi ↔ ESP32 communication (TLS or AES on serial frames)
- [ ] Secure boot on RPi (optional, research needed)
- [ ] SSH key-only access, no password auth
- [ ] Docker image signing (Docker Content Trust / Notary)
- [ ] Network segmentation if on airport WiFi
- [ ] Audit logging for all robot actions (compliance)

### PHASE 8 — Simulation & URDF (before MVP)

> **Goal:** Simulated Porter in Gazebo for testing without hardware.

- [ ] URDF/Xacro robot model matching real Porter dimensions
- [ ] Gazebo Ignition (Fortress/Harmonic) world: airport terminal
- [ ] Simulated LIDAR, sensors, differential drive
- [ ] Nav2 works in simulation
- [ ] CI tests run against simulation

### PHASE 9 — MVP & Investor Demo (target: ~10 weeks from start)

> **Goal:** Working prototype that demonstrates autonomous luggage carrying in a real space.

- [ ] Robot follows a path while carrying luggage
- [ ] Avoids obstacles (people, pillars, carts)
- [ ] Touchscreen shows useful info
- [ ] Reliable for a 5-minute demo loop
- [ ] Video recording for investor pitch

---

## 7. Technical Decisions Log

| # | Decision | Rationale | Date |
|---|----------|-----------|------|
| 1 | Option C: company-first driver, then open-source | Control over production-critical code, faster to prototype | 25 Feb 2026 |
| 2 | C++ for driver, Python for orchestration | Performance where needed, productivity where possible | 25 Feb 2026 |
| 3 | Custom driver from scratch (not fork/patch) | Official driver is EOL, need full control for startup product | 25 Feb 2026 |
| 4 | Docker on robot for deployment | Reproducible, portable, OTA-friendly | 28 Feb 2026 |
| 5 | USB CDC Serial for RPi ↔ ESP32 | Fast enough (~1 Mbps), reliable, simple, hot-pluggable, Zephyr native support | 28 Feb 2026 |
| 6 | Zephyr RTOS for ESP32 firmware | Production-grade RTOS, MCUboot OTA, good driver ecosystem | 28 Feb 2026 |
| 7 | Cross-compilation via Docker buildx | One build system for amd64 + arm64, no separate toolchain management | 28 Feb 2026 |
| 8 | YDLIDAR X4 Pro 360° for prototype | Available hardware; production model TBD. Driver must be model-agnostic. | 28 Feb 2026 |
| 9 | Driver model-agnostic via parameters | Swap LIDAR model = config YAML change, not code change | 28 Feb 2026 |
| 10 | S2PRO = S4 (SDK model code 4), marketed as "X4 Pro" | SDK has no "X4 Pro" model. `hasScanFrequencyCtrl()=false` → motor speed is hardware-regulated (DTR power), not software-controlled. 3.8 Hz actual rotation. | 10 Mar 2026 |
| 11 | Orchestrator built in Phase 1 (ahead of Phase 4 schedule) | Health monitor + state machine are tightly coupled to driver testing. Building them now ensures robust driver validation. | 10 Mar 2026 |
| 12 | `health_expected_freq` decoupled from motor `frequency` | S2PRO motor target 10 Hz but scan delivery ~3.8 Hz (5K samp ÷ 1300 pts/rev). Separate param avoids false health errors. | 10 Mar 2026 |
| 13 | Pure C for shared common library (crc16, protocol, transport) | Must compile for both Zephyr (Xtensa) and Linux (x86_64/arm64 ROS 2 bridge). C has universal ABI. | 10 Mar 2026 |
| 14 | Transport abstraction via Kconfig (compile-time, not runtime) | Zero runtime overhead; swap ESP32 variant (WROOM→S2/S3) = change Kconfig + overlay, not code. | 10 Mar 2026 |
| 15 | 500ms heartbeat watchdog on motor controller | Airport safety — motors must stop immediately if RPi crashes/disconnects. Short timeout prevents runaway robot. | 10 Mar 2026 |
| 16 | SMF (Zephyr State Machine Framework) for firmware states | Built into Zephyr, well-tested, enforces entry/run/exit handlers. Matches orchestrator pattern on RPi side. | 10 Mar 2026 |
| 17 | 1D Kalman filter for sensor fusion (not weighted average) | Handles noisy ToF/ultrasonic with predictive smoothing. Process noise tunable per sensor. Cross-validation catches drift. | 10 Mar 2026 |
| 18 | Qwen 2.5 1.5B Instruct for AI assistant (initially Gemma 3 270M) | Started with 270M (253 MB) for fast prototyping. Switched to Qwen 2.5 1.5B (~1.0 GB Q4_K_M) — 270M too small for production (language mixing, hallucinations). Qwen 2.5 1.5B is community gold standard sub-2B: strong instruction following, native tool calling, ChatML format. Fits RPi 4 (~1.1 GB RSS). | 8 Mar 2026 |
| 19 | llama-cpp-python for RPi inference (not vLLM/ONNX) | vLLM has no ARM64 support. llama.cpp has first-class ARM NEON + memory-mapping. GGUF format is compact and RPi-optimised. | 7 Mar 2026 |
| 20 | QLoRA 4-bit for fine-tuning (not full fine-tune) | RTX 5070 has 8 GB VRAM. Full fine-tune of even 270M needs >8 GB. QLoRA fits in 4–6 GB with 4-bit base + LoRA adapters. | 7 Mar 2026 |
| 21 | Modular LoRA adapters (conversational vs tool-use) | Separate adapters allow hot-swapping at inference. Airport queries routed by keyword classification to appropriate adapter. | 7 Mar 2026 |
| 22 | Base GGUF + runtime LoRA (not merge_and_unload) | `merge_and_unload()` on 4-bit QLoRA degrades weights (lossy NF4→bf16). Runtime LoRA loading via llama-cpp-python `lora_path` preserves full quality. | 8 Mar 2026 |
| 23 | AI persona "Virtue" distinct from product "Porter" | Avoids ambiguity in training data between robot hardware references and AI self-identification. | 8 Mar 2026 |
| 24 | Custom lightweight orchestrator (not LangChain) | LangChain too heavy for RPi 4 (~200 MB deps). Built custom ToolExecutor (14 tools) + ConversationOrchestrator (session memory, classify→infer→tool→re-infer loop) + ROS 2 node. Zero external deps beyond llama-cpp-python. | 8 Mar 2026 |
| 25 | Upgrade path: Qwen 2.5 1.5B + RAG (not just bigger model) | 270M hallucinated locations, mixed languages, fabricated details. Upgraded to Qwen 2.5 1.5B (5.6× parameters). RAG still needed for factual airport data grounding. 1.5B fits RPi 4 (~1.1 GB RSS) with nav stack. | 8 Mar 2026 |
| 26 | Flutter for GUI (not Web UI or Qt) | Cross-compiles to Linux ARM64 (RPi), native performance (~50 MB bundle), Material Design 3, hot-reload for dev, single codebase for future Android/iOS kiosk. Faster to build than Qt, lighter than web browser. | 8 Mar 2026 |
| 27 | No-op `onTap` in Flutter `GestureDetector` for e-stop | `onTap: null` removes tap recognizer from gesture arena, silently breaking `onLongPress` disambiguation. `onTap: () {}` keeps both recognizers active. | 8 Mar 2026 |
| 28 | `subosito/flutter-action@v2` for CI Flutter builds | Official community action, handles SDK caching, version pinning (`3.x` channel stable), and PATH setup. Lighter than manual install. | 8 Mar 2026 |
| 29 | Conventional commit labels in release notes (no emojis) | Categorized downloads with `[feat]`/`[fix]` type tags, collapsible Quick Start, metadata table. Clean professional appearance for investor demos. | 8 Mar 2026 |
| 30 | Streaming character batching (80ms/8chars) for Flutter chat | Initial 20ms/2chars caused ~50 rebuilds/sec per message. Batching reduces to ~12/sec (75% reduction). Combined with RepaintBoundary and AnimatedSize for smooth UX on RPi. | 8 Mar 2026 |

---

## 8. Constraints & Non-Negotiables

1. **Airport safety** — this robot operates around passengers, children, elderly. Safety is paramount. Fail-safe on any error. Emergency stop always available.
2. **Prototype before funding** — speed matters, but code must be production-grade. No throwaway code.
3. **All driver code written by Claude** — Antony reviews, tests on hardware, provides direction. Claude must produce complete, buildable, tested code.
4. **Docker everywhere** — dev, CI, and robot deployment all containerised.
5. **Cross-platform** — must build for RPi (arm64), Jetson (arm64), dev PC (amd64) from same source.
6. **Model-agnostic driver** — changing LIDAR model = YAML config change. No hardcoded model assumptions in code.
7. **2-week deadline for Phase 1** — LIDAR driver must be functional on hardware by ~14 Mar 2026.

---

## 9. Open Research Items

| # | Topic | Why | Priority |
|---|-------|-----|----------|
| 1 | Airport robot security standards | Airport product must meet safety/security regulations. Research ISO, GDPR (passenger data on display), airport authority requirements. | High |
| 2 | DDS Security (`sros2`) | Encrypt all ROS 2 communications. Evaluate performance overhead on RPi. | High |
| 3 | ~~Display: Web UI vs native Qt~~ | **Resolved:** Flutter selected (Decision #26). Cross-compiles to Linux ARM64, ~50 MB bundle, Material Design 3. | ~~Medium~~ ✅ |
| 4 | ESP32 firmware OTA via MCUboot | Zephyr + MCUboot DFU path. Research USB DFU vs BLE OTA for field updates. | Medium |
| 5 | Production LIDAR model selection | Higher-range, higher-resolution model for production. Evaluate options when funding secured. | Low (post-MVP) |
| 6 | Regulatory compliance for airport robots | Specific country/airport authority approvals needed for deployment. | Low (post-MVP) |
