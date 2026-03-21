# Virtus Stack -- Language Migration Plan
### When to Rewrite What, Why, and How
**Project:** Targeted language migration plan for the Virtusco/Porter-ROS stack
**Company:** Virtusco | virtusco.in
**Author:** Antony Austin
**Version:** 1.0
**Last Updated:** March 2026

---

## Philosophy

**Don't rewrite anything until there is a measured, concrete reason.**

The three legitimate reasons to rewrite in a lower-level language:

1. **Latency** -- Python's GIL or interpreter overhead is causing missed deadlines on a real-time path
2. **Concurrency** -- Python's async model is producing bugs or stalls under production load
3. **Safety** -- The component processes untrusted binary data at high throughput where memory errors would be serious

If none of these apply, the component stays Python.

---

## Current Stack Language Map

```
Porter-ROS Stack
│
├── ROS 2 Nodes (RPi 5)
│   ├── ydlidar_driver              C++17    -- Already correct
│   ├── porter_lidar_processor      Python   -- MIGRATE to C++ (before Nav2)
│   ├── porter_orchestrator         Python   -- STAY Python (forever)
│   ├── porter_esp32_bridge         C++17    -- PROTECT (do not touch)
│   └── porter_ai_assistant         Python   -- Partial migration (hot path only)
│
├── ESP32 Firmware (Zephyr RTOS)
│   ├── motor_controller            C        -- STAY C (Zephyr requirement)
│   └── sensor_fusion               C        -- STAY C (Zephyr requirement)
│
├── Flutter GUI                     Dart     -- STAY Dart
│
├── Infrastructure / Tools
│   ├── VCMS (virtus-deploy.py)    Python   -- STAY Python
│   └── Fleet Monitor backend      Python   -- MIGRATE to Rust (at 5+ robots)
│
└── Virtus SDK
    └── virtus_api_server           Python   -- MIGRATE to Rust (with Fleet backend)
```

---

## Migration 1 -- Lidar Processor: Python to C++17

**Priority: HIGH -- Do Before Nav2 Integration**

The lidar processor runs a 6-stage filter pipeline at 10 Hz. On an idle RPi 5 this is fast enough, but when Nav2 is running, GIL contention causes stale scan data. At 0.5 m/s, a 40 ms stale scan means the costmap misses an obstacle that appeared 2 cm closer.

**Performance target:** `/scan/processed` published within 2 ms of receiving `/scan`, regardless of Nav2 load.

**Estimated effort:** 4 days

---

## Migration 2 -- Inference Engine Hot Path: Python to C++ Extension

**Priority: MEDIUM -- After TTFT Measurement Shows Problem**

**Trigger:** Do not start until p95 TTFT exceeds 400 ms under production load. If p95 TTFT is under 350 ms, the Python overhead is acceptable.

Only the hot path is migrated via pybind11:

- `IntentClassifier` -- regex + keyword matching
- `ToolDispatcher` -- tool name to handler dispatch
- `ResponseFormatter` -- template-based formatting

The orchestrator, session memory, and multi-turn logic stay Python.

**Estimated effort:** 1 week

---

## Migration 3 -- ESP32 Bridge: Protect Existing C++

**Priority: ONGOING -- Architectural Protection**

The bridge runs a 500 ms heartbeat watchdog. Python's GIL cannot guarantee sub-millisecond timer callback latency under load. A missed heartbeat stops the motors mid-task.

!!! danger "Do not migrate to Python"
    The `esp32_motor_bridge` MUST remain in C++. GIL stalls of 150--300 ms are measurable on a loaded RPi 5 with Nav2 running. A 350 ms stall triggers the ESP32 watchdog and stops the robot.

---

## Migration 4 -- Fleet Backend: Python to Rust

**Priority: LOW -- Trigger at 5+ Deployed Robots**

**Trigger:** Start when any two of these are true:

- More than 5 robots deployed
- More than 3 simultaneous SSE streams
- Aggregator stalls pulling logs from 3+ robots
- Customer requests uptime SLA

**Rust stack:** axum (web), tokio (async), sqlx (compile-time SQL), russh (SSH), nom (binary parsing)

**Migration strategy:** Incremental behind feature flag, not big-bang. Python version stays live until each Rust step is fully validated.

**Estimated effort:** 3 weeks (incremental, across 4 steps)

---

## What Stays Python -- Forever

| Component | Why Python is correct forever |
|-----------|-------------------------------|
| `porter_orchestrator` | FSM at 5 Hz. Correctness and readability matter more than speed. |
| AI training scripts | Offline batch processing. PyTorch/Unsloth have no C++ equivalents. |
| VCMS `virtus-deploy.py` | CLI tool. Runs once per deployment. |
| VOS aggregator (initially) | Pull-based, off-robot, 30-second intervals. |
| Test infrastructure | Tests don't need to be fast. pytest ecosystem is superior for integration testing. |
| AI orchestration logic | Complex multi-turn logic. LLM inference dominates latency regardless. |

---

## Decision Framework

When someone proposes rewriting a component:

1. Is it on the critical path for robot movement or passenger safety? NO -- Stay Python
2. Does it run faster than 10 Hz? NO -- Stay Python
3. Have you MEASURED the latency problem? NO -- Measure first
4. Is the problem caused by the language (GIL) or by algorithm/architecture? Fix root cause first
5. Would a rewrite require maintaining two implementations? Plan parity tests before starting
6. What is the cost of a bug? HIGH -- C++/Rust. LOW -- Stay Python.

---

## Trigger Conditions Summary

| Migration | Do it when | Don't do it until |
|-----------|-----------|-------------------|
| Lidar processor to C++ | Now -- before Nav2 | -- |
| Inference hot path to C++ | p95 TTFT > 400 ms measured | p95 TTFT at most 350 ms |
| ESP32 bridge -- protect C++ | Ongoing | -- |
| Fleet backend to Rust | 5+ robots OR SSE stalls OR SLA requested | All three false |

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Lidar C++ rewrite introduces filter divergence | Medium | High | Run both versions in parallel for 1 hour; assert match |
| C++ intent classifier misses edge case | Medium | Medium | Full parity tests on 200+ utterances; Python fallback for 3 months |
| Heartbeat latency test false negative | Low | High | HIL test on real robot under Nav2 load before release |
| Rust migration stalls mid-way | Medium | Medium | Incremental behind feature flag; Python stays live |
| Team member migrates without measured trigger | Medium | Medium | This document is the reference; PRs require latency measurement |

---

*Update trigger conditions whenever hardware changes or workload profile shifts. A hardware upgrade may invalidate the latency problems that motivated these migrations.*
