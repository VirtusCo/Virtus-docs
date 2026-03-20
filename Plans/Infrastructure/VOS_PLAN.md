# Virtus Observability Stack (VOS)
### Detailed Development Plan
**Project:** Structured logging, metrics pipeline, alerting, and operational dashboards for deployed Virtus robots  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026  
**Build Priority:** #5 — Build before first paid deployment

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [Three Pillars of Observability](#2-three-pillars-of-observability)
3. [On-Robot Data Collection](#3-on-robot-data-collection)
4. [Structured Logging](#4-structured-logging)
5. [Metrics Pipeline](#5-metrics-pipeline)
6. [Event Journal](#6-event-journal)
7. [Off-Robot Aggregation](#7-off-robot-aggregation)
8. [Alerting System](#8-alerting-system)
9. [Operational Dashboard](#9-operational-dashboard)
10. [Data Retention & Privacy](#10-data-retention--privacy)
11. [Integration with Fleet Monitor](#11-integration-with-fleet-monitor)
12. [Phased Build Plan](#12-phased-build-plan)
13. [Tech Stack Summary](#13-tech-stack-summary)
14. [Risk Register](#14-risk-register)

---

## 1. Overview & Vision

### The Production Visibility Problem

During development, you can `ros2 topic echo` anything, SSH in anytime, and physically watch the robot. In production at an airport, you have none of that. When a robot stops at 14:32 on a Tuesday, you need to answer these questions without touching the robot:

- What was the FSM state for the 60 seconds before it stopped?
- What were the motor currents?
- Was there a sensor fault?
- Did the LiDAR see the obstacle or miss it?
- What did the AI assistant say to the passenger right before?

Without VOS, you are flying blind in production.

### What VOS Provides

```
On-robot:
  Structured JSON logs       → Every ROS 2 log event as queryable JSON
  Metrics emitter            → System vitals at 1Hz: CPU, RAM, battery, motors
  Event journal              → Append-only FSM transitions, errors, tool calls

Off-robot:
  Log aggregator             → Pulls structured logs from all robots
  Metrics store              → Timeseries database (SQLite → InfluxDB path)
  Alert rules engine         → "If battery < 20% AND robot is serving → alert"
  Telegram/email alerts      → Antony notified immediately on critical events
```

### What VOS Is Not

VOS is **not a passenger data system**. AI assistant conversations are **never** sent off-robot. VOS collects operational robot data only: hardware telemetry, FSM states, ROS 2 errors, navigation events.

---

## 2. Three Pillars of Observability

Classic observability has three pillars — Virtus needs all three:

| Pillar | What it answers | VOS component |
|---|---|---|
| **Logs** | What happened? | Structured JSON logs via ROS 2 logging hooks |
| **Metrics** | How is the system performing over time? | 1Hz metrics emitter → timeseries store |
| **Traces** | Why did a specific request take this path? | Event journal — FSM transitions with causal links |

---

## 3. On-Robot Data Collection

### Architecture on RPi 5

```
ROS 2 nodes (all packages)
    ↓ ROS 2 log messages (RCLPY logging API)
    ↓
virtus_log_bridge (new ROS 2 node)
    ├── Intercepts all /rosout messages
    ├── Enriches with: robot_id, node_name, severity, timestamp
    ├── Writes to: /opt/virtus/logs/structured.jsonl (rotation daily)
    └── Publishes: /observability/logs (for real-time streaming)

virtus_metrics_emitter (new ROS 2 node)
    ├── Subscribes to: /hardware/power, /hardware/motors, /sensor_fusion
    ├── Reads: psutil (CPU, RAM, disk, temp)
    ├── Publishes: /observability/metrics at 1Hz
    └── Writes to: /opt/virtus/metrics/metrics.jsonl

virtus_event_journal (new ROS 2 node)
    ├── Subscribes to: /orchestrator/state, /diagnostics, /ai_assistant/command
    ├── Writes: append-only /opt/virtus/events/journal.jsonl
    └── Pre-incident buffer: last 60s frozen on ERROR/FATAL
```

---

## 4. Structured Logging

### Current State: Unstructured

```
[INFO] [1742318422.841] [orchestrator]: State changed to NAVIGATING
[WARN] [1742318425.100] [lidar_processor]: Scan quality degraded: 12%
```

No robot ID, no structured fields, not queryable, hard to filter.

### Target State: Structured JSON

```json
{"ts":"2026-03-18T14:32:02.841Z","robot_id":"virtus-cial-001","node":"/orchestrator","level":"INFO","event":"fsm_transition","from":"IDLE","to":"NAVIGATING","trigger":"gate_requested","session_id":"pax_abc123"}
{"ts":"2026-03-18T14:32:05.100Z","robot_id":"virtus-cial-001","node":"/lidar_processor","level":"WARN","event":"scan_quality","quality_pct":12,"cause":"reflective_surface","affected_range_m":[1.2,2.4]}
```

Every log line is a complete, self-describing JSON object. Queryable with `jq`, importable into any database, correlatable by `session_id` and `robot_id`.

### Log Bridge Node

```python
# porter_observability/log_bridge.py

class VirtusLogBridge(Node):
    """
    Subscribes to /rosout, enriches log records to structured JSON,
    writes to rolling daily log file.
    """

    def __init__(self):
        super().__init__('virtus_log_bridge')
        self.robot_id = self.get_parameter('robot_id').value
        self.log_dir  = Path('/opt/virtus/logs')
        self.log_dir.mkdir(parents=True, exist_ok=True)

        self.sub = self.create_subscription(
            rcl_interfaces.msg.Log, '/rosout', self._on_log, 100
        )
        self._open_log_file()
        # Rotate log file at midnight
        self.create_timer(60.0, self._check_rotation)

    def _on_log(self, msg: rcl_interfaces.msg.Log):
        entry = {
            'ts':       datetime.utcfromtimestamp(
                            msg.stamp.sec + msg.stamp.nanosec * 1e-9
                        ).isoformat() + 'Z',
            'robot_id': self.robot_id,
            'node':     msg.name,
            'level':    LEVEL_NAMES.get(msg.level, 'UNKNOWN'),
            'msg':      msg.msg,
            'file':     msg.file,
            'line':     msg.line,
        }

        # Attempt structured field extraction from known log patterns
        entry.update(self._extract_structured_fields(msg.name, msg.msg))

        self.log_file.write(json.dumps(entry) + '\n')
        self.log_file.flush()

    def _extract_structured_fields(self, node: str, msg: str) -> dict:
        """
        Parses known log message patterns to extract structured fields.
        E.g. "State changed: IDLE -> NAVIGATING [gate_requested]"
        → {"event": "fsm_transition", "from": "IDLE", "to": "NAVIGATING", "trigger": "gate_requested"}
        """
        patterns = [
            (r'State changed: (\w+) -> (\w+) \[(.+)\]',
             lambda m: {'event': 'fsm_transition', 'from': m[1], 'to': m[2], 'trigger': m[3]}),
            (r'Scan quality degraded: (\d+)%',
             lambda m: {'event': 'scan_quality', 'quality_pct': int(m[1])}),
            (r'Motor overcurrent: (\w+) (\d+)mA',
             lambda m: {'event': 'motor_overcurrent', 'motor': m[1], 'current_ma': int(m[2])}),
        ]
        for pattern, extractor in patterns:
            match = re.search(pattern, msg)
            if match:
                return extractor(match.groups())
        return {}
```

---

## 5. Metrics Pipeline

### Metrics Emitter Node

```python
# porter_observability/metrics_emitter.py

class VirtusMetricsEmitter(Node):

    def __init__(self):
        super().__init__('virtus_metrics_emitter')
        self.robot_id = self.get_parameter('robot_id').value

        # Subscribe to hardware topics
        self.create_subscription(PowerTelemetry, '/hardware/power',  self._on_power,  10)
        self.create_subscription(MotorTelemetry, '/hardware/motors', self._on_motors, 10)
        self.create_subscription(SensorFusion,   '/sensor_fusion',   self._on_sensor, 10)
        self.create_subscription(OrchestratorState, '/orchestrator/state', self._on_state, 10)

        # Publish metrics at 1Hz
        self.metrics_pub = self.create_publisher(String, '/observability/metrics', 10)
        self.create_timer(1.0, self._emit)

        self._latest = {}   # Latest values from each source topic

    def _emit(self):
        metrics = {
            'ts':       datetime.utcnow().isoformat() + 'Z',
            'robot_id': self.robot_id,
            # System
            'cpu_pct':      psutil.cpu_percent(),
            'ram_pct':      psutil.virtual_memory().percent,
            'disk_free_gb': psutil.disk_usage('/').free / 1e9,
            'cpu_temp_c':   self._get_cpu_temp(),
            # Power (from latest PowerTelemetry message)
            'v12':          self._latest.get('v12', None),
            'v5':           self._latest.get('v5', None),
            'battery_pct':  self._latest.get('battery_pct', None),
            'i12_ma':       self._latest.get('i12_ma', None),
            # Motors
            'left_ma':      self._latest.get('left_ma', None),
            'right_ma':     self._latest.get('right_ma', None),
            'left_temp_c':  self._latest.get('left_temp_c', None),
            # Robot state
            'fsm_state':    self._latest.get('fsm_state', None),
            'is_serving':   self._latest.get('fsm_state') in [
                                'PASSENGER_DETECTED', 'FOLLOWING', 'NAVIGATING'],
        }

        # Write to local metrics store
        with open(f'/opt/virtus/metrics/{datetime.utcnow().date()}.jsonl', 'a') as f:
            f.write(json.dumps(metrics) + '\n')

        # Publish for Fleet Monitor real-time streaming
        self.metrics_pub.publish(String(data=json.dumps(metrics)))
```

### Metrics Schema

Every metrics record has exactly the same fields — null for unavailable. This makes aggregation predictable.

---

## 6. Event Journal

The event journal records **every significant state change** with enough context to reconstruct what happened. Unlike logs (which record everything), the journal records only events that matter for incident analysis.

```python
# porter_observability/event_journal.py

class VirtusEventJournal(Node):
    """
    Append-only event log of all significant robot events.
    Maintains a 60-second pre-incident rolling buffer.
    Freezes the buffer to disk on ERROR/FATAL events.
    """

    SIGNIFICANT_EVENTS = [
        '/orchestrator/state',      # Every FSM transition
        '/diagnostics',             # Every WARN/ERROR diagnostic
        '/ai_assistant/command',    # Every passenger command parsed
        '/cmd_vel',                 # Every non-zero velocity command (sampled at 1Hz)
    ]

    INCIDENT_TRIGGER_STATES = ['STATE_ERROR', 'STATE_RECOVERY']

    def __init__(self):
        super().__init__('virtus_event_journal')
        self.robot_id     = self.get_parameter('robot_id').value
        self.pre_incident = deque(maxlen=600)   # 60s at 10Hz
        self.journal_file = open('/opt/virtus/events/journal.jsonl', 'a')

        self.create_subscription(OrchestratorState, '/orchestrator/state', self._on_state, 10)
        self.create_subscription(DiagnosticArray,   '/diagnostics',         self._on_diag,  10)

    def _on_state(self, msg: OrchestratorState):
        event = {
            'ts':        self._ts(),
            'robot_id':  self.robot_id,
            'event':     'fsm_transition',
            'state':     msg.state,
            'trigger':   msg.last_trigger,
            'duration_ms': msg.state_duration_ms,
            'task':      msg.current_task,
        }
        self._record(event)

        if msg.state in self.INCIDENT_TRIGGER_STATES:
            self._freeze_incident(event)

    def _freeze_incident(self, trigger_event: dict):
        """Write the 60-second pre-incident buffer to a timestamped file"""
        incident = {
            'trigger':   trigger_event,
            'robot_id':  self.robot_id,
            'frozen_at': self._ts(),
            'events':    list(self.pre_incident),
        }
        path = f'/opt/virtus/incidents/{int(time.time())}.json'
        with open(path, 'w') as f:
            json.dump(incident, f, indent=2)
        self.get_logger().warn(f'Incident frozen: {path}')

    def _record(self, event: dict):
        self.pre_incident.append(event)
        self.journal_file.write(json.dumps(event) + '\n')
        self.journal_file.flush()
```

---

## 7. Off-Robot Aggregation

### Simple Pull Architecture (Phase 1)

```python
# off_robot/aggregator.py — runs on dev laptop / cloud server

class VirtusAggregator:
    """
    Pulls logs, metrics, and events from all registered robots via SSH.
    Stores locally in SQLite for querying.
    """

    def collect_all(self):
        for robot in self.registry.all_robots():
            ssh = paramiko.SSHClient()
            ssh.connect(robot.ssh_host, username='pi', key_filename=robot.key_path)

            # Pull today's log file
            sftp = ssh.open_sftp()
            sftp.get(
                f'/opt/virtus/logs/{datetime.utcnow().date()}.jsonl',
                f'data/logs/{robot.id}/{datetime.utcnow().date()}.jsonl'
            )
            # Similarly for metrics and events
            self._import_to_db(robot.id)
```

### SQLite Schema (Phase 1)

```sql
CREATE TABLE logs (
    ts          TEXT NOT NULL,
    robot_id    TEXT NOT NULL,
    node        TEXT,
    level       TEXT,
    event       TEXT,
    msg         TEXT,
    extra_json  TEXT,           -- Additional structured fields as JSON
    INDEX idx_robot_ts (robot_id, ts)
);

CREATE TABLE metrics (
    ts          TEXT NOT NULL,
    robot_id    TEXT NOT NULL,
    cpu_pct     REAL,
    ram_pct     REAL,
    v12         REAL,
    battery_pct REAL,
    left_ma     REAL,
    right_ma    REAL,
    fsm_state   INTEGER,
    INDEX idx_robot_ts (robot_id, ts)
);

CREATE TABLE incidents (
    id          INTEGER PRIMARY KEY,
    ts          TEXT NOT NULL,
    robot_id    TEXT NOT NULL,
    trigger_event TEXT,         -- JSON of the triggering event
    events_json   TEXT,         -- JSON array of 60s pre-incident events
);
```

### Query Examples

```python
# "What was the FSM state of cial-001 for the hour before the incident?"
import sqlite3, json
conn = sqlite3.connect('data/virtus_observability.db')
rows = conn.execute("""
    SELECT ts, event, extra_json FROM logs
    WHERE robot_id = 'virtus-cial-001'
      AND event = 'fsm_transition'
      AND ts BETWEEN '2026-03-18T13:32:00Z' AND '2026-03-18T14:32:00Z'
    ORDER BY ts
""").fetchall()
for ts, event, extra in rows:
    print(ts, json.loads(extra).get('from'), '→', json.loads(extra).get('to'))
```

---

## 8. Alerting System

### Alert Rules

```python
# off_robot/alert_rules.py

ALERT_RULES = [
    AlertRule(
        name    = 'low_battery_serving',
        query   = "SELECT * FROM metrics WHERE battery_pct < 20 AND is_serving = 1",
        message = lambda r: f"⚠️ {r['robot_id']}: Battery {r['battery_pct']}% while serving passenger",
        severity = 'WARN',
        cooldown_s = 300,           # Don't spam — once per 5 minutes
    ),
    AlertRule(
        name    = 'motor_overcurrent',
        query   = "SELECT * FROM logs WHERE event = 'motor_overcurrent' AND level = 'ERROR'",
        message = lambda r: f"🔴 {r['robot_id']}: Motor overcurrent {r.get('current_ma')}mA — possible stall",
        severity = 'CRITICAL',
        cooldown_s = 60,
    ),
    AlertRule(
        name    = 'robot_offline',
        query   = """SELECT robot_id FROM metrics
                     WHERE ts < datetime('now', '-5 minutes')
                     GROUP BY robot_id HAVING MAX(ts) < datetime('now', '-5 minutes')""",
        message = lambda r: f"🔴 {r['robot_id']}: No heartbeat for >5 minutes",
        severity = 'CRITICAL',
        cooldown_s = 600,
    ),
    AlertRule(
        name    = 'repeated_recovery',
        query   = """SELECT robot_id, COUNT(*) as n FROM logs
                     WHERE event = 'fsm_transition' AND json_extract(extra_json,'$.to') = 'STATE_RECOVERY'
                       AND ts > datetime('now', '-1 hour')
                     GROUP BY robot_id HAVING n >= 3""",
        message = lambda r: f"⚠️ {r['robot_id']}: Entered recovery {r['n']} times in last hour",
        severity = 'WARN',
        cooldown_s = 1800,
    ),
]
```

### Alert Delivery

```python
# off_robot/alert_delivery.py

class TelegramAlerter:
    """Sends alerts to Virtusco team Telegram group"""
    def send(self, rule: AlertRule, data: dict):
        msg = rule.message(data)
        requests.post(
            f'https://api.telegram.org/bot{BOT_TOKEN}/sendMessage',
            json={'chat_id': TEAM_CHAT_ID, 'text': msg, 'parse_mode': 'HTML'}
        )

class EmailAlerter:
    """Fallback email for CRITICAL alerts"""
    def send(self, rule: AlertRule, data: dict):
        # smtplib to virtusco.tech@gmail.com
        pass
```

Alert rules run every 60 seconds. Telegram chosen because the whole team already uses it.

---

## 9. Operational Dashboard

A lightweight web dashboard (not VS Code — this is for operational monitoring, potentially shareable with airport operators) showing live robot status:

```
Virtusco Operations Dashboard — CIAL
───────────────────────────────────────────────────────
Robot         State        Battery  Serving   Errors/hr
───────────────────────────────────────────────────────
virtus-cial-001  NAVIGATING   78%     YES       0
virtus-cial-002  IDLE         92%     NO        0
virtus-cial-003  RECOVERY     65%     NO        2  ⚠
───────────────────────────────────────────────────────
Active alerts:
⚠ virtus-cial-003: Entered recovery 2x in last hour
```

Built as a simple FastAPI + Jinja2 web app. Not the Fleet Monitor VS Code extension — this is a lightweight ops view accessible from any browser, including on a tablet at the airport operations desk.

---

## 10. Data Retention & Privacy

### What IS Collected (Operational Data Only)

- FSM state transitions with timestamps
- Hardware telemetry (power, motors, sensors)
- ROS 2 error and warning logs
- Navigation events (obstacle detected, recovery triggered)
- System metrics (CPU, RAM, temperature)

### What Is NEVER Collected

- Passenger conversation content (AI assistant logs stay on-robot, never forwarded)
- Passenger identifiers (session IDs are random UUIDs — no PII)
- Video or audio (no camera on Virtus currently)
- Location of specific individuals

### Retention Policy

```
On-robot logs:    7 days rolling (automatic rotation)
Aggregated logs:  30 days
Metrics:          90 days
Incidents:        1 year (important for warranty/liability)
```

---

## 11. Integration with Fleet Monitor

VOS is the **data layer** that the Fleet Monitor VS Code extension and web dashboard read from. The Fleet Monitor doesn't collect data — it reads from VOS:

```
VOS (on-robot agents + aggregator + SQLite)
    ↑ pulls from robots
    ↓ serves to
Fleet Monitor (VS Code extension) — reads via REST API
Ops Dashboard (web app) — reads via REST API
Alert engine — reads directly from SQLite
```

---

## 12. Phased Build Plan

### Phase 1 — Structured Logging (1 week)
- `virtus_log_bridge` ROS 2 node
- Structured JSON log file per robot per day
- Manual SSH pull to dev laptop
- `jq` query examples documented
- **Deliverable:** Every ROS 2 log event is a queryable JSON record

### Phase 2 — Metrics Emitter (3 days)
- `virtus_metrics_emitter` ROS 2 node
- 1Hz metrics file per robot per day
- SQLite import script
- **Deliverable:** System vitals queryable historically

### Phase 3 — Event Journal + Incident Freeze (3 days)
- `virtus_event_journal` ROS 2 node with 60s rolling buffer
- Incident freeze on ERROR/RECOVERY state
- **Deliverable:** Pre-incident context always captured

### Phase 4 — Aggregator + SQLite (3 days)
- `aggregator.py` SSH pull from all robots
- SQLite schema + import
- Query examples for common incident analysis
- **Deliverable:** All robot data queryable from one place

### Phase 5 — Alerting (3 days)
- 4 core alert rules (battery, overcurrent, offline, repeated recovery)
- Telegram delivery
- Cooldown logic
- **Deliverable:** Team alerted on critical events without checking manually

### Phase 6 — Ops Dashboard (3 days)
- FastAPI + Jinja2 web dashboard
- Fleet status table
- Active alerts panel
- **Deliverable:** Airport operator-accessible status view

---

## 13. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| Log format | JSON Lines (JSONL) | Appendable, no lock, `jq`-queryable |
| Metrics format | JSON Lines at 1Hz | Consistent with logs |
| Local storage | SQLite | Zero infrastructure, sufficient for 10 robots |
| Future storage | InfluxDB | Clear migration path when SQLite becomes limiting |
| SSH pull | paramiko | Already used in Fleet Monitor |
| Alerting | Telegram Bot API | Team already uses Telegram |
| Alert rules | Pure Python + SQLite queries | No dependencies, testable |
| Web dashboard | FastAPI + Jinja2 | Lightweight, no frontend framework needed |
| Pattern extraction | Python `re` | Simple regex on known log patterns |

---

## 14. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Log files fill RPi SD card | Medium | High | Daily rotation, 7-day TTL, disk free alert at <2GB |
| Passenger conversation data accidentally logged | Low | Critical | Explicit exclusion in log_bridge: filter `/ai_assistant/response` and `/ai_assistant/query` |
| SSH pull fails → stale data | Medium | Medium | Alert if last pull > 2 hours old |
| Alert spam on transient faults | High | Low | Cooldown periods per rule; `WARN` vs `CRITICAL` severity thresholds |
| SQLite write lock under concurrent pulls | Low | Low | WAL mode enabled; pull serialized per robot |
| JSONL corruption from incomplete write | Low | Medium | Each line is atomic (Python `write` + `flush`); incomplete lines skipped on import |
