# Virtus Hardware Dashboard — VS Code Extension
### Detailed Development Plan
**Project:** Live hardware telemetry dashboard for Virtus robot — power rails, motor current, sensor readings, relay states, alert system  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [Extension Architecture](#2-extension-architecture)
3. [Module 1 — Telemetry Receiver](#3-module-1--telemetry-receiver)
4. [Module 2 — Power Rail Monitor](#4-module-2--power-rail-monitor)
5. [Module 3 — Motor Driver Health](#5-module-3--motor-driver-health)
6. [Module 4 — Sensor Readings Panel](#6-module-4--sensor-readings-panel)
7. [Module 5 — Alert System](#7-module-5--alert-system)
8. [Module 6 — Schematic Cross-Reference](#8-module-6--schematic-cross-reference)
9. [Module 7 — Power Event Log](#9-module-7--power-event-log)
10. [Telemetry Protocol](#10-telemetry-protocol)
11. [Firmware Integration Requirements](#11-firmware-integration-requirements)
12. [Windows & Cross-Platform Support](#12-windows--cross-platform-support)
13. [Production Quality Standards](#13-production-quality-standards)
14. [Phased Build Plan](#14-phased-build-plan)
15. [Tech Stack Summary](#15-tech-stack-summary)
16. [Risk Register](#16-risk-register)

---

## 1. Overview & Vision

### The Problem

Danush (hardware/PCB lead) currently has **no live view of the robot's electrical state** during firmware development and integration testing. He physically probes with a multimeter while running tests — which means:
- He can't see motor current draw while the robot is moving
- He can't detect undervoltage events that happen and recover in milliseconds
- He can't correlate software events (motor command sent) with hardware responses (current spike) in a time-aligned view
- When a BTS7960 H-bridge trips its thermal protection, there's no log of what current was flowing before it happened

### What Virtus Hardware Dashboard Provides

A VS Code extension that receives live hardware telemetry from the RPi 5 (which aggregates data from ESP32 #1 and #2 via USB CDC) and renders it as a real-time dashboard:

```
Power Rails        → 12V, 5V, 3.3V — voltage + current, undervoltage alerts
Motor Drivers      → Left/right BTS7960 — RPWM/LPWM duty%, current mA, EN state, temp estimate
Sensor Readings    → ToF mm, Ultrasonic cm, Microwave bool, Kalman estimate cm
Relay States       → Which relays are active, how long
Alert System       → Threshold-based alerts with VS Code notifications
Power Event Log    → Timestamped log of voltage/current anomalies
Schematic XRef     → Click a reading → jump to relevant firmware line
```

### Who Uses It

**Primary:** Danush — during hardware bring-up, PCB testing, motor driver integration
**Secondary:** Antony — during software integration testing to correlate firmware events with hardware state

---

## 2. Extension Architecture

```
virtus-hardware-dashboard/
├── package.json
├── tsconfig.json
├── esbuild.config.js
│
├── src/
│   ├── extension.ts                        ← Registers commands, creates panel
│   ├── panel/
│   │   └── HardwareDashboardPanel.ts       ← WebviewPanel lifecycle
│   ├── telemetry/
│   │   ├── TelemetryReceiver.ts            ← USB CDC serial reader (node-serialport)
│   │   ├── TelemetryDecoder.ts             ← JSON or binary telemetry parser
│   │   ├── TelemetryStore.ts               ← Ring buffer: last 60s of all readings
│   │   └── TelemetryStreamer.ts            ← Forwards to webview via postMessage
│   ├── alerts/
│   │   ├── AlertEngine.ts                  ← Threshold evaluation engine
│   │   ├── AlertConfig.ts                  ← Default thresholds for Virtus hardware
│   │   └── AlertLogger.ts                  ← Writes alert events to log file
│   ├── schematic/
│   │   └── SchematicIndex.ts               ← Maps telemetry field → firmware file:line
│   └── platform/
│       └── PlatformUtils.ts                ← Serial port path resolution per OS
│
├── python/
│   └── telemetry_bridge.py                 ← Optional: serial → WebSocket bridge for WSL2
│
└── webview-ui/
    └── src/
        ├── App.tsx
        ├── pages/
        │   ├── OverviewPage.tsx            ← All panels at a glance
        │   ├── PowerPage.tsx               ← Detailed power rail view
        │   ├── MotorPage.tsx               ← Motor driver deep dive
        │   ├── SensorPage.tsx              ← Sensor readings + Kalman state
        │   ├── AlertsPage.tsx              ← Alert history + threshold config
        │   └── EventLogPage.tsx            ← Full power event timeline
        └── components/
            ├── VoltageGauge.tsx            ← Needle gauge for voltage rails
            ├── CurrentBar.tsx              ← Horizontal bar with threshold line
            ├── DutyChart.tsx               ← PWM duty cycle live sparkline
            ├── RelayIndicator.tsx          ← On/off pill with duration
            ├── AlertBadge.tsx              ← Color-coded severity badge
            └── TimeSeriesChart.tsx         ← Rolling 30s chart (Recharts)
```

---

## 3. Module 1 — Telemetry Receiver

### Data Path

```
ESP32 #1 (motor + power data)  ─┐
ESP32 #2 (sensor fusion data)  ─┤→ RPi 5 USB CDC → USB cable → Dev laptop
Power Management Board          ─┘   (aggregated by a telemetry ROS 2 node)
```

The RPi runs a lightweight **telemetry aggregator node** that collects data from both ESP32s and the power management board, then streams it as JSON over a second USB CDC serial port (or via the existing `porter_esp32_bridge` with a new message type).

### Telemetry Packet Format (JSON over Serial)

JSON chosen over binary for this channel — easier to debug, human-readable, and telemetry is low-Hz (10Hz):

```json
{
  "t": 1742318422841,
  "power": {
    "v12":    11.94,
    "v5":      4.97,
    "v33":     3.31,
    "i12_ma": 2340,
    "i5_ma":   890,
    "relay_states": [true, false, true, false]
  },
  "motors": {
    "left":  { "rpwm": 45, "lpwm": 0,  "en": true,  "i_ma": 1240 },
    "right": { "rpwm": 45, "lpwm": 0,  "en": true,  "i_ma": 1180 }
  },
  "sensors": {
    "tof_mm":    342,
    "sonic_cm":  34,
    "microwave": false,
    "kalman_cm": 34.2
  },
  "esp32_health": {
    "esp1_uptime_ms": 142000,
    "esp2_uptime_ms": 142100,
    "esp1_errors": 0,
    "esp2_errors": 0
  }
}
```

### TelemetryReceiver.ts

```typescript
import SerialPort from 'serialport';
import { ReadlineParser } from '@serialport/parser-readline';

export class TelemetryReceiver extends EventEmitter {
  private port: SerialPort | null = null;

  async connect(portPath: string, baud = 115200): Promise<void> {
    this.port = new SerialPort({ path: portPath, baudRate: baud });
    const parser = this.port.pipe(new ReadlineParser({ delimiter: '\n' }));
    parser.on('data', (line: string) => {
      try {
        const packet = JSON.parse(line) as TelemetryPacket;
        this.emit('packet', packet);
      } catch {
        // Malformed line — skip silently, log to debug
      }
    });
  }

  async disconnect(): Promise<void> {
    this.port?.close();
  }

  // Auto-detect: scans serial ports for one that sends valid telemetry
  static async autoDetect(): Promise<string | null> {
    const ports = await SerialPort.list();
    for (const p of ports) {
      if (p.manufacturer?.includes('Raspberry') || p.productId === '...') {
        return p.path;
      }
    }
    return null;
  }
}
```

### Ring Buffer Store

```typescript
// telemetry/TelemetryStore.ts
const HISTORY_SECONDS = 60;
const SAMPLE_HZ = 10;
const MAX_SAMPLES = HISTORY_SECONDS * SAMPLE_HZ; // 600 samples

export class TelemetryStore {
  private buffers: Map<string, number[]> = new Map();

  push(packet: TelemetryPacket): void {
    this.pushField('v12',     packet.power.v12);
    this.pushField('v5',      packet.power.v5);
    this.pushField('v33',     packet.power.v33);
    this.pushField('i12_ma',  packet.power.i12_ma);
    this.pushField('left_i',  packet.motors.left.i_ma);
    this.pushField('right_i', packet.motors.right.i_ma);
    // ... all fields
  }

  private pushField(key: string, value: number): void {
    if (!this.buffers.has(key)) this.buffers.set(key, []);
    const buf = this.buffers.get(key)!;
    buf.push(value);
    if (buf.length > MAX_SAMPLES) buf.shift();
  }

  getHistory(key: string): number[] {
    return this.buffers.get(key) ?? [];
  }

  getLatest(key: string): number | null {
    const buf = this.buffers.get(key);
    return buf?.length ? buf[buf.length - 1] : null;
  }
}
```

---

## 4. Module 2 — Power Rail Monitor

### Hardware Context

The Virtus power management board (designed by Danush) has:
- **12V rail** — main input from battery/PSU, powers BTS7960 motors
- **5V rail** — LM7805 regulator, powers RPi 5
- **3.3V rail** — AMS1117-3.3, powers ESP32s and sensors
- **Relay switching** — Arduino Nano controlled relay bank

### Power Rail UI

```
┌── Power Rails ───────────────────────────────────────────────────────────┐
│                                                                           │
│  12V Rail    [████████████░░] 11.94V   ↑ 2340mA  ● OK                  │
│              Min: 10.5V  Nominal: 12.0V  Max seen: 12.6V                │
│                                                                           │
│  5V Rail     [██████████████] 4.97V    ↑ 890mA   ● OK                  │
│              Min: 4.75V  Nominal: 5.0V                                   │
│                                                                           │
│  3.3V Rail   [██████████████] 3.31V    ↑ 340mA   ● OK                  │
│              Min: 3.1V   Nominal: 3.3V                                   │
│                                                                           │
│  Relays:  [K1: ON ● 00:02:14]  [K2: OFF ○]  [K3: ON ● 00:02:14]  [K4: OFF ○]  │
│                                                                           │
│  30s History ─────────────────────────────────────────────────────────── │
│  12V: ─────────────────────────────────── (stable)                       │
│  5V:  ─────────────────────────────────── (stable)                       │
└───────────────────────────────────────────────────────────────────────────┘
```

### Voltage Thresholds

```typescript
// alerts/AlertConfig.ts
export const POWER_THRESHOLDS = {
  v12: { warn: 11.5,  critical: 11.0,  max: 13.0 },  // BTS7960 needs ≥ 10V
  v5:  { warn: 4.75,  critical: 4.6,   max: 5.5  },  // RPi 5 min USB-C spec
  v33: { warn: 3.1,   critical: 3.0,   max: 3.6  },  // ESP32 absolute min
};

export const CURRENT_THRESHOLDS = {
  i12_ma: { warn: 8000,  critical: 10000 },  // BTS7960 per-side: 43A capable, PCB traces rated ~10A
  i5_ma:  { warn: 3000,  critical: 4000  },  // RPi 5 max ~5A
};
```

---

## 5. Module 3 — Motor Driver Health

### BTS7960 Context

The Virtus robot uses 2× BTS7960 dual H-bridge drivers (one per motor side). Each BTS7960 has:
- RPWM / LPWM PWM inputs (direction + speed)
- R_EN / L_EN enable pins
- IS pin (current sense — analog output proportional to motor current)
- Thermal protection that trips and disables the driver at ~150°C

### Motor Driver UI

```
┌── Motor Drivers ─────────────────────────────────────────────────────────┐
│                                                                           │
│  LEFT MOTOR                          RIGHT MOTOR                         │
│  EN: ● ON                            EN: ● ON                            │
│  Direction: FORWARD                  Direction: FORWARD                   │
│  RPWM: 45%  ████████░░░░░░░          RPWM: 45%  ████████░░░░░░░          │
│  LPWM:  0%  ░░░░░░░░░░░░░░░          LPWM:  0%  ░░░░░░░░░░░░░░░          │
│  Current:  1240 mA  ██░░░░░░░░       Current:  1180 mA  ██░░░░░░░░       │
│  Temp est: 42°C  ● COOL              Temp est: 41°C  ● COOL              │
│                                                                           │
│  Current History (30s) ──────────────────────────────────────────────── │
│  Left:   ▁▁▂▂▄▄▅▅▅▅▄▄▃▃▂▂▁▁ (ramp up, stable, ramp down)               │
│  Right:  ▁▁▂▂▄▄▅▅▅▅▄▄▃▃▂▂▁▁                                             │
└───────────────────────────────────────────────────────────────────────────┘
```

### Temperature Estimation

The BTS7960 doesn't expose die temperature directly. Estimate it from current and duty cycle using a simple thermal model:

```typescript
// Simplified thermal model — calibrate against real measurements
function estimateTemp(current_ma: number, duty_pct: number, ambient_c = 30): number {
  const power_w = (current_ma / 1000) * (duty_pct / 100) * 12; // P = I * V * duty
  const rth_jc  = 1.5;  // BTS7960 thermal resistance junction-to-case (°C/W)
  return ambient_c + power_w * rth_jc;
}
```

### Motor Alert Thresholds

```typescript
export const MOTOR_THRESHOLDS = {
  current_warn_ma:     3000,   // Sustained >3A → check for binding/stall
  current_critical_ma: 6000,   // >6A → stall condition, kill enable
  temp_warn_c:         80,
  temp_critical_c:     120,    // BTS7960 trips at ~150°C, warn well before
};
```

---

## 6. Module 4 — Sensor Readings Panel

```
┌── Sensors ───────────────────────────────────────────────────────────────┐
│                                                                           │
│  ToF (VL53L0X)        342 mm     ██████░░░░░░░░  Range: 20–4000mm       │
│  Ultrasonic (HC-SR04)  34 cm     ██████░░░░░░░░  Range: 2–400cm         │
│  Microwave (RCWL)     false      ○ No motion                             │
│                                                                           │
│  Kalman Estimate       34.2 cm   ██████░░░░░░░░                          │
│  Kalman Variance        0.043    (low = confident)                       │
│  Obstacle Class:       CLEAR                                             │
│                                                                           │
│  Sensor Agreement: ✓ ToF and Ultrasonic within 5% — Kalman healthy      │
│                                                                           │
│  ESP32 #2 Health:  Uptime: 02:22:14  Errors: 0  ● OK                   │
└───────────────────────────────────────────────────────────────────────────┘
```

**Sensor agreement check:** If ToF and Ultrasonic disagree by > 20%, show a warning — likely one sensor is dirty/faulty or there's a reflective surface confusing ToF.

---

## 7. Module 5 — Alert System

### Alert Engine

```typescript
// alerts/AlertEngine.ts
export class AlertEngine {
  private config: AlertConfig;
  private cooldowns: Map<string, number> = new Map(); // prevent alert spam

  evaluate(packet: TelemetryPacket): Alert[] {
    const alerts: Alert[] = [];
    const now = Date.now();

    // Power rail checks
    if (packet.power.v12 < POWER_THRESHOLDS.v12.critical) {
      alerts.push(this.makeAlert('CRITICAL', '12V_UNDERVOLTAGE',
        `12V rail at ${packet.power.v12}V — below critical threshold ${POWER_THRESHOLDS.v12.critical}V`));
    }

    // Motor current checks
    if (packet.motors.left.i_ma > MOTOR_THRESHOLDS.current_critical_ma) {
      alerts.push(this.makeAlert('CRITICAL', 'LEFT_MOTOR_OVERCURRENT',
        `Left motor drawing ${packet.motors.left.i_ma}mA — possible stall. EN will be cut.`));
    }

    // Sensor disagreement
    const tof_cm  = packet.sensors.tof_mm / 10;
    const sonic   = packet.sensors.sonic_cm;
    if (Math.abs(tof_cm - sonic) / sonic > 0.2) {
      alerts.push(this.makeAlert('WARN', 'SENSOR_DISAGREEMENT',
        `ToF (${tof_cm.toFixed(1)}cm) and Ultrasonic (${sonic}cm) disagree by >20%`));
    }

    return alerts.filter(a => this.checkCooldown(a.id, now, 5000)); // 5s cooldown
  }
}
```

### Alert Severity Levels

| Level | Color | VS Code Notification | Sound | Auto-action |
|---|---|---|---|---|
| INFO | Blue | No | No | None |
| WARN | Yellow | Yes (once) | No | Log to file |
| CRITICAL | Red | Yes (persistent) | Yes | Log + optional auto-stop |
| FATAL | Red flashing | Yes (modal) | Yes | Force stop motors |

### Alert Panel UI

```
┌── Active Alerts ─────────────────────────────────────────────────────────┐
│  ● CRITICAL  14:33:42  Left motor overcurrent: 6240mA (limit: 6000mA)  │
│  ⚠ WARN      14:33:01  Sensor disagreement: ToF 34cm vs Sonic 28cm     │
│  ● INFO      14:31:22  12V rail: 11.6V (approaching warn threshold)     │
└───────────────────────────────────────────────────────────────────────────┘

Threshold Configuration:
  12V warn:      [11.5] V    12V critical:  [11.0] V
  Motor current: [3000] mA warn  [6000] mA critical
  [Save Thresholds]  [Reset to Defaults]
```

---

## 8. Module 6 — Schematic Cross-Reference

### Concept

Click any telemetry reading → the extension opens the relevant firmware file at the relevant line. This bridges Danush's hardware world and Antony's software world.

### Cross-Reference Map

```typescript
// schematic/SchematicIndex.ts
export const SCHEMATIC_XREF: Record<string, XRef> = {
  'motors.left.i_ma': {
    firmware_file: 'esp32_firmware/motor_controller/src/bts7960.c',
    firmware_line: 42,
    note: 'IS pin ADC read — left motor current sense'
  },
  'power.v12': {
    firmware_file: 'esp32_firmware/motor_controller/src/power_monitor.c',
    firmware_line: 18,
    note: 'ADC channel 0 — 12V rail via voltage divider'
  },
  'sensors.tof_mm': {
    firmware_file: 'esp32_firmware/sensor_fusion/src/vl53l0x.c',
    firmware_line: 67,
    note: 'VL53L0X I2C read — ranging result register'
  },
};
```

When a user clicks "View in firmware" on any reading, VS Code opens the file at that line via:
```typescript
vscode.workspace.openTextDocument(uri).then(doc =>
  vscode.window.showTextDocument(doc, { selection: new vscode.Range(line, 0, line, 0) })
);
```

---

## 9. Module 7 — Power Event Log

Every threshold crossing, relay state change, and alert is written to a structured log:

```
.virtus-hw/
└── power-events.jsonl
```

```json
{"t":1742318422841,"type":"UNDERVOLTAGE","rail":"v12","value":11.3,"threshold":11.5,"severity":"WARN"}
{"t":1742318425100,"type":"RELAY_CHANGE","relay":2,"state":true,"duration_prev_ms":0}
{"t":1742318430200,"type":"OVERCURRENT","motor":"left","value_ma":6240,"threshold_ma":6000,"severity":"CRITICAL"}
{"t":1742318430250,"type":"AUTO_STOP","motor":"left","reason":"OVERCURRENT"}
```

**Event Log UI:** Filterable timeline with severity filter, time range picker, and export to CSV for Danush to include in PCB revision reports.

---

## 10. Telemetry Protocol

```typescript
// Webview → Host
type WebviewMessage =
  | { type: 'connect';         payload: { port: string; baud: number } }
  | { type: 'disconnect';      payload: void }
  | { type: 'autoDetect';      payload: void }
  | { type: 'setThresholds';   payload: AlertConfig }
  | { type: 'clearAlerts';     payload: void }
  | { type: 'openInFirmware';  payload: { field: string } }
  | { type: 'exportEventLog';  payload: { format: 'csv' | 'jsonl' } }

// Host → Webview
type HostMessage =
  | { type: 'telemetry';       payload: TelemetryPacket }
  | { type: 'alert';           payload: Alert }
  | { type: 'connectionStatus';payload: { connected: boolean; port: string } }
  | { type: 'portList';        payload: string[] }
  | { type: 'historySnapshot'; payload: Record<string, number[]> }
```

---

## 11. Firmware Integration Requirements

The extension requires a **telemetry aggregator** to be added to the RPi 5 ROS 2 stack. This is a new lightweight node:

```python
# porter_telemetry/telemetry_node.py
# Subscribes to: /esp32_bridge/rx, /sensor_fusion, /power_monitor
# Publishes to: /dev/ttyUSB_telemetry (second USB CDC) as JSON @ 10Hz

class TelemetryAggregatorNode(Node):
    def __init__(self):
        super().__init__('telemetry_aggregator')
        self.bridge_sub   = self.create_subscription(BridgeFrame, '/esp32_bridge/rx', self.on_bridge, 10)
        self.sensor_sub   = self.create_subscription(SensorFusion, '/sensor_fusion', self.on_sensor, 10)
        self.serial_port  = serial.Serial('/dev/ttyUSB_telemetry', 115200)
        self.timer        = self.create_timer(0.1, self.publish_telemetry)  # 10Hz

    def publish_telemetry(self):
        packet = self.build_packet()
        self.serial_port.write((json.dumps(packet) + '\n').encode())
```

This is a small addition to the existing firmware — roughly 80 lines of Python.

---

## 12. Windows & Cross-Platform Support

| Feature | Linux | Windows |
|---|---|---|
| Serial port (node-serialport) | `/dev/ttyUSB*` | `COM*` ports |
| Port auto-detection | VID/PID match | VID/PID match (same logic) |
| Serial reading | ✅ Native | ✅ Native (Windows COM ports) |
| Schematic file open | ✅ | ✅ |
| Event log write | ✅ | ✅ |

Serial port naming is abstracted:

```typescript
// platform/PlatformUtils.ts
static defaultTelemetryPort(): string {
  return Platform.isWindows ? 'COM5' : '/dev/ttyUSB2';
}
```

Windows users can use the hardware dashboard natively — no WSL2 needed, since USB serial works natively on Windows via COM ports.

---

## 13. Production Quality Standards

- **No data loss:** Ring buffer never blocks the serial read thread — overflow drops oldest samples silently
- **Graceful disconnect:** Serial port loss (cable pulled) detected within 2s; reconnect button shown; no crash
- **Threshold persistence:** Alert thresholds saved to `.virtus-hw/thresholds.json` — survive VS Code restart
- **FATAL auto-stop:** On FATAL alert (>6A motor current), extension sends a stop command via the ROS 2 bridge to `/cmd_vel` (if ROS 2 Studio is also running and connected)
- **Log rotation:** `power-events.jsonl` rotates at 10MB — archive to `power-events-{date}.jsonl`

---

## 14. Phased Build Plan

### Phase 1 — Scaffold + Serial Connection (Week 1)
- Serial port list + connect/disconnect UI
- Auto-detect RPi telemetry port
- JSON packet parsing
- **Deliverable:** Panel shows raw telemetry JSON

### Phase 2 — Power Rail Monitor (Week 2)
- Voltage + current gauges
- 30s history charts
- Relay state indicators
- **Deliverable:** Live power rail dashboard

### Phase 3 — Motor Driver Health (Week 3)
- RPWM/LPWM duty bars
- Current bars with thresholds
- Temperature estimation
- **Deliverable:** Live motor health view

### Phase 4 — Alert System (Week 4)
- `AlertEngine.ts` with all threshold checks
- VS Code notification integration
- Alert panel with threshold config UI
- **Deliverable:** Working alert system with configurable thresholds

### Phase 5 — Sensor Panel + Event Log (Week 5)
- Sensor readings panel with agreement check
- `power-events.jsonl` logging
- Event log UI with filter + export
- **Deliverable:** Full sensor view + searchable event history

### Phase 6 — Schematic Cross-Reference + Polish (Week 6)
- `SchematicIndex.ts` with all field mappings
- "View in firmware" click handler
- Windows COM port support
- **Deliverable:** Production-ready extension

---

## 15. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| Serial port | `serialport` (npm) | Cross-platform, maintained, Node.js native |
| Telemetry format | JSON Lines over serial | Human-readable, easy to debug, low overhead at 10Hz |
| Charts | Recharts | Consistent with other extensions |
| Gauges | Custom SVG React components | Needle gauges not in Recharts |
| Alert engine | Pure TypeScript | No dependencies, fully testable |
| Log format | JSONL | Appendable, queryable, git-diff friendly |

---

## 16. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Telemetry aggregator node not yet built | Certain (it doesn't exist yet) | High | Build it in Phase 1 — small node, ~80 lines |
| Serial port enumeration differs per OS | High | Medium | `PlatformUtils.defaultTelemetryPort()` per OS; manual override always available |
| 10Hz JSON over serial causes buffer overflow | Low | Medium | Each JSON packet ~200 bytes; 115200 baud handles 1150 bytes/s easily |
| BTS7960 temperature model inaccurate | Medium | Low | Label as "estimate"; calibrate against thermocouple measurements; never used for safety decisions alone |
| Relay states not exposed in current firmware | Medium | High | Add relay state to ESP32 #1 telemetry report in firmware; small addition |
| Auto-stop via ROS 2 creates unsafe dependency | Medium | High | Auto-stop is opt-in, disabled by default; requires ROS 2 Studio connected |
