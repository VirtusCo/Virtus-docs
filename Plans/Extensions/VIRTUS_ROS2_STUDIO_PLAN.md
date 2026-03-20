# Virtus ROS 2 Studio — VS Code Extension
### Detailed Development Plan
**Project:** Virtusco-specific ROS 2 development environment — topic monitoring, node graph, FSM viewer, ESP32 bridge debugger, launch file builder  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [Extension Architecture](#2-extension-architecture)
3. [Module 1 — Topic Monitor](#3-module-1--topic-monitor)
4. [Module 2 — Node Graph Visualizer](#4-module-2--node-graph-visualizer)
5. [Module 3 — FSM State Viewer](#5-module-3--fsm-state-viewer)
6. [Module 4 — ESP32 Bridge Debugger](#6-module-4--esp32-bridge-debugger)
7. [Module 5 — Launch File Builder](#7-module-5--launch-file-builder)
8. [Module 6 — Command Palette Panel](#8-module-6--command-palette-panel)
9. [Message Protocol](#9-message-protocol)
10. [ROS 2 Bridge Architecture](#10-ros-2-bridge-architecture)
11. [Windows & Cross-Platform Support](#11-windows--cross-platform-support)
12. [Production Quality Standards](#12-production-quality-standards)
13. [Phased Build Plan](#13-phased-build-plan)
14. [Tech Stack Summary](#14-tech-stack-summary)
15. [Risk Register](#15-risk-register)

---

## 1. Overview & Vision

### What Virtus ROS 2 Studio Is

A VS Code extension that gives Antony a **Virtusco-specific ROS 2 development environment** — pre-loaded with the Porter robot's package structure, topic names, custom message schemas, ESP32 bridge protocol, and 9-state FSM orchestrator. Generic ROS extensions (like Microsoft's ROS extension) don't know any of this context.

### Why Generic ROS Extensions Fall Short

The existing Microsoft ROS extension for VS Code provides:
- Basic syntax highlighting for `.msg`, `.srv`, `.action` files
- A launch file runner
- A generic `ros2 topic list` panel

What it does NOT provide:
- Knowledge of your specific topic graph (`/scan/processed`, `/sensor_fusion`, `/cmd_vel`, `/diagnostics`)
- Decoding of your CRC16-CCITT binary serial protocol between RPi and ESP32s
- Visualization of your 9-state orchestrator FSM
- Pre-built test commands for your hardware
- Nav2 parameter editing tuned to your robot's dimensions

### What Virtus ROS 2 Studio Provides

```
Live Topic Monitor        → /scan, /cmd_vel, /sensor_fusion decoded in real time
Node Graph Visualizer     → Live 9-package graph, dead node alerts
FSM State Viewer          → Orchestrator state machine with transition history
ESP32 Bridge Debugger     → CRC16-CCITT binary frame decoder
Launch File Builder       → Visual multi-node launch composer
Command Palette Panel     → Pre-built ros2 commands for Virtus hardware
```

### Virtus Robot Context Baked In

The extension knows:
- **9 ROS 2 packages:** `ydlidar_driver`, `porter_lidar_processor`, `porter_orchestrator`, `porter_esp32_bridge`, `porter_ai_assistant`, plus Nav2 packages
- **Key topics:** `/scan`, `/scan/processed`, `/cmd_vel`, `/sensor_fusion`, `/diagnostics`, `/ai_assistant/response`, `/odom`
- **Custom message types:** sensor fusion struct, ESP32 bridge protocol frames, orchestrator state
- **9 FSM states:** `BOOT`, `HEALTH_CHECK`, `IDLE`, `PASSENGER_DETECTED`, `FOLLOWING`, `NAVIGATING`, `OBSTACLE_AVOIDANCE`, `ERROR`, `RECOVERY`
- **ESP32 binary protocol:** CRC16-CCITT framed packets, little-endian, defined in `common/protocol.h`

---

## 2. Extension Architecture

### Directory Structure

```
virtus-ros2-studio/
├── package.json                          ← Manifest, commands, activation on ROS 2 workspace
├── tsconfig.json
├── esbuild.config.js                     ← Dual bundle: host + webview
│
├── src/                                  ← Extension Host (TypeScript)
│   ├── extension.ts                      ← Entry: detects ROS 2 workspace, registers commands
│   ├── panel/
│   │   └── ROS2StudioPanel.ts            ← WebviewPanel with tabbed navigation
│   ├── ros2/
│   │   ├── ROS2Bridge.ts                 ← Spawns roslibjs-compatible bridge (rosbridge_server)
│   │   ├── TopicSubscriber.ts            ← Subscribes to topics, streams to webview
│   │   ├── NodeGraphReader.ts            ← Calls ros2 node list + ros2 node info
│   │   ├── ParameterClient.ts            ← Gets/sets ROS 2 params via ros2 param CLI
│   │   └── WorkspaceDetector.ts          ← Finds colcon workspace, reads package.xml files
│   ├── esp32/
│   │   └── FrameDecoder.ts               ← CRC16-CCITT binary protocol parser
│   ├── fsm/
│   │   └── FSMTracker.ts                 ← Subscribes to /orchestrator/state topic
│   ├── launch/
│   │   ├── LaunchParser.ts               ← Parses existing .py launch files
│   │   └── LaunchGenerator.ts            ← Generates launch Python from visual graph
│   └── platform/
│       └── PlatformUtils.ts              ← Cross-platform ros2 CLI path resolution
│
├── python/
│   ├── ros2_bridge.py                    ← rosbridge WebSocket server wrapper
│   ├── topic_echo.py                     ← Streams topic data as SSE to extension
│   ├── node_graph.py                     ← ros2 node/topic graph as JSON
│   └── param_server.py                   ← ros2 param get/set wrapper
│
└── webview-ui/
    └── src/
        ├── App.tsx                       ← Tabbed panel: Monitor / Graph / FSM / Bridge / Launch / Commands
        ├── pages/
        │   ├── TopicMonitorPage.tsx      ← Live topic table with decoded values
        │   ├── NodeGraphPage.tsx         ← Interactive node graph (react-flow)
        │   ├── FSMViewerPage.tsx         ← State machine diagram with live state highlight
        │   ├── BridgeDebuggerPage.tsx    ← Binary frame decoder panel
        │   ├── LaunchBuilderPage.tsx     ← Visual launch file composer
        │   └── CommandsPage.tsx          ← Pre-built command shortcuts
        └── components/
            ├── TopicRow.tsx              ← Single topic with type badge + value preview
            ├── MessageInspector.tsx      ← Expandable nested message tree
            ├── StateNode.tsx             ← FSM state box (active/inactive/error)
            ├── FrameHexView.tsx          ← Hex + decoded struct side by side
            └── FrequencyGraph.tsx        ← Real-time publish rate sparkline
```

---

## 3. Module 1 — Topic Monitor

### Architecture

The topic monitor subscribes to ROS 2 topics via a **rosbridge WebSocket server** (`rosbridge_suite`) running as a subprocess. The extension connects to it via WebSocket and streams decoded messages to the webview via postMessage.

```
ROS 2 graph
    ↓ rosbridge_server (WebSocket on port 9090)
    ↓ TopicSubscriber.ts (WebSocket client)
    ↓ postMessage to webview
    ↓ TopicMonitorPage.tsx
```

### Virtus Topic Registry

The extension pre-loads this topic map — no manual configuration needed:

```typescript
// ros2/TopicRegistry.ts
export const VIRTUS_TOPICS: TopicDef[] = [
  { name: '/scan',              type: 'sensor_msgs/LaserScan',       category: 'lidar',    hz_expected: 10  },
  { name: '/scan/processed',    type: 'sensor_msgs/LaserScan',       category: 'lidar',    hz_expected: 10  },
  { name: '/cmd_vel',           type: 'geometry_msgs/Twist',         category: 'control',  hz_expected: 10  },
  { name: '/odom',              type: 'nav_msgs/Odometry',           category: 'nav',      hz_expected: 30  },
  { name: '/sensor_fusion',     type: 'virtus_msgs/SensorFusion',    category: 'sensors',  hz_expected: 50  },
  { name: '/diagnostics',       type: 'diagnostic_msgs/DiagnosticArray', category: 'health', hz_expected: 1  },
  { name: '/orchestrator/state',type: 'virtus_msgs/OrchestratorState', category: 'fsm',    hz_expected: 5   },
  { name: '/ai_assistant/query',type: 'std_msgs/String',             category: 'ai',       hz_expected: 0.1 },
  { name: '/ai_assistant/response', type: 'std_msgs/String',         category: 'ai',       hz_expected: 0.1 },
  { name: '/esp32_bridge/rx',   type: 'virtus_msgs/BridgeFrame',     category: 'bridge',   hz_expected: 50  },
  { name: '/esp32_bridge/tx',   type: 'virtus_msgs/BridgeFrame',     category: 'bridge',   hz_expected: 50  },
  { name: '/map',               type: 'nav_msgs/OccupancyGrid',      category: 'nav',      hz_expected: 0.5 },
  { name: '/tf',                type: 'tf2_msgs/TFMessage',          category: 'tf',       hz_expected: 30  },
];
```

### Topic Monitor UI

```
┌── Topic Monitor ─────────────────────────────────────────────────────────┐
│  [● ROS 2 Connected]  [Filter: all ▼]  [Search topics...]  [Pause ■]    │
│                                                                           │
│  Topic                    Type                      Hz     Status         │
│  ─────────────────────────────────────────────────────────────────────── │
│  ▶ /scan                  sensor_msgs/LaserScan     10.2   ● OK           │
│  ▶ /scan/processed        sensor_msgs/LaserScan     10.1   ● OK           │
│  ▶ /cmd_vel               geometry_msgs/Twist        0.0   ○ Silent       │
│  ▶ /sensor_fusion         virtus_msgs/SensorFusion  49.8   ● OK           │
│  ▼ /diagnostics           diagnostic_msgs/...        1.0   ⚠ WARN        │
│      level: WARN                                                          │
│      message: "LiDAR intensity low — check laser window"                 │
│  ▶ /orchestrator/state    virtus_msgs/Orchestrator   5.0   ● OK           │
│  ▶ /esp32_bridge/rx       virtus_msgs/BridgeFrame   50.1   ● OK           │
│  ✗ /odom                  nav_msgs/Odometry           —    ✗ Not present  │
└───────────────────────────────────────────────────────────────────────────┘
```

**Key features:**
- **Hz monitoring:** Each topic shows measured publish rate vs expected. Red if > 20% off expected.
- **Silent alert:** Topics expected to publish but haven't in > 2s turn orange
- **Not present alert:** Topics in registry not found in `ros2 topic list` shown in red with ✗
- **Inline expand:** Click any row to see last message decoded as nested tree
- **Pause:** Freezes display without unsubscribing — useful for inspecting a message
- **Custom topics:** Add any topic not in the registry via the `+` button

### Message Inspector (Nested Tree)

```
▼ /sensor_fusion  [virtus_msgs/SensorFusion]  @ 49.8 Hz
    tof_distance_mm:     342
    ultrasonic_dist_cm:  34.8
    microwave_detected:  false
    kalman_estimate_cm:  34.2
    kalman_variance:     0.043
    obstacle_class:      CLEAR
    timestamp:           1742318422.841
    frame_id:            "base_link"
```

---

## 4. Module 2 — Node Graph Visualizer

### Data Source

Polls `ros2 node list` + `ros2 node info <node>` every 2 seconds to build the graph. Parses publishers, subscribers, services, and action servers per node.

### Graph Layout

Uses React Flow with a **dagre** auto-layout algorithm. Nodes are color-coded by package:

```
ydlidar_driver            → Blue
porter_lidar_processor    → Cyan
porter_orchestrator       → Purple
porter_esp32_bridge       → Orange
porter_ai_assistant       → Green
nav2_*                    → Grey
```

Edges represent topic connections (publisher → subscriber). Edge thickness = publish Hz.

### Dead Node Detection

```typescript
// NodeGraphReader.ts
interface NodeHealth {
  name:         string;
  package:      string;
  status:       'alive' | 'dead' | 'degraded';
  last_seen:    number;     // timestamp
  pub_count:    number;
  sub_count:    number;
}

// A node is "dead" if it was in the graph 10s ago and is now gone
// A node is "degraded" if its topics are publishing at < 50% expected rate
```

Dead nodes shown in red on graph. VS Code notification fires when a required node dies during testing.

### Required Node List (Virtus-specific)

```typescript
export const REQUIRED_NODES = [
  '/ydlidar_node',
  '/lidar_processor',
  '/orchestrator',
  '/esp32_bridge',
  '/ai_assistant',
];
// Nav2 nodes added when Nav2 launch profile is active
```

---

## 5. Module 3 — FSM State Viewer

### State Machine Definition

Hard-coded from `porter_orchestrator` source — the 9 states and their valid transitions:

```typescript
// fsm/FSMDefinition.ts
export const ORCHESTRATOR_FSM = {
  states: [
    'BOOT', 'HEALTH_CHECK', 'IDLE',
    'PASSENGER_DETECTED', 'FOLLOWING', 'NAVIGATING',
    'OBSTACLE_AVOIDANCE', 'ERROR', 'RECOVERY'
  ],
  transitions: [
    { from: 'BOOT',               to: 'HEALTH_CHECK',       trigger: 'system_ready'       },
    { from: 'HEALTH_CHECK',       to: 'IDLE',               trigger: 'all_healthy'         },
    { from: 'HEALTH_CHECK',       to: 'ERROR',              trigger: 'health_fail'         },
    { from: 'IDLE',               to: 'PASSENGER_DETECTED', trigger: 'passenger_detected'  },
    { from: 'PASSENGER_DETECTED', to: 'FOLLOWING',          trigger: 'confirmed'           },
    { from: 'PASSENGER_DETECTED', to: 'IDLE',               trigger: 'timeout'             },
    { from: 'FOLLOWING',          to: 'NAVIGATING',         trigger: 'gate_requested'      },
    { from: 'FOLLOWING',          to: 'OBSTACLE_AVOIDANCE', trigger: 'obstacle_detected'   },
    { from: 'NAVIGATING',         to: 'IDLE',               trigger: 'arrived'             },
    { from: 'NAVIGATING',         to: 'OBSTACLE_AVOIDANCE', trigger: 'obstacle_detected'   },
    { from: 'OBSTACLE_AVOIDANCE', to: 'NAVIGATING',         trigger: 'clear'               },
    { from: 'OBSTACLE_AVOIDANCE', to: 'ERROR',              trigger: 'stuck'               },
    { from: 'ERROR',              to: 'RECOVERY',           trigger: 'recovery_triggered'  },
    { from: 'RECOVERY',           to: 'HEALTH_CHECK',       trigger: 'reset'               },
  ]
};
```

### FSM Viewer UI

```
┌── FSM State Viewer ──────────────────────────────────────────────────────┐
│  Current State: [ NAVIGATING ]  Duration: 00:02:14                       │
│                                                                           │
│   [BOOT] → [HEALTH_CHECK] → [IDLE]                                       │
│                               ↓                                          │
│                    [PASSENGER_DETECTED]                                   │
│                               ↓                                          │
│                          [FOLLOWING] ←────────────────────────┐          │
│                               ↓                               │          │
│                         ●[NAVIGATING]──→[OBSTACLE_AVOIDANCE]──┘          │
│                               ↓                ↓                         │
│                             [IDLE]           [ERROR]→[RECOVERY]           │
│                                                                           │
│  Transition History (last 10):                                            │
│  14:32:01  IDLE → PASSENGER_DETECTED  [passenger_detected]               │
│  14:32:04  PASSENGER_DETECTED → FOLLOWING  [confirmed]                   │
│  14:32:09  FOLLOWING → NAVIGATING  [gate_requested]                      │
└───────────────────────────────────────────────────────────────────────────┘
```

**Active state** shown with filled circle (●) and highlight. **Transition history** shows last 10 transitions with timestamps and trigger events. Alert fires if FSM enters `ERROR` state.

---

## 6. Module 4 — ESP32 Bridge Debugger

### Protocol Background

The `porter_esp32_bridge` uses a binary serial protocol (CRC16-CCITT) defined in `esp32_firmware/common/protocol.h`. Raw frames come in over USB CDC. Without this debugger, Danush and Antony can only see raw hex in a serial monitor.

### Frame Format

```
[START_BYTE: 0xAA] [MSG_TYPE: 1B] [LENGTH: 2B LE] [PAYLOAD: NB] [CRC16: 2B LE]
```

### Message Types

```typescript
// esp32/FrameDefinitions.ts
export const FRAME_TYPES: Record<number, FrameDef> = {
  0x01: { name: 'MOTOR_CMD',    fields: [
    { name: 'left_speed',  type: 'int16', unit: 'duty%'  },
    { name: 'right_speed', type: 'int16', unit: 'duty%'  },
    { name: 'lift_cmd',    type: 'uint8', unit: 'enum'   },
  ]},
  0x02: { name: 'SENSOR_DATA',  fields: [
    { name: 'tof_mm',      type: 'uint16', unit: 'mm'    },
    { name: 'ultrasonic',  type: 'uint16', unit: 'cm'    },
    { name: 'microwave',   type: 'uint8',  unit: 'bool'  },
    { name: 'kalman_cm',   type: 'float32', unit: 'cm'   },
  ]},
  0x03: { name: 'MOTOR_STATUS', fields: [
    { name: 'left_current',  type: 'uint16', unit: 'mA'  },
    { name: 'right_current', type: 'uint16', unit: 'mA'  },
    { name: 'en_state',      type: 'uint8',  unit: 'bool'},
  ]},
  0x04: { name: 'HEARTBEAT',    fields: [
    { name: 'uptime_ms',   type: 'uint32', unit: 'ms'    },
    { name: 'error_flags', type: 'uint8',  unit: 'bitmask'},
  ]},
  0xFF: { name: 'ERROR_FRAME',  fields: [
    { name: 'error_code',  type: 'uint8',  unit: 'enum'  },
    { name: 'context',     type: 'uint16', unit: 'raw'   },
  ]},
};
```

### Bridge Debugger UI

```
┌── ESP32 Bridge Debugger ─────────────────────────────────────────────────┐
│  Source: [/dev/ttyUSB0 ▼]  Baud: [115200]  [● Connected]  [Clear]       │
│                                                                           │
│  Hex Stream:                        Decoded:                              │
│  AA 01 04 00 FF 00 00 00 B2 3C  →  MOTOR_CMD                            │
│                                     left_speed:  -1  duty%               │
│                                     right_speed: 0   duty%               │
│                                     lift_cmd:    0   STOP                │
│                                     CRC: 0x3CB2 ✓                        │
│  ─────────────────────────────────────────────────────────────────────── │
│  AA 02 08 00 56 01 22 00 01 ...  →  SENSOR_DATA                         │
│                                     tof_mm:     342  mm                  │
│                                     ultrasonic: 34   cm                  │
│                                     microwave:  false                    │
│                                     kalman_cm:  34.2 cm                  │
│                                     CRC: 0x... ✓                         │
│  ─────────────────────────────────────────────────────────────────────── │
│  AA FF 02 00 03 00 91 2A       →  ⚠ ERROR_FRAME                         │
│                                     error_code: 3 (MOTOR_STALL)         │
│                                     context:    0x0000                   │
└───────────────────────────────────────────────────────────────────────────┘
```

**CRC validation:** Each decoded frame shows CRC check result (✓ or ✗). Bad CRC frames highlighted in red — indicates serial corruption or protocol mismatch.

**Error frame alerts:** When an `ERROR_FRAME` (0xFF) arrives, a VS Code notification fires with the error code name.

---

## 7. Module 5 — Launch File Builder

### Visual Launch Composer

Drag nodes representing ROS 2 nodes onto a canvas, configure parameters, wire namespaces, then export as a valid `launch.py` file.

### Node Templates

Pre-defined templates for all Virtus packages:

```typescript
export const LAUNCH_TEMPLATES: LaunchNodeTemplate[] = [
  {
    pkg:        'ydlidar_ros2_driver',
    executable: 'ydlidar_ros2_driver_node',
    name:       'ydlidar_node',
    params:     { port: '/dev/ttyUSB0', baudrate: 230400, frame_id: 'laser_frame' }
  },
  {
    pkg:        'porter_lidar_processor',
    executable: 'lidar_processor_node',
    name:       'lidar_processor',
    params:     { range_min: 0.1, range_max: 10.0, downsample_factor: 2 }
  },
  {
    pkg:        'porter_orchestrator',
    executable: 'orchestrator_node',
    name:       'orchestrator',
    params:     { health_check_timeout: 5.0, recovery_attempts: 3 }
  },
  {
    pkg:        'porter_esp32_bridge',
    executable: 'esp32_bridge_node',
    name:       'esp32_bridge',
    params:     { port: '/dev/ttyUSB1', baud: 115200, timeout: 1.0 }
  },
  {
    pkg:        'porter_ai_assistant',
    executable: 'ai_assistant_node',
    name:       'ai_assistant',
    params:     { model_path: 'models/virtus-llm.gguf', n_ctx: 2048 }
  },
];
```

### Generated launch.py

```python
# Auto-generated by Virtus ROS 2 Studio — DO NOT EDIT
# Re-generate from the .virtuslaunch canvas file
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='ydlidar_ros2_driver',
            executable='ydlidar_ros2_driver_node',
            name='ydlidar_node',
            parameters=[{'port': '/dev/ttyUSB0', 'baudrate': 230400}]
        ),
        Node(
            package='porter_lidar_processor',
            executable='lidar_processor_node',
            name='lidar_processor',
        ),
        # ... more nodes
    ])
```

---

## 8. Module 6 — Command Palette Panel

Pre-built `ros2` commands for Virtus hardware. One click executes in VS Code terminal.

```
┌── Virtus Commands ────────────────────────────────────────────────────────┐
│  MOVEMENT                                                                 │
│  [Move Forward]     ros2 topic pub /cmd_vel geometry_msgs/Twist ...      │
│  [Move Backward]    ros2 topic pub /cmd_vel ...                           │
│  [Rotate Left]      ros2 topic pub /cmd_vel ...                           │
│  [Stop]             ros2 topic pub /cmd_vel ... "linear: {x: 0.0}"       │
│                                                                           │
│  DIAGNOSTICS                                                              │
│  [List All Nodes]   ros2 node list                                        │
│  [Topic Hz Check]   ros2 topic hz /scan /sensor_fusion /cmd_vel          │
│  [Param Dump]       ros2 param dump /orchestrator                        │
│  [TF Tree]          ros2 run tf2_tools view_frames                       │
│                                                                           │
│  BUILD                                                                    │
│  [Build All]        colcon build --symlink-install                        │
│  [Build Package]    colcon build --packages-select [package ▼]           │
│  [Source Workspace] source install/setup.bash                             │
│                                                                           │
│  RECORDING                                                                │
│  [Record All]       ros2 bag record -a -o bags/test_{timestamp}          │
│  [Record Key Topics] ros2 bag record /scan /cmd_vel /sensor_fusion ...   │
│  [Play Last Bag]    ros2 bag play bags/{last_bag}                        │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Message Protocol

```typescript
// Webview → Host
type WebviewMessage =
  | { type: 'subscribeTopic';   payload: { name: string; type: string } }
  | { type: 'unsubscribeTopic'; payload: { name: string } }
  | { type: 'publishTopic';     payload: { name: string; msg: object } }
  | { type: 'runCommand';       payload: { cmd: string } }
  | { type: 'getNodeGraph';     payload: void }
  | { type: 'getParams';        payload: { node: string } }
  | { type: 'setParam';         payload: { node: string; param: string; value: any } }
  | { type: 'saveLaunch';       payload: { graph: LaunchGraph; outputPath: string } }

// Host → Webview
type HostMessage =
  | { type: 'topicMessage';     payload: { name: string; msg: object; stamp: number } }
  | { type: 'topicHz';          payload: { name: string; hz: number } }
  | { type: 'nodeGraph';        payload: NodeGraph }
  | { type: 'fsmState';         payload: { state: string; trigger: string; stamp: number } }
  | { type: 'bridgeFrame';      payload: DecodedFrame }
  | { type: 'nodeAlert';        payload: { node: string; type: 'dead' | 'degraded' } }
  | { type: 'ros2Status';       payload: { connected: boolean; version: string } }
```

---

## 10. ROS 2 Bridge Architecture

### rosbridge WebSocket

The extension spawns a `rosbridge_server` WebSocket on port 9090. This is the standard ROS 2 web bridge — well maintained, no custom code needed:

```bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml port:=9090
```

The `ROS2Bridge.ts` connects to `ws://localhost:9090` using the standard `roslib.js` protocol.

### Fallback: CLI Polling

If `rosbridge_suite` is not installed, the extension falls back to polling `ros2 topic echo --once` via subprocess. Slower (2Hz max) but works without installing rosbridge. The UI shows a "rosbridge not found — install for real-time monitoring" banner.

---

## 11. Windows & Cross-Platform Support

| Feature | Linux | Windows |
|---|---|---|
| rosbridge WebSocket | ✅ Native | ✅ Via WSL2 (ROS 2 runs in WSL2) |
| `ros2` CLI calls | ✅ Native | ✅ Via WSL2 (`wsl ros2 node list`) |
| `/dev/ttyUSB*` serial | ✅ Native | ✅ Via WSL2 USB passthrough (`usbipd`) |
| Node graph polling | ✅ | ✅ Via WSL2 |
| Topic monitor | ✅ | ✅ Via WSL2 |
| Launch file generation | ✅ | ✅ (file written to workspace) |
| Build (colcon) | ✅ | ✅ Via WSL2 terminal |

Windows users run ROS 2 inside WSL2. The extension detects WSL2 availability and prefixes all `ros2` CLI calls with `wsl`:

```typescript
// platform/PlatformUtils.ts
function ros2Cmd(args: string[]): string {
  const cmd = ['ros2', ...args].join(' ');
  return Platform.isWindows ? `wsl ${cmd}` : cmd;
}
```

---

## 12. Production Quality Standards

- **rosbridge disconnect handling:** Auto-reconnect every 5s with exponential backoff. UI shows connection status badge always.
- **Topic overflow protection:** Cap message buffer at 100 messages per topic — oldest dropped. Prevents memory leak on high-Hz topics like `/tf`.
- **No topic spam:** Subscribing to `/tf` at 30Hz in the webview is fine; subscribing to `/scan` (full LaserScan array) at 10Hz is heavy. Extension sub-samples: renders at 5Hz max regardless of topic Hz.
- **colcon build detection:** Extension detects if workspace is not built (no `install/` dir) and shows a banner before running any ros2 commands.
- **Package.xml parsing:** Reads all `package.xml` files in the workspace to know which packages exist. Validates that launch templates reference real packages.

---

## 13. Phased Build Plan

### Phase 1 — Scaffold + ROS 2 Connection (Week 1)
- Extension activates on workspace with `package.xml` or `colcon.meta`
- `ROS2Bridge.ts` spawns rosbridge, connects WebSocket
- Dashboard shows ROS 2 version + connection status
- **Deliverable:** Connected panel that confirms ROS 2 is running

### Phase 2 — Topic Monitor (Weeks 2–3)
- `TopicSubscriber.ts` subscribes to all VIRTUS_TOPICS
- `TopicMonitorPage.tsx` with Hz monitoring + silent alerts
- Message inspector with nested tree expand
- **Deliverable:** Live topic table with Virtus topics pre-loaded

### Phase 3 — Node Graph (Week 4)
- `NodeGraphReader.ts` polls ros2 CLI
- React Flow graph with dagre layout + color coding
- Dead node detection + VS Code alerts
- **Deliverable:** Live node graph with health monitoring

### Phase 4 — FSM State Viewer (Week 5)
- `FSMTracker.ts` subscribes to `/orchestrator/state`
- `FSMViewerPage.tsx` with state diagram + transition history
- ERROR state alert
- **Deliverable:** Live FSM visualization

### Phase 5 — ESP32 Bridge Debugger (Week 6)
- `FrameDecoder.ts` implements CRC16-CCITT parser
- All 5 frame types decoded
- Hex + decoded side-by-side UI
- **Deliverable:** Real-time binary protocol debugger

### Phase 6 — Launch File Builder (Week 7)
- `LaunchParser.ts` reads existing `.py` launch files
- Canvas with all Virtus package templates
- `LaunchGenerator.ts` outputs valid `launch.py`
- **Deliverable:** Visual launch file composer

### Phase 7 — Command Palette + Polish (Week 8)
- Pre-built commands panel
- Windows / WSL2 support
- Full error handling + rosbridge fallback
- **Deliverable:** Production-ready extension

---

## 14. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| ROS 2 bridge | `rosbridge_suite` WebSocket | Standard, well-maintained |
| WebSocket client | `roslib` (roslibjs) | Standard rosbridge client protocol |
| Node graph | React Flow + dagre | Consistent with firmware builder |
| Charts | Recharts | Hz sparklines, consistent with AI Studio |
| FSM layout | Custom React Flow layout | Static layout matches known FSM |
| Serial decoding | Node.js `Buffer` API | CRC16-CCITT in TypeScript |
| CLI runner | `child_process.exec` | ros2 CLI calls |
| Cross-platform | WSL2 bridge for Windows | ROS 2 has no native Windows support |

---

## 15. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| rosbridge_suite not installed | Medium | High | CLI polling fallback; install guide shown |
| `/orchestrator/state` topic schema changes | Medium | Medium | Schema versioned in registry; warn on mismatch |
| High-Hz topics (/tf, /scan) flood WebSocket | High | Medium | Sub-sample to 5Hz in extension host before forwarding |
| Windows USB passthrough to WSL2 complex | High | Medium | Document usbipd setup; serial monitor works via WSL2 |
| CRC16 polynomial mismatch with firmware | Low | High | Unit test decoder against known good frames from firmware test suite |
| ROS 2 version mismatch (Jazzy vs future) | Low | Medium | Version detected at startup; warn if unsupported version |
