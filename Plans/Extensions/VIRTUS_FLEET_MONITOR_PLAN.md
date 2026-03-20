# Virtus Fleet Monitor — VS Code Extension
### Detailed Development Plan
**Project:** Multi-unit robot fleet management, remote monitoring, OTA updates, and incident timeline — the SaaS component of Virtusco's ARR model  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [Extension Architecture](#2-extension-architecture)
3. [Module 1 — Fleet Map](#3-module-1--fleet-map)
4. [Module 2 — Per-Robot Status Dashboard](#4-module-2--per-robot-status-dashboard)
5. [Module 3 — Remote Log Viewer](#5-module-3--remote-log-viewer)
6. [Module 4 — OTA Update Manager](#6-module-4--ota-update-manager)
7. [Module 5 — Incident Timeline](#7-module-5--incident-timeline)
8. [Module 6 — Airport Operator View](#8-module-6--airport-operator-view)
9. [Backend Architecture](#9-backend-architecture)
10. [Robot-Side Agent](#10-robot-side-agent)
11. [Security Architecture](#11-security-architecture)
12. [Message Protocol](#12-message-protocol)
13. [Windows & Cross-Platform Support](#13-windows--cross-platform-support)
14. [Production Quality Standards](#14-production-quality-standards)
15. [Phased Build Plan](#15-phased-build-plan)
16. [Tech Stack Summary](#16-tech-stack-summary)
17. [Risk Register](#17-risk-register)

---

## 1. Overview & Vision

### Business Context

This extension is fundamentally different from the other 7. While extensions #1–7 are **developer tools**, Fleet Monitor serves two audiences simultaneously:

1. **Virtusco team** — monitor, debug, and update all deployed robots remotely
2. **Airport operators (customers)** — the SaaS component of Virtusco's ₹1.1L ARR per unit

When Virtusco sells a Virtus unit to CIAL or Kempegowda T2, the airport operator gets access to a fleet dashboard. That dashboard is powered by this extension (or a web version of it). This is what the ₹1.1L annual recurring revenue actually pays for — not just software maintenance, but live operational visibility.

### When to Build This

**Do NOT build this until:** First robot is deployed and generating real operational data. Building fleet infrastructure before a real deployment is premature. Target: build Phase 1–3 when first pilot (BIAL or CIAL) is confirmed.

### What Fleet Monitor Provides

```
Fleet Map           → All Virtus units on airport terminal map, live positions
Per-Robot Status    → Battery, current task, error state, uptime, model versions
Remote Log Viewer   → Pull and search ROS 2 logs from any unit via SSH/MQTT
OTA Update Manager  → Push firmware/software updates to selected units
Incident Timeline   → 60s event log before any stop/error event
Airport Operator View → Simplified read-only view for airport staff (non-technical)
```

---

## 2. Extension Architecture

```
virtus-fleet-monitor/
├── package.json
├── tsconfig.json
├── esbuild.config.js
│
├── src/
│   ├── extension.ts
│   ├── panel/
│   │   └── FleetMonitorPanel.ts
│   ├── fleet/
│   │   ├── FleetRegistry.ts              ← Stores robot inventory (id, name, location, SSH config)
│   │   ├── FleetPoller.ts                ← Polls all robots for heartbeat + status
│   │   └── FleetStore.ts                 ← In-memory state for all robots
│   ├── robot/
│   │   ├── RobotConnector.ts             ← SSH connection per robot
│   │   ├── RobotStatusReader.ts          ← Reads status from robot's MQTT or REST API
│   │   ├── LogFetcher.ts                 ← Pulls logs via SSH/SCP
│   │   └── IncidentReader.ts             ← Reads incident log from robot
│   ├── ota/
│   │   ├── OTAPackager.ts                ← Packages firmware/ROS pkg update
│   │   ├── OTADeployer.ts                ← SCP + remote install script
│   │   └── OTARollback.ts                ← Reverts to previous version
│   ├── backend/
│   │   └── FleetBackendClient.ts         ← Optional: connects to Virtusco cloud backend
│   └── platform/
│       └── PlatformUtils.ts
│
├── robot-agent/                          ← Runs on each Virtus RPi 5
│   ├── virtus_agent.py                   ← FastAPI server on robot exposing status API
│   ├── heartbeat.py                      ← Sends heartbeat to fleet backend
│   ├── incident_logger.py                ← Records 60s rolling buffer of events
│   └── install.sh                        ← Installs agent as systemd service
│
└── webview-ui/
    └── src/
        ├── pages/
        │   ├── FleetMapPage.tsx           ← Airport terminal map with robot positions
        │   ├── RobotDetailPage.tsx        ← Single robot deep-dive
        │   ├── LogViewerPage.tsx          ← Remote log search + filter
        │   ├── OTAPage.tsx                ← Update manager
        │   ├── IncidentPage.tsx           ← Incident timeline viewer
        │   └── OperatorPage.tsx           ← Simplified airport operator view
        └── components/
            ├── TerminalMap.tsx            ← SVG airport map with robot overlays
            ├── RobotCard.tsx              ← Status card per robot
            ├── BatteryIndicator.tsx       ← Battery percentage with low alert
            ├── LogLine.tsx                ← Colored log line (INFO/WARN/ERROR)
            ├── OTAProgressBar.tsx         ← Per-robot update progress
            └── IncidentEvent.tsx          ← Timeline event with severity
```

---

## 3. Module 1 — Fleet Map

### Airport Terminal Map

The map is an **SVG representation of the airport terminal** — not a geographic map, but a floor plan. Each Virtusco deployment will have a custom floor plan SVG for their specific terminal.

```typescript
// Fleet map data structure
interface FleetMapConfig {
  terminal_name: string;           // "Kochi International Airport — Terminal"
  svg_path:      string;           // Path to terminal floor plan SVG
  scale_m_per_px: number;          // Meters per SVG pixel for position calculation
  origin:        { x: number; y: number }; // SVG coordinates of (0,0) in robot frame
  zones: {
    id:    string;
    label: string;
    polygon: [number, number][];   // SVG polygon for zone highlight
  }[];
}
```

### Fleet Map UI

```
┌── Fleet Map — Kochi International Airport ───────────────────────────────┐
│  [3 robots online]  [1 robot error]  [Refresh: auto ●]                  │
│                                                                           │
│  ┌── Terminal Floor Plan ─────────────────────────────────────────────┐  │
│  │                                                                     │  │
│  │  Gate A  ║  Gate B  ║  Gate C                                      │  │
│  │          ║          ║                                               │  │
│  │  [V1]→   ║   [V2]   ║        Check-in                              │  │
│  │          ║          ║   ⚠[V3]  ║  Baggage                          │  │
│  │  Security║          ║          ║                                   │  │
│  │                                                                     │  │
│  │  [V1]=Virtus-001  [V2]=Virtus-002  ⚠[V3]=Virtus-003 (ERROR)      │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  Click a robot to open detail view                                       │
└───────────────────────────────────────────────────────────────────────────┘
```

**Robot position:** Published by each robot's ROS 2 navigation stack (`/odom` or `/amcl_pose`) → forwarded by the on-robot agent to the fleet backend. Position updated at 1Hz.

**Robot state color:**
- Green = IDLE or FOLLOWING/NAVIGATING (operational)
- Yellow = HEALTH_CHECK or RECOVERY
- Red = ERROR or offline

---

## 4. Module 2 — Per-Robot Status Dashboard

### Robot Status Schema

```typescript
interface RobotStatus {
  robot_id:       string;          // "virtus-001"
  robot_name:     string;          // "CIAL Porter #1"
  online:         boolean;
  last_heartbeat: number;          // Unix ms

  // Position
  position:       { x: number; y: number; heading_deg: number };
  current_zone:   string;          // "Gate B"

  // State
  fsm_state:      string;          // "NAVIGATING"
  current_task:   string | null;   // "Escorting passenger to Gate C5"
  uptime_s:       number;

  // Health
  battery_pct:    number;
  battery_v:      number;
  cpu_pct:        number;
  ram_pct:        number;
  temp_c:         number;          // RPi CPU temperature
  disk_free_gb:   number;

  // Versions
  ros_pkg_version:   string;       // "0.4.2"
  firmware_version:  string;       // "1.2.0"
  ai_model_version:  string;       // "qwen2.5-1.5b-virtus-v3"
  os_version:        string;

  // Metrics (today)
  tasks_completed: number;
  distance_m:      number;
  errors_today:    number;
  uptime_today_s:  number;
}
```

### Robot Detail UI

```
┌── Virtus-001 — CIAL Porter #1 ───────────────────────────────────────────┐
│  ● Online  Uptime: 06:14:32  State: NAVIGATING                           │
│  Current task: Escorting passenger to Gate C5                            │
│                                                                           │
│  Battery: [████████░░] 78%  11.8V  (est. 4h 20min remaining)            │
│  CPU:     [███░░░░░░░] 34%  Temperature: 52°C                            │
│  RAM:     [████░░░░░░] 42%  Disk free: 12.4 GB                          │
│                                                                           │
│  Today's stats:                                                           │
│  Tasks completed: 14  Distance: 2.3 km  Errors: 1  Uptime: 97.2%       │
│                                                                           │
│  Software versions:                                                       │
│  ROS packages:    0.4.2  ✓ Latest                                        │
│  Firmware:        1.2.0  ✓ Latest                                        │
│  AI model:        virtus-v3  ⚠ v4 available — [Update]                 │
│                                                                           │
│  [View Logs]  [View Incidents]  [Open SSH Terminal]  [Send Command]      │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Module 3 — Remote Log Viewer

### Log Fetching Strategy

```typescript
// robot/LogFetcher.ts
export class LogFetcher {
  // Fetches last N lines from ROS 2 log file on robot via SSH
  async fetchRecent(robotId: string, lines = 500): Promise<LogLine[]> {
    const ssh     = await this.getConnection(robotId);
    const logPath = '~/.ros/log/latest/*.log';  // ROS 2 log location
    const raw     = await ssh.execCommand(`tail -n ${lines} ${logPath}`);
    return this.parseROS2Logs(raw.stdout);
  }

  // Streams live logs via SSH tail -f
  streamLive(robotId: string, onLine: (line: LogLine) => void): void {
    // SSH tail -f — streams in real time
  }

  // Search logs by keyword across all robots
  async searchAll(query: string): Promise<Record<string, LogLine[]>> {
    const results: Record<string, LogLine[]> = {};
    for (const robot of this.fleet.getAllRobots()) {
      const ssh = await this.getConnection(robot.id);
      const raw = await ssh.execCommand(`grep -n "${query}" ~/.ros/log/latest/*.log | tail -100`);
      results[robot.id] = this.parseROS2Logs(raw.stdout);
    }
    return results;
  }
}
```

### Log Viewer UI

```
┌── Remote Logs — Virtus-001 ──────────────────────────────────────────────┐
│  [● Live Stream]  [Search: motor stall        ]  [Filter: ERROR ▼]      │
│                                                                           │
│  14:32:01.234  INFO   orchestrator  State: NAVIGATING → OBSTACLE_AVOIDANCE│
│  14:32:01.891  WARN   lidar_proc    Scan quality degraded: 12% points    │
│  14:32:02.103  INFO   esp32_bridge  Motor cmd sent: L=0.3 R=0.2          │
│  14:32:05.442  ERROR  orchestrator  Stuck detection triggered after 3.4s │
│  14:32:05.443  INFO   orchestrator  State: OBSTACLE_AVOIDANCE → RECOVERY │
│  14:32:08.001  INFO   orchestrator  Recovery: rotating 45° clockwise     │
│  14:32:10.233  INFO   orchestrator  State: RECOVERY → NAVIGATING         │
│                                                                           │
│  [↓ Load older]  Showing: 500 lines  Total today: 24,421                │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Module 4 — OTA Update Manager

### Update Types

Three types of updates, each with different risk and rollback strategy:

| Type | Contents | Risk | Rollback |
|---|---|---|---|
| Firmware | Zephyr `.bin` for ESP32s | Medium | Flash previous `.bin` |
| ROS packages | colcon-built workspace | Medium | Swap `install/` symlink |
| AI model | GGUF or HEF file | Low | Swap model file reference |

### OTA Flow

```
Antony prepares update bundle on laptop
    ↓
OTAPackager.ts: creates versioned .tar.gz
    {version, type, files, checksum, install_script}
    ↓
OTADeployer.ts: SCP bundle to robot → SSH: run install_script
    ↓
install_script:
    1. Backup current version to /opt/virtus/backups/
    2. Extract new files to staging area
    3. Run smoke test (ROS 2 launch, check all nodes alive for 10s)
    4. If smoke test passes: swap symlink, restart services
    5. If smoke test fails: auto-rollback + alert Antony
    ↓
Robot reports back: success / failure + smoke test logs
```

### OTA Manager UI

```
┌── OTA Update Manager ────────────────────────────────────────────────────┐
│  Bundle: ros-packages-v0.4.3.tar.gz  (prepared from workspace)          │
│  Type: ROS Packages  Size: 48 MB  Checksum: sha256:abc123...            │
│                                                                           │
│  Select robots to update:                                                 │
│  [✓] Virtus-001 — CIAL #1     Currently: v0.4.2  → v0.4.3              │
│  [✓] Virtus-002 — CIAL #2     Currently: v0.4.2  → v0.4.3              │
│  [ ] Virtus-003 — CIAL #3     SKIP (has active task — will update later) │
│                                                                           │
│  Update window: [Now] [Schedule: 02:00 AM ▼]                            │
│                                                                           │
│  [▶ Deploy to Selected]                                                  │
│                                                                           │
│  Virtus-001: [████████░░] 80%  Smoke test running...                    │
│  Virtus-002: [██░░░░░░░░] 20%  Transferring...                          │
└───────────────────────────────────────────────────────────────────────────┘
```

**Safety rule:** Never deploy OTA to a robot with an active task (`fsm_state != 'IDLE'`). Either warn and skip, or queue for the next idle window.

---

## 7. Module 5 — Incident Timeline

### Rolling Event Buffer (On Robot)

The `incident_logger.py` agent on each robot maintains a 60-second rolling buffer of all ROS 2 events, hardware telemetry samples, and FSM transitions. When a triggering event occurs (robot stops unexpectedly, enters ERROR state, collision detected), the buffer is **frozen** and written to disk as an incident report.

```python
# robot-agent/incident_logger.py
class IncidentLogger:
    BUFFER_SECONDS = 60
    TRIGGER_STATES = ['ERROR', 'RECOVERY']

    def __init__(self):
        self.buffer = deque(maxlen=600)  # 60s at 10Hz

    def on_event(self, event: dict):
        self.buffer.append({ **event, 't': time.time() })
        if event.get('fsm_state') in self.TRIGGER_STATES:
            self.freeze_incident(event)

    def freeze_incident(self, trigger: dict):
        incident = {
            'trigger':    trigger,
            'timestamp':  time.time(),
            'events':     list(self.buffer),  # Last 60s
        }
        path = f'/opt/virtus/incidents/{int(time.time())}.json'
        with open(path, 'w') as f:
            json.dump(incident, f)
```

### Incident Timeline UI

```
┌── Incident #7 — Virtus-001 — 14:32:05 ──────────────────────────────────┐
│  Trigger: FSM entered ERROR — obstacle_avoidance.stuck                   │
│  Duration of incident analysis: 60 seconds pre-trigger                   │
│                                                                           │
│  T-60s ─────────────────────────────────────────── T+0 (trigger)        │
│                                                                           │
│  T-42s  NAVIGATING → path replanning (goal: Gate C5)                    │
│  T-38s  Lidar scan: obstacle detected at 0.4m ahead                     │
│  T-38s  NAVIGATING → OBSTACLE_AVOIDANCE                                  │
│  T-38s  cmd_vel: linear.x=0.0 (stop)                                    │
│  T-37s  Attempting recovery path: rotate left 30°                       │
│  T-36s  cmd_vel: angular.z=0.3                                           │
│  T-35s  Lidar: obstacle still at 0.4m (not clearing)                    │
│  T-34s  Lidar: obstacle still at 0.4m (2nd check)                       │
│  T-34s  Lidar: obstacle still at 0.4m (3rd check — stuck threshold)     │
│  T-34s  → ERROR state triggered (stuck_timeout=3.4s exceeded)           │
│                                                                           │
│  Hardware at T-0:                                                         │
│  Left motor: 0mA  Right motor: 0mA  Battery: 78%  CPU: 34%             │
│                                                                           │
│  [Export Incident Report]  [Create GitHub Issue]                         │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Module 6 — Airport Operator View

A **simplified, non-technical read-only view** designed for airport operations staff — not Virtusco developers. This is the customer-facing SaaS product.

```
┌── CIAL Porter Fleet — Operations View ───────────────────────────────────┐
│  3 of 4 robots operational  Last updated: 12 seconds ago                │
│                                                                           │
│  VIRTUS-001  ● Assisting passenger to Gate C5   Battery: 78%            │
│  VIRTUS-002  ● Idle at Check-in Zone B          Battery: 92%            │
│  VIRTUS-003  ⚠ Recovery mode — auto-recovering  Battery: 65%            │
│  VIRTUS-004  ✗ Offline — maintenance            —                        │
│                                                                           │
│  Today's service:                                                         │
│  Passengers assisted: 47   Total distance: 8.4 km   Uptime: 94.2%      │
│                                                                           │
│  [Call robot to my location]  (requires passenger app integration)       │
│                                                                           │
│  For technical issues, contact Virtusco: virtusco.tech@gmail.com        │
└───────────────────────────────────────────────────────────────────────────┘
```

This view can also be served as a **web app** (separate from the VS Code extension) for airport operator convenience — the same backend data, a different frontend. The VS Code extension hosts the operator view in a WebviewPanel; the web app version is a separate deployment.

---

## 9. Backend Architecture

For the fleet monitor to work across multiple robots at multiple airports (not just on Antony's laptop), a lightweight cloud backend is needed.

### Backend Stack

```
Virtusco Cloud Backend (minimal, low-cost)
├── FastAPI server (Railway.app or Render.com — free tier to start)
│   ├── POST /heartbeat/{robot_id}     ← Robot sends status every 30s
│   ├── GET  /fleet/{airport_id}       ← Extension polls for all robot statuses
│   ├── GET  /robot/{robot_id}/status  ← Single robot status
│   └── POST /robot/{robot_id}/command ← Send command to robot
│
├── PostgreSQL (Supabase free tier)
│   ├── robots table                   ← Robot inventory
│   ├── heartbeats table               ← Status history
│   └── incidents table                ← Incident reports
│
└── MQTT broker (HiveMQ Cloud free tier)
    ← Real-time robot → backend → extension updates
```

### Robot → Backend Heartbeat

```python
# robot-agent/heartbeat.py
import httpx, asyncio, json

async def heartbeat_loop(robot_id: str, backend_url: str):
    while True:
        status = collect_robot_status()
        await httpx.post(f'{backend_url}/heartbeat/{robot_id}',
                         json=status, timeout=5.0)
        await asyncio.sleep(30)
```

### Extension → Backend Polling

```typescript
// fleet/FleetPoller.ts — polls backend every 10s
async poll(): Promise<void> {
  const response = await fetch(`${BACKEND_URL}/fleet/${airportId}`);
  const robots   = await response.json() as RobotStatus[];
  this.store.updateAll(robots);
  this.panel.postMessage({ type: 'fleetUpdate', payload: robots });
}
```

---

## 10. Robot-Side Agent

The `virtus_agent.py` FastAPI server runs on each RPi 5 as a systemd service. It exposes a local API that the extension can query directly via SSH tunnel (for development) or via the cloud backend (for production):

```
GET  /status         → Full RobotStatus JSON
GET  /logs?lines=500 → Last N log lines
GET  /incidents      → List of incident files
GET  /incidents/{id} → Single incident JSON
POST /command        → Execute a command (stop, navigate_to, etc.)
GET  /versions       → All software version strings
```

---

## 11. Security Architecture

Fleet monitoring involves SSH access to production robots and remote command execution — security is critical.

- **SSH key auth only:** No password SSH. Each deployment has a dedicated deploy key pair. Private key stored in VS Code SecretStorage (encrypted).
- **Command allowlist:** `/command` endpoint only accepts whitelisted commands — no arbitrary shell execution.
- **Backend auth:** JWT tokens per Virtusco team member. Airport operators get read-only tokens.
- **Encrypted heartbeats:** HTTPS only to backend. Robot heartbeats include HMAC signature to prevent spoofing.
- **Audit log:** Every command sent to a robot is logged server-side with timestamp, sender, and robot ID.

---

## 12. Message Protocol

```typescript
type WebviewMessage =
  | { type: 'selectRobot';   payload: { robotId: string } }
  | { type: 'fetchLogs';     payload: { robotId: string; lines: number } }
  | { type: 'streamLogs';    payload: { robotId: string } }
  | { type: 'deployOTA';     payload: { robotIds: string[]; bundlePath: string } }
  | { type: 'fetchIncident'; payload: { robotId: string; incidentId: string } }
  | { type: 'sendCommand';   payload: { robotId: string; cmd: string; params: object } }

type HostMessage =
  | { type: 'fleetUpdate';   payload: RobotStatus[] }
  | { type: 'robotDetail';   payload: RobotStatus }
  | { type: 'logLines';      payload: { robotId: string; lines: LogLine[] } }
  | { type: 'otaProgress';   payload: { robotId: string; pct: number; status: string } }
  | { type: 'incidentData';  payload: IncidentReport }
  | { type: 'commandResult'; payload: { robotId: string; success: boolean; output: string } }
```

---

## 13. Windows & Cross-Platform Support

SSH (node-ssh), HTTPS (fetch), and WebSocket all work natively on Windows. No WSL2 needed. The cloud backend is OS-agnostic. Full Windows support from day one.

---

## 14. Production Quality Standards

- **Graceful degradation:** If backend is unreachable, extension falls back to direct SSH polling. If SSH unreachable, robot shown as "offline."
- **No stale data:** Every robot status display shows "last updated: Xs ago." Anything > 60s old shown as "stale."
- **OTA safety:** Never OTA to a robot that is not IDLE. Never OTA without pre-computing SHA256 checksum. Always auto-rollback if smoke test fails.
- **Operator view security:** Airport operator tokens are read-only. They cannot trigger OTA, view detailed logs, or send commands.
- **GDPR / privacy:** Passenger interaction logs (AI assistant conversations) are never sent to the backend. Only robot operational data.

---

## 15. Phased Build Plan

### Phase 1 — Single Robot Direct SSH Monitor (When pilot confirmed)
- Robot agent (`virtus_agent.py`) installed on first pilot robot
- Extension connects via SSH, shows single robot status + basic metrics
- Log viewer (tail last 500 lines)
- **Deliverable:** Remote visibility into first deployed robot

### Phase 2 — Incident Timeline
- `incident_logger.py` rolling buffer on robot
- Incident freeze + fetch to extension
- Incident timeline UI
- **Deliverable:** Post-incident debugging from VS Code

### Phase 3 — Multi-Robot + Fleet Map
- Fleet registry (multiple robots configured)
- Terminal floor plan SVG integration
- Fleet map with live positions
- **Deliverable:** Fleet overview for first multi-robot deployment

### Phase 4 — Cloud Backend
- Lightweight FastAPI backend on Railway
- Robot heartbeat → backend → extension poll
- Supabase for status history
- **Deliverable:** Fleet accessible without direct SSH to each robot

### Phase 5 — OTA Update Manager
- OTA package format + installer script
- Extension OTA UI with safety checks
- Rollback mechanism
- **Deliverable:** Remote software updates from VS Code

### Phase 6 — Airport Operator View + Web Version
- Simplified operator dashboard
- Read-only JWT tokens for airport staff
- Web app version (separate from VS Code)
- **Deliverable:** Customer-facing SaaS dashboard (the ARR product)

---

## 16. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| SSH | `node-ssh` | Same as other extensions, cross-platform |
| HTTP | Native `fetch` | Backend polling |
| MQTT | `mqtt` npm package | Real-time robot updates |
| Backend | FastAPI on Railway | Cheap, fast to deploy |
| DB | Supabase (PostgreSQL) | Free tier, managed |
| Auth | JWT (python-jose) | Standard, simple |
| Fleet map | SVG + React | Custom terminal floor plans |
| Robot agent | FastAPI + Python | Consistent with AI Studio backend |

---

## 17. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Building fleet monitor before first deployment | Certain (if built too early) | High | Hard gate: do not start until pilot confirmed |
| SSH keys exposed in VS Code workspace | Medium | Critical | Keys in VS Code SecretStorage only — never in workspace files |
| OTA update bricks robot | Low | Critical | Always backup; always smoke test; always auto-rollback |
| Cloud backend costs exceed ARR | Low | Medium | Railway free tier handles 10 robots easily; scale only when needed |
| Airport WiFi blocks MQTT/HTTPS | Medium | High | Fallback to polling over SSH tunnel; standard HTTPS port 443 usually open |
| Robot agent Python process crash | Medium | Medium | Systemd `Restart=always` with 5s backoff |
| Passenger data privacy in logs | High | Critical | AI conversation logs never sent to backend; operational logs only |
