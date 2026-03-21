# Virtus Fleet Backend

The fleet management backend is a Rust server that aggregates telemetry from all deployed Porter robots, serves real-time status to airport operator dashboards, and runs alert rules.

## Tech Stack

| Component | Library | Purpose |
|-----------|---------|---------|
| **Async runtime** | tokio | Concurrent SSH pulls + SSE streams |
| **Web framework** | axum | REST API + Server-Sent Events |
| **Database** | sqlx + SQLite | Compile-time verified SQL queries |
| **SSH client** | russh | Pull logs from deployed robots |
| **Protocol parser** | nom | Binary incident log parsing |
| **Serialization** | serde + serde_json | JSON API responses |
| **Streaming** | tokio-stream | SSE keep-alive streams |

## Architecture

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Porter #1   │  │  Porter #2   │  │  Porter #N   │
│  (Airport A) │  │  (Airport A) │  │  (Airport B) │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │ SSH              │ SSH              │ SSH
       v                  v                  v
┌──────────────────────────────────────────────────┐
│            Virtus Fleet Backend (Rust)            │
│                                                   │
│  Aggregator ──> SQLite ──> Alert Engine           │
│                    │                               │
│                    v                               │
│              Axum REST API + SSE                  │
└──────────────────────┬───────────────────────────┘
                       │ HTTPS
                       v
              Airport Operator Dashboard
```

## Key Features

### Log Aggregation

The aggregator pulls structured JSONL logs from each robot over SSH on a configurable interval (default 30 seconds). Logs are parsed, deduplicated, and stored in SQLite.

### Real-Time SSE Streaming

Airport operator dashboards subscribe to Server-Sent Events for live fleet status:

```
GET /api/fleet/stream
Accept: text/event-stream

data: {"robot_id":"CIAL-001","battery":85,"state":"NAVIGATING","passenger":"PAX-1234"}
data: {"robot_id":"CIAL-002","battery":42,"state":"CHARGING","passenger":null}
```

### Alert Engine

Rules are evaluated against the latest metrics with compile-time SQL verification:

| Alert | Condition | Severity |
|-------|-----------|----------|
| Low battery while serving | battery < 20% AND is_serving | Warning |
| Motor overcurrent | current > 30A for > 5s | Critical |
| Heartbeat timeout | No heartbeat for > 2s | Critical |
| Sensor degradation | sensor_health < 50% | Warning |

### Binary Log Parser

Incident logs from robots use the same binary frame format as the ESP32 protocol. The `nom` parser handles malformed, truncated, and legacy-format frames safely.

## Deployment

The fleet backend runs on a cloud server (not on the robot). It is deployed as a single statically-linked binary.

```bash
cargo build --release
./target/release/virtus-fleet --config fleet.toml
```

!!! info "Migration trigger"
    The fleet backend was originally Python (FastAPI). The Rust migration is triggered when any two of these are true: 5+ robots deployed, 3+ simultaneous SSE streams, or an uptime SLA is requested by a customer.
