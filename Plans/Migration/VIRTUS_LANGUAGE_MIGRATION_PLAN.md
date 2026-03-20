# Virtus Stack — Language Migration Plan
### When to Rewrite What, Why, and How
**Project:** Targeted language migration plan for the Virtusco/Porter-ROS stack — C++, Rust, and what stays Python  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026

---

## Table of Contents

1. [Philosophy](#1-philosophy)
2. [Current Stack Language Map](#2-current-stack-language-map)
3. [Migration 1 — Lidar Processor: Python → C++17](#3-migration-1--lidar-processor-python--c17)
4. [Migration 2 — Inference Engine Hot Path: Python → C++ Extension](#4-migration-2--inference-engine-hot-path-python--c-extension)
5. [Migration 3 — ESP32 Bridge: Protect Existing C++](#5-migration-3--esp32-bridge-protect-existing-c)
6. [Migration 4 — Fleet Backend: Python → Rust](#6-migration-4--fleet-backend-python--rust)
7. [What Stays Python — Forever](#7-what-stays-python--forever)
8. [Decision Framework](#8-decision-framework)
9. [Trigger Conditions Summary](#9-trigger-conditions-summary)
10. [Risk Register](#10-risk-register)

---

## 1. Philosophy

### The Rule

**Don't rewrite anything until there is a measured, concrete reason.**

Rewrites cost time — time that competes with hardware bring-up, investor meetings, Nav2 integration, and everything else on the roadmap. Every component on this list has a specific **trigger condition** that tells you when the rewrite is justified. Until that condition is met, the Python implementation is correct and should not be touched.

The three legitimate reasons to rewrite in a lower-level language:

1. **Latency** — Python's GIL or interpreter overhead is causing missed deadlines on a real-time path
2. **Concurrency** — Python's async model is producing bugs or stalls under production load
3. **Safety** — The component processes untrusted binary data at high throughput where a buffer overflow or memory error would be a serious problem

If none of these apply, the component stays Python.

### Why Not Rewrite Everything in C++ Now?

- Python code with a correct architecture is easier to maintain, test, and hand off to new contributors
- The RPi 5 (Cortex-A76 @ 2.4GHz) is significantly faster than the hardware robotics software was designed for — Python bottlenecks that were real on RPi 3 often don't exist on RPi 5
- Your team's highest-leverage engineering time right now is on features and architecture, not optimization

### Why Not Just Stay Python?

- Specific components on the robot's critical path (LiDAR processing, motor bridge) have real-time requirements that Python's GIL genuinely violates under concurrent load
- The Fleet Monitor backend will handle real-time SSE streams and binary log parsing at scale — Python's async model has known failure modes here
- Some components are safety-critical — a Python crash that stops a robot in front of a passenger at an airport is a business-ending event

---

## 2. Current Stack Language Map

```
Porter-ROS Stack
│
├── ROS 2 Nodes (RPi 5)
│   ├── ydlidar_driver              C++17    ← Already correct
│   ├── porter_lidar_processor      Python   ← MIGRATE to C++ (before Nav2)
│   ├── porter_orchestrator         Python   ← STAY Python (forever)
│   ├── porter_esp32_bridge         C++17    ← PROTECT — do not touch
│   └── porter_ai_assistant         Python   ← Partial migration (hot path only)
│       └── inference_engine.py     Python   ← MIGRATE hot path to C++ extension
│
├── ESP32 Firmware (Zephyr RTOS)
│   ├── motor_controller            C        ← STAY C (Zephyr requirement)
│   └── sensor_fusion               C        ← STAY C (Zephyr requirement)
│
├── Flutter GUI                     Dart     ← STAY Dart (Flutter requirement)
│
├── Infrastructure / Tools
│   ├── VCMS (virtus-deploy.py)    Python   ← STAY Python (CLI, not real-time)
│   ├── VOS aggregator             Python   ← STAY Python (pull-based, off-robot)
│   └── Fleet Monitor backend      Python   ← MIGRATE to Rust (at 5+ robots)
│
└── Virtus SDK
    └── virtus_api_server           Python   ← MIGRATE to Rust (with Fleet backend)
```

---

## 3. Migration 1 — Lidar Processor: Python → C++17

### Priority: HIGH — Do Before Nav2 Integration

### The Problem

`porter_lidar_processor` processes LiDAR scans from `ydlidar_driver` at 10Hz using six composable numpy filter stages:

```
/scan (LaserScan, ~720 points at 10Hz)
    ↓ range_clamp
    ↓ outlier_rejection (MAD-based)
    ↓ median_filter
    ↓ moving_average
    ↓ roi_crop
    ↓ downsample
/scan/processed
```

On an idle RPi 5 this is fast enough. The problem is what happens when Nav2 is running.

Nav2 runs three concurrent workloads on the same RPi 5:
- **Costmap updates** — recomputing local and global costmaps from `/scan/processed`
- **Planner** — running NavFn or Smac to compute a path
- **Controller** — running DWB or Pure Pursuit at 20Hz to compute `/cmd_vel`

These are all Python-unfriendly workloads because they compete for the GIL. When the Nav2 planner grabs the GIL for a path computation (which can take 20–40ms), the lidar processor's timer callback is blocked. The next LiDAR scan arrives but can't be processed. Nav2's costmap update runs on stale data.

At 0.5 m/s in a terminal corridor, a 40ms stale scan means the costmap doesn't know about an obstacle that appeared 2cm closer. This is not a theoretical concern — it is the standard reason robotic lidar processing is written in C++.

### Why the Existing C++ Driver Doesn't Solve This

`ydlidar_driver` is already C++ and publishes `/scan` at the correct rate. But the published ROS 2 message then goes through the Python middleware (rclpy), gets deserialized into a Python `LaserScan` object, passes through six numpy filter stages, and gets republished. Every one of those steps goes through the GIL. The C++ driver only helps at the I/O layer.

### What to Rewrite

The entire `porter_lidar_processor` package. The ROS 2 interface stays identical:
- Subscribe: `/scan` (`sensor_msgs/LaserScan`)
- Publish: `/scan/processed` (`sensor_msgs/LaserScan`)
- Publish: `/diagnostics` (`diagnostic_msgs/DiagnosticArray`)

All six filter stages are stateless array transforms — they map cleanly to C++ with no design changes.

### Filter Translation Reference

```python
# Python (current)
def range_clamp(ranges: np.ndarray, min_r: float, max_r: float) -> np.ndarray:
    clamped = ranges.copy()
    clamped[ranges < min_r] = np.nan
    clamped[ranges > max_r] = np.nan
    return clamped
```

```cpp
// C++17 equivalent
std::vector<float> range_clamp(
    const std::vector<float>& ranges,
    float min_r, float max_r)
{
    std::vector<float> out(ranges.size());
    std::transform(ranges.begin(), ranges.end(), out.begin(),
        [min_r, max_r](float r) {
            return (r < min_r || r > max_r) ? std::numeric_limits<float>::quiet_NaN() : r;
        });
    return out;
}
```

```python
# Python (current) — MAD-based outlier rejection
def outlier_rejection(ranges: np.ndarray, threshold: float = 3.0) -> np.ndarray:
    valid = ranges[~np.isnan(ranges)]
    if len(valid) == 0:
        return ranges
    median = np.median(valid)
    mad = np.median(np.abs(valid - median))
    out = ranges.copy()
    out[np.abs(ranges - median) > threshold * mad] = np.nan
    return out
```

```cpp
// C++17 equivalent
std::vector<float> outlier_rejection(
    const std::vector<float>& ranges,
    float threshold = 3.0f)
{
    // Collect valid (non-NaN) values
    std::vector<float> valid;
    for (float r : ranges) {
        if (!std::isnan(r)) valid.push_back(r);
    }
    if (valid.empty()) return ranges;

    // Compute median
    std::nth_element(valid.begin(), valid.begin() + valid.size()/2, valid.end());
    float median = valid[valid.size() / 2];

    // Compute MAD
    std::vector<float> abs_devs;
    for (float v : valid) abs_devs.push_back(std::abs(v - median));
    std::nth_element(abs_devs.begin(), abs_devs.begin() + abs_devs.size()/2, abs_devs.end());
    float mad = abs_devs[abs_devs.size() / 2];

    // Apply threshold
    std::vector<float> out(ranges);
    for (float& r : out) {
        if (!std::isnan(r) && std::abs(r - median) > threshold * mad)
            r = std::numeric_limits<float>::quiet_NaN();
    }
    return out;
}
```

### New Package Structure

```
porter_lidar_processor/          ← Rewritten in C++17
├── CMakeLists.txt
├── package.xml
├── include/
│   └── porter_lidar_processor/
│       ├── filters.hpp          ← All 6 filter function declarations
│       └── processor_node.hpp   ← Node class declaration
└── src/
    ├── filters.cpp              ← All 6 filter implementations
    ├── processor_node.cpp       ← ROS 2 node — same interface as Python version
    └── main.cpp                 ← Node entry point
```

### Performance Target

After migration, `/scan/processed` should be published within **2ms** of receiving `/scan` regardless of Nav2 load on the same RPi 5. Measure with `ros2 topic delay /scan/processed` before and after.

### Migration Steps

1. Write `filters.hpp` / `filters.cpp` — pure functions, no ROS dependency, full Ztest coverage
2. Write `processor_node.cpp` — same subscriptions/publications as Python version
3. Run both Python and C++ versions simultaneously for 1 hour — verify `/scan/processed` outputs match within floating point tolerance
4. Remove Python version
5. Update CI to build C++ package instead of running Python tests

### Estimated Effort: 4 days

---

## 4. Migration 2 — Inference Engine Hot Path: Python → C++ Extension

### Priority: MEDIUM — After TTFT Measurement Shows Problem

### Trigger Condition

Do not start this migration until you have measured **TTFT (Time to First Token)** in a realistic environment — multiple Nav2 processes running, Hailo-10H loaded, RPi 5 under production load. If TTFT is consistently under 350ms for 95th percentile, the Python overhead is acceptable. If p95 TTFT exceeds 400ms, proceed with this migration.

### The Problem

The inference pipeline for a passenger query is:

```
Voice input
    ↓ STT (Whisper — already fast, runs on Hailo or CPU)
    ↓ intent_classifier.py      ← KEYWORD/REGEX — 5-15ms Python overhead
    ↓ tool_executor.py          ← 14-tool dispatch — 10-25ms Python overhead
    ↓ llama.cpp inference       ← Already C++ — not the problem
    ↓ response_formatter.py     ← String manipulation — 8-15ms Python overhead
    ↓ TTS output
```

The llama.cpp inference itself is already C++ and not the bottleneck. The Python wrapper code around it adds 23–55ms of overhead per request. On an RPi 5 with Nav2 and the LiDAR processor running, GIL contention can push this to 80–120ms on bad ticks.

For a passenger asking "Where is gate C5?", this means they wait an extra 80–120ms for no reason. The intent is trivially classifiable (keyword "gate" + "C5"), the tool lookup is a dictionary lookup, the response is a template fill. This work should take < 5ms.

### What to Migrate

**Do NOT rewrite the full AI assistant.** The orchestrator, session memory, multi-turn logic, tool registration — these are complex and correct in Python. Only extract the hot path:

```
IntentClassifier  →  C++ extension (pybind11)
ToolDispatcher    →  C++ extension (pybind11)  [just the dispatch, not tool impl]
ResponseFormatter →  C++ extension (pybind11)
```

### Architecture: pybind11 Extension Module

```
porter_ai_assistant/
├── src/
│   ├── python/                    ← Existing Python (unchanged)
│   │   ├── orchestrator.py
│   │   ├── tool_executor.py       ← Tools still implemented in Python
│   │   └── inference_engine.py   ← Calls C++ hot path via import
│   └── cpp/                       ← NEW: C++ extension module
│       ├── CMakeLists.txt
│       ├── intent_classifier.cpp  ← Regex + keyword matching in C++
│       ├── tool_dispatcher.cpp    ← Tool name → handler dispatch (string switch)
│       ├── response_formatter.cpp ← Template-based response formatting
│       └── bindings.cpp           ← pybind11 Python bindings
└── tests/
    ├── test_classifier_parity.py  ← Python vs C++ output must match exactly
    └── test_classifier_latency.py ← Benchmark: must be < 5ms per call
```

### IntentClassifier in C++

```cpp
// cpp/intent_classifier.cpp

#include <string>
#include <regex>
#include <unordered_map>

struct IntentResult {
    std::string intent;      // "NAVIGATE", "INFO_QUERY", "FOLLOW", "STOP", etc.
    std::string destination; // Extracted destination if intent is NAVIGATE
    float confidence;        // 0.0–1.0
};

static const std::vector<std::pair<std::regex, std::string>> INTENT_PATTERNS = {
    { std::regex(R"(gate\s+([A-Z]\d+))",    std::regex::icase), "NAVIGATE" },
    { std::regex(R"(check.?in)",             std::regex::icase), "NAVIGATE" },
    { std::regex(R"(follow\s+me)",           std::regex::icase), "FOLLOW"   },
    { std::regex(R"(\bstop\b|\bwait\b)",     std::regex::icase), "STOP"     },
    { std::regex(R"(weigh|weight|heavy)",    std::regex::icase), "WEIGH"    },
    { std::regex(R"(help|assist|staff)",     std::regex::icase), "ASSIST"   },
    { std::regex(R"(where|how|when|what)",   std::regex::icase), "INFO"     },
};

IntentResult classify_intent(const std::string& text) {
    for (const auto& [pattern, intent] : INTENT_PATTERNS) {
        std::smatch match;
        if (std::regex_search(text, match, pattern)) {
            IntentResult result;
            result.intent     = intent;
            result.confidence = 0.9f;
            if (intent == "NAVIGATE" && match.size() > 1) {
                result.destination = match[1].str();
            }
            return result;
        }
    }
    // No match — fall through to LLM
    return { "UNKNOWN", "", 0.0f };
}
```

```cpp
// cpp/bindings.cpp — pybind11 interface
#include <pybind11/pybind11.h>
#include "intent_classifier.cpp"

namespace py = pybind11;

PYBIND11_MODULE(virtus_ai_core, m) {
    py::class_<IntentResult>(m, "IntentResult")
        .def_readwrite("intent",      &IntentResult::intent)
        .def_readwrite("destination", &IntentResult::destination)
        .def_readwrite("confidence",  &IntentResult::confidence);

    m.def("classify_intent", &classify_intent,
          "Classify passenger utterance into a structured intent");
}
```

```python
# inference_engine.py — updated to use C++ hot path
try:
    from virtus_ai_core import classify_intent  # C++ fast path
    USING_CPP_CLASSIFIER = True
except ImportError:
    from .intent_classifier_py import classify_intent  # Python fallback
    USING_CPP_CLASSIFIER = False

def process_query(self, text: str) -> str:
    # C++ classifier runs in < 1ms vs 10-15ms Python equivalent
    intent = classify_intent(text)

    if intent.confidence > 0.85 and intent.intent != "UNKNOWN":
        # High-confidence intent — skip LLM, direct tool dispatch
        return self._fast_path_response(intent)
    else:
        # Low confidence — full LLM inference
        return self._llm_inference(text)
```

### Parity Testing

The migration is only complete when Python and C++ classifiers produce identical output on the full test set:

```python
# tests/test_classifier_parity.py
import pytest
from porter_ai_assistant.intent_classifier_py import classify_intent as py_classify
from virtus_ai_core import classify_intent as cpp_classify

TEST_UTTERANCES = [
    ("Where is gate C5?",            "NAVIGATE",    "C5"),
    ("Follow me please",             "FOLLOW",      ""),
    ("Stop here",                    "STOP",        ""),
    ("How heavy is my bag?",         "WEIGH",       ""),
    ("I need help",                  "ASSIST",      ""),
    ("What time does the flight leave", "INFO",     ""),
    ("asdfgh gibberish xyz",         "UNKNOWN",     ""),
]

@pytest.mark.parametrize("utterance,expected_intent,expected_dest", TEST_UTTERANCES)
def test_cpp_matches_python(utterance, expected_intent, expected_dest):
    py_result  = py_classify(utterance)
    cpp_result = cpp_classify(utterance)
    assert py_result.intent == cpp_result.intent, \
        f"Intent mismatch for '{utterance}': Python={py_result.intent}, C++={cpp_result.intent}"
```

### Estimated Effort: 1 week (including parity tests)

---

## 5. Migration 3 — ESP32 Bridge: Protect Existing C++

### Priority: ONGOING — Architectural Protection, Not a Migration

### The Situation

`porter_esp32_bridge` is already C++17 and is correct. This section exists to document **why it must stay C++** — because there will be pressure at some point to simplify it to Python for maintainability, and that pressure must be resisted.

### Why It Cannot Be Python

The bridge runs a 500ms heartbeat watchdog. The ESP32 motor controller's Zephyr watchdog expects a heartbeat frame every 500ms. If it doesn't receive one — for any reason — it stops the motors. This is a deliberate safety design.

Python's GIL means that if any other Python thread grabs the GIL for more than ~100ms (not uncommon during inference, costmap computation, or even garbage collection), the heartbeat timer callback is delayed. On a loaded RPi 5 with Nav2 running, GIL stalls of 150–300ms are measurable. A 350ms GIL stall delays the heartbeat. The ESP32 watchdog fires. The robot stops. In a corridor. With a passenger's luggage on it.

The C++ bridge guarantees sub-millisecond timer callback latency regardless of what Python is doing on the same machine, because it runs in its own OS thread outside the GIL entirely.

### What to Protect

Add a comment at the top of `esp32_motor_bridge.cpp` and in the `README.md`:

```cpp
// ARCHITECTURAL NOTE — DO NOT MIGRATE TO PYTHON
//
// This node MUST remain in C++. The 500ms heartbeat watchdog on the ESP32
// motor controller requires sub-millisecond timer callback latency.
// Python's GIL cannot guarantee this under load (Nav2 + inference concurrent).
// A missed heartbeat stops the motors mid-task.
//
// If you are considering changing this to Python for any reason,
// read LANGUAGE_MIGRATION_PLAN.md section 5 first.
```

### Add a Latency Regression Test

```cpp
// test_bridge_heartbeat_latency.cpp
// Verifies heartbeat is sent within 50ms of scheduled time even under CPU load

TEST(HeartbeatLatency, WithCpuLoad) {
    // Spin up CPU-intensive work in background threads
    // Measure actual heartbeat send times over 30 seconds
    // Assert max jitter < 50ms (well within 500ms watchdog window)
}
```

This test runs in CI. If it starts failing — which would mean something changed in the bridge that introduced latency — the build breaks before it reaches a robot.

---

## 6. Migration 4 — Fleet Backend: Python → Rust

### Priority: LOW — Trigger at 5+ Deployed Robots

### Trigger Condition

Start this migration when **any two of these are true**:
- More than 5 robots are deployed across airports
- A single Fleet Monitor backend instance is handling more than 3 simultaneous SSE streams
- The aggregator is pulling logs from more than 3 robots concurrently and experiencing stalls
- You have a paying customer who has asked about uptime SLA (because you now need one)

Until these conditions are met, FastAPI/Python is the correct choice — it's fast to iterate on, well-understood, and sufficient for the current scale.

### Why Rust Specifically (Not C++)

Three specific properties of Rust make it the right choice for this workload over C++:

**1. Memory safety under async concurrency without GC**
The Fleet Monitor backend is fundamentally an async concurrent system — it's pulling data from multiple robots simultaneously over SSH, serving SSE streams to multiple airport operator dashboards, parsing binary incident logs, and running alert rules on a schedule. Python's asyncio has a known failure mode: mixing CPU-bound work (log parsing, schema validation) with I/O-bound work (SSE streaming, SSH) causes async event loop stalls. C++ with async (Boost.Asio or similar) is safe but requires careful manual memory management across async boundaries — this is where C++ concurrency bugs live. Rust's ownership model makes the borrow checker enforce memory safety at compile time, with no runtime cost.

**2. Binary protocol parsing safety**
The VOS aggregator parses binary incident log frames pulled from deployed robots. These frames come from real hardware in the field — potentially corrupted, truncated, or from older firmware versions. A C++ parser that encounters an unexpected frame can silently read out-of-bounds memory. A Rust parser with the `nom` parsing library makes malformed input handling explicit at the type level — a parse error is a `Result::Err`, not undefined behavior.

**3. Compile-time SQL verification**
The `sqlx` crate verifies SQL queries against the actual database schema at compile time. This eliminates an entire class of bugs — type mismatches between Rust structs and SQLite columns, wrong column names, missing fields — that currently only appear at runtime in the Python/SQLite version. As the schema grows organically over 6 months of production use, this becomes increasingly valuable.

### What to Rewrite

```
Python (current)                  →  Rust (migration target)
────────────────────────────────────────────────────────────
off_robot/aggregator.py           →  src/aggregator/ (tokio async SSH pull)
off_robot/alert_rules.py          →  src/alerts/ (alert engine)
server/virtus_api_server/         →  src/api/ (axum REST + SSE)
virtus_sdk/client.py (server side)→  src/sdk_server/ (axum routes)
```

### Rust Stack

```toml
# Cargo.toml
[dependencies]
tokio    = { version = "1", features = ["full"] }  # Async runtime
axum     = "0.7"                                    # Web framework (replaces FastAPI)
sqlx     = { version = "0.7", features = ["sqlite", "runtime-tokio"] }
serde    = { version = "1", features = ["derive"] } # JSON serialization
serde_json = "1"
russh    = "0.40"                                   # SSH client (replaces paramiko)
nom      = "7"                                      # Binary protocol parser
tokio-stream = "0.1"                               # SSE streaming
```

### Architecture

```rust
// src/api/routes/status.rs — Axum SSE endpoint (replaces FastAPI streaming)

use axum::{
    extract::State,
    response::sse::{Event, Sse},
};
use tokio_stream::StreamExt;

pub async fn stream_fleet_status(
    State(state): State<AppState>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = state.fleet_store
        .subscribe_all()  // Receives updates from all robots via channels
        .map(|status| {
            Ok(Event::default()
                .data(serde_json::to_string(&status).unwrap()))
        });

    Sse::new(stream).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(Duration::from_secs(15))
    )
}
```

```rust
// src/aggregator/ssh_puller.rs — async SSH log pull (replaces paramiko)

use russh::client;
use tokio::time::{interval, Duration};

pub async fn pull_robot_logs(robot: &RobotConfig) -> Result<Vec<LogEntry>> {
    let session = connect_ssh(&robot.ssh_host, &robot.ssh_user, &robot.key_path).await?;
    let output  = session.exec(&format!(
        "tail -n 500 /opt/virtus/logs/{}.jsonl", today_date()
    )).await?;

    output.lines()
        .filter_map(|line| serde_json::from_str::<LogEntry>(line).ok())
        .collect::<Vec<_>>()
        .into()
}
```

```rust
// src/alerts/engine.rs — Alert rule evaluation

pub struct AlertEngine {
    db: SqlitePool,
}

impl AlertEngine {
    pub async fn evaluate_all(&self) -> Vec<Alert> {
        let mut alerts = vec![];

        // sqlx: compile-time verified SQL
        let low_battery_robots = sqlx::query_as!(
            MetricsRow,
            "SELECT robot_id, battery_pct FROM metrics
             WHERE battery_pct < 20 AND is_serving = 1
               AND ts > datetime('now', '-2 minutes')"
        )
        .fetch_all(&self.db)
        .await
        .unwrap_or_default();

        for row in low_battery_robots {
            alerts.push(Alert {
                severity: Severity::Warn,
                robot_id: row.robot_id,
                message:  format!("Battery {}% while serving passenger", row.battery_pct),
            });
        }

        alerts
    }
}
```

### Migration Strategy

Do not do a big-bang rewrite. Migrate incrementally behind a feature flag:

**Step 1:** Rewrite the aggregator SSH puller in Rust. The Python FastAPI server still runs — it just calls the Rust binary for log pulling.

**Step 2:** Rewrite the alert engine in Rust. Python still handles the REST API.

**Step 3:** Rewrite the FastAPI routes in Axum. The Rust binary now serves the full API.

**Step 4:** Remove the Python server entirely.

At every step, both implementations run in parallel and output is compared before the Python version is removed.

### Estimated Effort: 3 weeks (incremental, across 4 steps)

---

## 7. What Stays Python — Forever

These components have no business case for language migration. Explicitly documented to prevent future over-engineering.

| Component | Why Python is correct forever |
|---|---|
| `porter_orchestrator` | FSM logic runs at 5Hz. Latency-insensitive. Correctness and readability matter more than speed. A bug in the FSM is dangerous — Python's expressiveness reduces bug surface. |
| AI training scripts | Offline batch processing. PyTorch, Unsloth, TRL have no C++ equivalents. Not on any real-time path. |
| VCMS `virtus-deploy.py` | CLI tool. Runs once per deployment. No latency requirement. |
| VOS aggregator (initially) | Pull-based, runs off-robot, 30-second intervals. Python is sufficient until 5+ robots. |
| Test infrastructure | Tests don't need to be fast. Python's pytest ecosystem is significantly better than C++ test frameworks for integration testing. |
| AI assistant orchestration logic | Complex multi-turn logic, session memory, tool composition. Correctness is critical. 100–300ms response time is acceptable. The LLM inference dominates the latency budget regardless. |
| VCMS config validation | Schema validation + business rules. Runs offline. |
| Fleet Monitor UI (VS Code extension) | TypeScript. Non-negotiable — VS Code extensions are TypeScript. |

---

## 8. Decision Framework

When someone proposes rewriting a component in C++/Rust, apply this checklist before starting:

```
1. Is it on the critical path for robot movement or passenger safety?
   NO → Stay Python

2. Does it run faster than 10Hz (i.e., is it a real-time component)?
   NO → Stay Python

3. Have you MEASURED the latency problem in a realistic environment?
   NO → Measure first, rewrite second

4. Is the latency problem caused by the language (GIL, interpreter overhead)?
   Or by algorithm, I/O, or architecture?
   → Fix the root cause first. A bad algorithm in C++ is still slow.

5. Would a C++ rewrite require maintaining two implementations during migration?
   YES → Plan the parity test before starting, not after.

6. What is the cost of a bug in this component?
   HIGH (motors stop, robot crashes) → C++ (or Rust for async/safety concerns)
   LOW (log parsing, config tools) → Stay Python
```

---

## 9. Trigger Conditions Summary

| Migration | Do it when | Don't do it until |
|---|---|---|
| Lidar processor → C++ | **Now — before Nav2** | — |
| Inference hot path → C++ extension | p95 TTFT > 400ms measured in production load | p95 TTFT ≤ 350ms |
| ESP32 bridge → protect C++ | Ongoing — add comment + latency regression test | — |
| Fleet backend → Rust | 5+ robots deployed, OR SSE stalls measured, OR uptime SLA requested | All three false |

---

## 10. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Lidar processor C++ rewrite introduces filter output divergence | Medium | High | Run Python and C++ versions in parallel for 1 hour; assert outputs match within float tolerance |
| C++ intent classifier misses an edge case Python caught | Medium | Medium | Full parity test suite on 200+ utterances; Python fallback remains in code for 3 months post-migration |
| ESP32 bridge heartbeat latency test false negative (passes in CI, fails on real RPi under load) | Low | High | Run HIL heartbeat latency test on real robot under Nav2 load before each release |
| Rust fleet backend migration stalls mid-way, leaving two half-working backends | Medium | Medium | Incremental migration behind feature flag; Python version stays live until each Rust step is fully validated |
| Team member migrates a Python component to C++ without a measured trigger | Medium | Medium | This document is the reference; migration PRs require a latency measurement in the PR description |
| pybind11 Python/C++ boundary introduces marshalling overhead that negates the gain | Low | Low | Benchmark the boundary overhead before committing to migration; IntentResult struct is trivially copyable so marshalling should be < 0.1ms |

---

*Update the trigger conditions in Section 9 whenever the robot's hardware changes (RPi 6, faster SoC) or the workload profile changes (faster inference, more concurrent Nav2 threads). A hardware upgrade may invalidate the latency problems that motivated some of these migrations.*
