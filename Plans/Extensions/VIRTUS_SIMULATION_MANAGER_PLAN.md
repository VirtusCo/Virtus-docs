# Virtus Simulation Manager — VS Code Extension
### Detailed Development Plan
**Project:** Gazebo simulation environment manager for Virtus robot — one-click launch profiles, URDF preview, Nav2 parameter editor, bag file manager, scenario runner  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [Extension Architecture](#2-extension-architecture)
3. [Module 1 — Launch Profile Manager](#3-module-1--launch-profile-manager)
4. [Module 2 — URDF Live Preview](#4-module-2--urdf-live-preview)
5. [Module 3 — Nav2 Parameter Editor](#5-module-3--nav2-parameter-editor)
6. [Module 4 — Bag File Manager](#6-module-4--bag-file-manager)
7. [Module 5 — Test Scenario Runner](#7-module-5--test-scenario-runner)
8. [Module 6 — World Manager](#8-module-6--world-manager)
9. [Message Protocol](#9-message-protocol)
10. [Windows & Cross-Platform Support](#10-windows--cross-platform-support)
11. [Production Quality Standards](#11-production-quality-standards)
12. [Phased Build Plan](#12-phased-build-plan)
13. [Tech Stack Summary](#13-tech-stack-summary)
14. [Risk Register](#14-risk-register)

---

## 1. Overview & Vision

### The Problem

Every Gazebo simulation session for Virtus requires executing 6+ terminal commands in the right order, across multiple terminals:

```bash
# Terminal 1 — Gazebo
ros2 launch gazebo_ros gazebo.launch.py world:=src/worlds/airport.world

# Terminal 2 — Robot spawn
ros2 run gazebo_ros spawn_entity.py -topic robot_description -entity virtus

# Terminal 3 — Robot state publisher
ros2 launch porter_bringup robot_state_publisher.launch.py

# Terminal 4 — Nav2
ros2 launch nav2_bringup navigation_launch.py params_file:=config/nav2_params.yaml

# Terminal 5 — RViz2
rviz2 -d config/virtus_nav.rviz

# Terminal 6 — Your actual development node
ros2 run porter_orchestrator orchestrator_node
```

This is repeated dozens of times per development session. When Nav2 parameters need tuning, the developer edits a YAML by hand, restarts the entire stack, and hopes.

### What Virtus Simulation Manager Provides

```
Launch Profiles      → One-click: bare_robot / with_nav2 / full_stack / ai_enabled
URDF Preview         → Renders Virtus URDF in webview — no Gazebo needed
Nav2 Parameter Editor → GUI for all Nav2 params with descriptions and defaults
Bag File Manager     → Record, replay, trim, annotate .mcap bags from VS Code
Scenario Runner      → Define and replay test scenarios (obstacles, passengers)
World Manager        → Browse and swap Gazebo worlds
```

---

## 2. Extension Architecture

```
virtus-simulation-manager/
├── package.json
├── tsconfig.json
├── esbuild.config.js
│
├── src/
│   ├── extension.ts
│   ├── panel/
│   │   └── SimManagerPanel.ts
│   ├── launch/
│   │   ├── ProfileManager.ts           ← Stores/loads launch profiles
│   │   ├── ProfileRunner.ts            ← Spawns ros2 launch processes
│   │   └── ProcessTracker.ts           ← Tracks all spawned processes
│   ├── urdf/
│   │   ├── URDFParser.ts               ← Parses .urdf XML
│   │   └── URDFRenderer.ts             ← Converts URDF to Three.js scene
│   ├── nav2/
│   │   ├── Nav2ParamLoader.ts          ← Loads nav2_params.yaml
│   │   ├── Nav2ParamSchema.ts          ← Schema: all params + descriptions
│   │   └── Nav2ParamWriter.ts          ← Writes modified YAML back to disk
│   ├── bags/
│   │   ├── BagManager.ts               ← Scans bags/ directory
│   │   ├── BagRecorder.ts              ← ros2 bag record subprocess
│   │   └── BagPlayer.ts                ← ros2 bag play subprocess
│   ├── scenarios/
│   │   ├── ScenarioStore.ts            ← Loads/saves .virtusscenario files
│   │   └── ScenarioRunner.ts           ← Injects obstacles/entities into Gazebo
│   └── platform/
│       └── PlatformUtils.ts
│
└── webview-ui/
    └── src/
        ├── pages/
        │   ├── LaunchPage.tsx          ← Profile cards + process status
        │   ├── URDFPage.tsx            ← 3D URDF viewer
        │   ├── Nav2Page.tsx            ← Parameter editor with descriptions
        │   ├── BagsPage.tsx            ← Bag file browser + controls
        │   ├── ScenariosPage.tsx       ← Scenario editor + runner
        │   └── WorldsPage.tsx          ← World file browser
        └── components/
            ├── ProfileCard.tsx         ← Launch profile tile with status
            ├── ProcessStatusBar.tsx    ← Per-process health indicators
            ├── URDFViewer.tsx          ← Three.js canvas component
            ├── ParamGroup.tsx          ← Collapsible param group with descriptions
            ├── BagRow.tsx              ← Bag file row with actions
            └── ScenarioCanvas.tsx     ← 2D top-down scenario editor
```

---

## 3. Module 1 — Launch Profile Manager

### Virtus Launch Profiles

```typescript
// launch/ProfileManager.ts
export const VIRTUS_PROFILES: LaunchProfile[] = [
  {
    id:    'bare_robot',
    label: 'Bare Robot',
    description: 'Gazebo + URDF only. No Nav2, no AI.',
    color: '#4A90D9',
    steps: [
      { name: 'Gazebo',          cmd: 'ros2 launch gazebo_ros gazebo.launch.py world:=src/worlds/airport.world' },
      { name: 'URDF Publisher',  cmd: 'ros2 launch porter_bringup robot_state_publisher.launch.py' },
      { name: 'Robot Spawn',     cmd: 'ros2 run gazebo_ros spawn_entity.py -topic robot_description -entity virtus', delay_ms: 3000 },
    ]
  },
  {
    id:    'with_nav2',
    label: 'Nav2 Stack',
    description: 'Full navigation stack with RViz2.',
    color: '#7B68EE',
    steps: [
      { name: 'Gazebo',          cmd: 'ros2 launch gazebo_ros gazebo.launch.py world:=src/worlds/airport.world' },
      { name: 'URDF Publisher',  cmd: 'ros2 launch porter_bringup robot_state_publisher.launch.py' },
      { name: 'Robot Spawn',     cmd: 'ros2 run gazebo_ros spawn_entity.py -topic robot_description -entity virtus', delay_ms: 3000 },
      { name: 'Nav2',            cmd: 'ros2 launch nav2_bringup navigation_launch.py params_file:=config/nav2_params.yaml', delay_ms: 2000 },
      { name: 'RViz2',           cmd: 'rviz2 -d config/virtus_nav.rviz', delay_ms: 1000 },
    ]
  },
  {
    id:    'full_stack',
    label: 'Full Stack',
    description: 'Nav2 + Orchestrator + ESP32 bridge (real or simulated).',
    color: '#50C878',
    steps: [
      // All of with_nav2 steps +
      { name: 'Orchestrator',    cmd: 'ros2 run porter_orchestrator orchestrator_node', delay_ms: 2000 },
      { name: 'ESP32 Bridge',    cmd: 'ros2 run porter_esp32_bridge esp32_bridge_node --ros-args -p sim_mode:=true', delay_ms: 500 },
    ]
  },
  {
    id:    'ai_enabled',
    label: 'AI Enabled',
    description: 'Full stack + AI assistant (Virtue/Qwen).',
    color: '#FF8C00',
    steps: [
      // All of full_stack steps +
      { name: 'AI Assistant',    cmd: 'ros2 run porter_ai_assistant ai_assistant_node', delay_ms: 3000 },
    ]
  },
];
```

### Launch Page UI

```
┌── Launch Profiles ───────────────────────────────────────────────────────┐
│                                                                           │
│  ┌─ Bare Robot ──────┐  ┌─ Nav2 Stack ───────┐  ┌─ Full Stack ─────────┐│
│  │ Gazebo + URDF     │  │ Full navigation    │  │ Nav2 + Orchestrator  ││
│  │ No Nav2, no AI    │  │ stack with RViz2   │  │ + ESP32 bridge       ││
│  │                   │  │                    │  │                      ││
│  │ [▶ Launch]        │  │ [▶ Launch]         │  │ [▶ Launch]           ││
│  └───────────────────┘  └────────────────────┘  └──────────────────────┘│
│                                                                           │
│  Active: Nav2 Stack — Running for 00:04:22  [■ Stop All]                │
│  ─────────────────────────────────────────────────────────────────────── │
│  ● Gazebo              Running  PID 12341                                 │
│  ● URDF Publisher      Running  PID 12355                                 │
│  ● Robot Spawn         Done     (exited 0)                               │
│  ● Nav2                Running  PID 12389                                 │
│  ● RViz2               Running  PID 12401                                 │
└───────────────────────────────────────────────────────────────────────────┘
```

**Step sequencing with delays:** Each step can have a `delay_ms` — critical because Gazebo must be fully loaded before the robot can be spawned. The extension waits for each process to emit a ready signal (or times out after configurable duration) before starting the next step.

**Stop All:** Kills all tracked processes in reverse order (graceful SIGTERM, then SIGKILL after 3s).

---

## 4. Module 2 — URDF Live Preview

### Architecture

Parses the Virtus URDF (`porter_bringup/urdf/virtus.urdf`) and renders it as a 3D scene in the webview using **Three.js**. No Gazebo or RViz2 needed — instant preview while editing URDF.

### URDF Parser

```typescript
// urdf/URDFParser.ts
interface URDFLink {
  name:      string;
  visual:    URDFGeometry | null;
  collision: URDFGeometry | null;
  inertial:  URDFInertial | null;
}

interface URDFJoint {
  name:   string;
  type:   'fixed' | 'revolute' | 'continuous' | 'prismatic';
  parent: string;
  child:  string;
  origin: { xyz: [number, number, number]; rpy: [number, number, number] };
  axis:   [number, number, number];
  limits: { lower: number; upper: number; velocity: number; effort: number } | null;
}

export function parseURDF(xml: string): { links: URDFLink[]; joints: URDFJoint[] } {
  const doc    = new DOMParser().parseFromString(xml, 'text/xml');
  const links  = [...doc.querySelectorAll('link')].map(parseLink);
  const joints = [...doc.querySelectorAll('joint')].map(parseJoint);
  return { links, joints };
}
```

### URDF Viewer UI

```
┌── URDF Preview — virtus.urdf ────────────────────────────────────────────┐
│  [Wireframe ○] [Solid ●] [Show Collision ○] [Show Joints ●]  [Reset View]│
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐│
│  │                                                                      ││
│  │                    [3D Virtus robot rendered]                        ││
│  │                    Click joint to highlight                          ││
│  │                    Scroll to zoom, drag to rotate                    ││
│  │                                                                      ││
│  └──────────────────────────────────────────────────────────────────────┘│
│                                                                           │
│  Links: 12  Joints: 11  Meshes: 8  ⚠ 1 warning: joint_lift has no limit │
└───────────────────────────────────────────────────────────────────────────┘
```

**Live reload:** Watches the URDF file for changes (VS Code `workspace.createFileSystemWatcher`). Re-renders automatically when Danush or Azeem pushes a URDF update. No manual refresh.

**Validation:** Highlights malformed URDF — missing joint origins, undefined link references, missing inertial (Nav2 requires inertial for all non-fixed links).

---

## 5. Module 3 — Nav2 Parameter Editor

### Why Nav2 YAML is Painful

Nav2 has 200+ parameters across 8 nodes (costmap, planner, controller, bt_navigator, etc.). The YAML file is 300+ lines. Without documentation in the editor, tuning is guesswork.

### Virtus Nav2 Parameter Schema

```typescript
// nav2/Nav2ParamSchema.ts
export const NAV2_SCHEMA: ParamGroup[] = [
  {
    group: 'controller_server',
    label: 'Controller (Pure Pursuit / DWB)',
    params: [
      { key: 'controller_server.ros__parameters.controller_frequency',
        label: 'Control Frequency (Hz)',
        type: 'float', default: 20.0, min: 5.0, max: 50.0,
        description: 'How often the controller computes velocity commands. Higher = smoother but more CPU.' },
      { key: 'controller_server.ros__parameters.FollowPath.max_vel_x',
        label: 'Max Linear Velocity (m/s)',
        type: 'float', default: 0.5, min: 0.1, max: 1.5,
        description: 'Maximum forward speed. Virtus nominal: 0.5 m/s. Reduce in crowded areas.' },
      { key: 'controller_server.ros__parameters.FollowPath.min_vel_x',
        label: 'Min Linear Velocity (m/s)',
        type: 'float', default: 0.1, min: 0.0, max: 0.3,
        description: 'Minimum speed while moving. Too low causes stalling.' },
      // ... 50+ more params with descriptions
    ]
  },
  {
    group: 'local_costmap',
    label: 'Local Costmap',
    params: [
      { key: 'local_costmap.local_costmap.ros__parameters.width',
        label: 'Width (m)',
        type: 'float', default: 3.0,
        description: 'Local costmap width. Should be 3-4x robot width for adequate lookahead.' },
      { key: 'local_costmap.local_costmap.ros__parameters.resolution',
        label: 'Resolution (m/cell)',
        type: 'float', default: 0.05, min: 0.02, max: 0.2,
        description: 'Smaller = more precise but heavier CPU. 0.05m standard for indoor robots.' },
      // ...
    ]
  },
  // global_costmap, planner_server, bt_navigator, ...
];
```

### Nav2 Editor UI

```
┌── Nav2 Parameters ───────────────────────────────────────────────────────┐
│  File: config/nav2_params.yaml  [● Unsaved changes]  [Save] [Reset]     │
│                                                                           │
│  ▼ Controller (Pure Pursuit / DWB)                                       │
│    Control Frequency        [20.0] Hz                                    │
│    ℹ How often controller computes velocity. Higher = smoother, more CPU │
│                                                                           │
│    Max Linear Velocity      [0.5 ] m/s  ████████░░  (0.1–1.5)           │
│    ℹ Virtus nominal 0.5m/s. Reduce in crowded areas.                    │
│                                                                           │
│    Min Linear Velocity      [0.1 ] m/s  ██░░░░░░░░  (0.0–0.3)           │
│                                                                           │
│  ▶ Local Costmap                                                          │
│  ▶ Global Costmap                                                         │
│  ▶ Planner (NavFn / Smac)                                                 │
│  ▶ BT Navigator                                                           │
│                                                                           │
│  [Apply & Restart Nav2]                                                   │
└───────────────────────────────────────────────────────────────────────────┘
```

**Apply & Restart Nav2:** Writes the modified YAML, then sends SIGTERM to the Nav2 process tracked by ProfileRunner, waits 2s, and re-launches just the Nav2 step. No need to restart Gazebo.

**Parameter diff:** Shows what changed from the committed YAML in git — so you know exactly what you've tuned.

---

## 6. Module 4 — Bag File Manager

### Bag Recording

```typescript
// bags/BagRecorder.ts
export class BagRecorder {
  start(topics: string[], name?: string): void {
    const bagName = name ?? `bags/test_${new Date().toISOString().replace(/[:.]/g, '-')}`;
    const topicArgs = topics.length ? topics.join(' ') : '-a';
    this.process = spawn('ros2', ['bag', 'record', topicArgs, '-o', bagName],
      PlatformUtils.spawnOpts(workspaceRoot));
  }

  stop(): void { this.process?.kill('SIGINT'); }
}

// Pre-defined recording presets for Virtus
export const BAG_PRESETS = [
  { label: 'All Topics',     topics: [] },   // -a
  { label: 'Nav2 Debug',     topics: ['/scan', '/scan/processed', '/cmd_vel', '/odom', '/tf', '/map', '/costmap_raw'] },
  { label: 'Sensor Fusion',  topics: ['/scan', '/sensor_fusion', '/esp32_bridge/rx', '/esp32_bridge/tx'] },
  { label: 'AI Interaction', topics: ['/ai_assistant/query', '/ai_assistant/response', '/orchestrator/state'] },
];
```

### Bag Browser UI

```
┌── Bag Files ─────────────────────────────────────────────────────────────┐
│  [● Record: Nav2 Debug ▼]  [■ Stop]                Recording: 00:01:23  │
│                                                                           │
│  bags/                                                                    │
│  ├── test_2026-03-18_nav2_tuning.mcap     2.3 GB  45min  [▶][✂][🗑]    │
│  │      Topics: 14  Duration: 45:12  Annotations: 3                      │
│  ├── test_2026-03-17_obstacle_test.mcap   890 MB  18min  [▶][✂][🗑]    │
│  │      Topics: 8   Duration: 18:34  Annotations: 1                      │
│  └── test_2026-03-15_baseline.mcap        120 MB   4min  [▶][✂][🗑]    │
│         Topics: 14  Duration: 04:01  Annotations: 0                      │
│                                                                           │
│  [▶ Play]  Rate: [1.0x ▼]  Start: [00:00] End: [45:12]                 │
└───────────────────────────────────────────────────────────────────────────┘
```

**Annotations:** While recording or reviewing, add timestamped notes: "obstacle placed at 02:14", "robot got stuck at 03:45". Stored as a sidecar `.annotation.json` file. Shown as vertical lines on the playback timeline.

**Trim:** Select start/end time → creates a new trimmed bag. Uses `ros2 bag convert` under the hood.

---

## 7. Module 5 — Test Scenario Runner

### Concept

Define repeatable test scenarios as JSON files. A scenario specifies: robot start pose, obstacle positions and sizes, simulated passenger positions, and a sequence of events. The runner injects these into the running Gazebo simulation.

### Scenario Format

```json
{
  "name": "Narrow Corridor Obstacle",
  "description": "Robot must navigate past a stationary suitcase in a 1.5m wide corridor",
  "robot_start": { "x": 0.0, "y": 0.0, "yaw": 0.0 },
  "robot_goal":  { "x": 5.0, "y": 0.0, "yaw": 0.0 },
  "obstacles": [
    { "type": "box", "x": 2.5, "y": 0.3, "z": 0.5, "w": 0.6, "d": 0.4, "h": 0.8, "label": "suitcase" }
  ],
  "passengers": [
    { "x": 4.0, "y": 0.0, "behavior": "standing" }
  ],
  "events": [
    { "at_time_s": 10, "action": "spawn_obstacle", "params": { "x": 3.5, "y": 0.0, "label": "person_crossing" } }
  ],
  "success_criteria": {
    "reach_goal": true,
    "max_time_s": 30,
    "no_collision": true
  }
}
```

### Scenario Canvas (2D Top-Down Editor)

```
┌── Scenario Editor ───────────────────────────────────────────────────────┐
│  [New] [Load] [Save]  Name: [Narrow Corridor Obstacle]                   │
│                                                                           │
│  ┌── 2D Canvas ────────────────────────────────────────────────────────┐ │
│  │  ···················································                │ │
│  │  ·  [R]→→→→→→→→→→ [■] →→→→→→→→ [G]  ·            │ │
│  │  ·                                    ·            │ │
│  │  ···················································                │ │
│  │  [R]=Robot start  [G]=Goal  [■]=Obstacle [P]=Passenger             │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  [▶ Run Scenario]  [Results: ✓ Success — 14.3s, no collision]           │
└───────────────────────────────────────────────────────────────────────────┘
```

Obstacles and passengers are draggable on the 2D canvas. The runner uses `gazebo_ros` service calls to spawn/despawn entities.

---

## 8. Module 6 — World Manager

Browses all `.world` files in the workspace, shows a thumbnail (generated via Gazebo screenshot), and allows one-click world switching in the active Gazebo session:

```
┌── Gazebo Worlds ─────────────────────────────────────────────────────────┐
│  ● Active: airport.world                                                  │
│                                                                           │
│  [airport.world]        Airport terminal with gates A1–C12               │
│  [empty.world]          Flat plane — fast startup for unit tests          │
│  [corridor_test.world]  Single 10m corridor for nav testing              │
│  [obstacles.world]      Random obstacle field                             │
│                                                                           │
│  [Switch World]  (requires Gazebo restart)                               │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Message Protocol

```typescript
type WebviewMessage =
  | { type: 'launchProfile';     payload: { profileId: string } }
  | { type: 'stopAll';           payload: void }
  | { type: 'stopProcess';       payload: { processId: string } }
  | { type: 'getURDF';           payload: { path: string } }
  | { type: 'saveNav2Params';    payload: { params: Record<string, any> } }
  | { type: 'startRecording';    payload: { preset: string; name?: string } }
  | { type: 'stopRecording';     payload: void }
  | { type: 'playBag';           payload: { path: string; rate: number; start_s: number } }
  | { type: 'stopBag';           payload: void }
  | { type: 'runScenario';       payload: { scenario: Scenario } }

type HostMessage =
  | { type: 'processStatus';     payload: ProcessStatus[] }
  | { type: 'urdfParsed';        payload: { links: URDFLink[]; joints: URDFJoint[] } }
  | { type: 'urdfValidation';    payload: URDFValidationResult }
  | { type: 'nav2ParamsCurrent'; payload: Record<string, any> }
  | { type: 'bagList';           payload: BagFile[] }
  | { type: 'recordingStatus';   payload: { recording: boolean; duration_s: number; size_mb: number } }
  | { type: 'scenarioResult';    payload: ScenarioResult }
```

---

## 10. Windows & Cross-Platform Support

ROS 2 on Windows requires WSL2. All simulation features run inside WSL2:

```typescript
function launchCmd(cmd: string): string {
  return Platform.isWindows ? `wsl bash -c "${cmd}"` : cmd;
}
```

URDF file path on Windows is converted to WSL path for parsing. The URDF viewer (Three.js) runs entirely in the webview — no Linux dependency.

Nav2 YAML editing writes to the workspace filesystem — works natively on Windows (file is on the Windows FS, WSL2 mounts it as `/mnt/c/...`).

---

## 11. Production Quality Standards

- **Process leak prevention:** All spawned processes are tracked in `ProcessTracker`. On VS Code shutdown (`deactivate()` event), all processes are killed. No zombie Gazebo processes.
- **Delay sequencing:** Steps with `delay_ms` use an async sleep — not a fixed timer. The next step starts only after the delay AND the previous process has emitted stdout indicating readiness (e.g., Gazebo emits "Gazebo multi-robot simulator" on ready).
- **URDF hot reload:** File watcher debounced at 500ms — rapid saves don't trigger multiple re-parses.
- **Nav2 param validation:** Before writing YAML, validate all values are within schema bounds. Show errors inline; refuse to write invalid params.
- **Bag disk space warning:** Show warning when available disk < 5GB. Bags can fill disks fast (full topic recording: ~50MB/min).

---

## 12. Phased Build Plan

### Phase 1 — Launch Profile Manager (Weeks 1–2)
- 4 built-in profiles with step sequencing + delays
- Process tracker with status indicators
- Stop All + stop individual process
- **Deliverable:** One-click simulation launch

### Phase 2 — URDF Preview (Week 3)
- URDF parser for Virtus robot
- Three.js renderer with solid/wireframe/joint modes
- File watcher for live reload
- URDF validation warnings
- **Deliverable:** Live 3D URDF preview without Gazebo

### Phase 3 — Nav2 Parameter Editor (Week 4)
- Full Nav2 schema with descriptions for all key params
- Slider + numeric input UI with range enforcement
- YAML read/write
- Apply & Restart Nav2 (hot-reload)
- **Deliverable:** GUI Nav2 tuning without YAML hand-editing

### Phase 4 — Bag File Manager (Week 5)
- Bag scanner + browser UI
- Recording with presets
- Playback with rate control
- Annotations (sidecar JSON)
- **Deliverable:** Full bag workflow from VS Code

### Phase 5 — Scenario Runner (Week 6)
- Scenario JSON format
- 2D canvas editor
- Gazebo entity spawning via service calls
- Success/failure result display
- **Deliverable:** Repeatable test scenarios

### Phase 6 — World Manager + Polish (Week 7)
- World file browser
- Windows WSL2 support
- Full process cleanup on deactivate
- **Deliverable:** Production-ready extension

---

## 13. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| 3D URDF rendering | Three.js | Runs in webview, no native deps |
| URDF parsing | DOMParser (browser API) | XML parsing in webview, no deps |
| Process management | `child_process.spawn` | ROS 2 launch commands |
| YAML read/write | `js-yaml` | Well-maintained, round-trip safe |
| 2D scenario canvas | React + SVG | Simple, no extra canvas lib needed |
| File watching | `vscode.workspace.createFileSystemWatcher` | Native VS Code API |
| Bag metadata | `ros2 bag info` CLI | Standard ROS 2 tooling |

---

## 14. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Gazebo startup time varies — step delay too short | High | Medium | Detect readiness via stdout pattern, not fixed delay |
| URDF uses mesh files (.STL/.DAE) not basic shapes | High | High | Three.js STL/Collada loaders; resolve mesh paths relative to package |
| Nav2 param schema changes between Jazzy and future | Medium | Medium | Schema versioned; warn if param not found in loaded YAML |
| Bag files >10GB cause UI freeze during scan | Low | Medium | Lazy-load bag metadata; scan runs in Python subprocess |
| Gazebo service calls for scenario injection require specific plugins | Medium | High | Document required Gazebo plugins; provide setup script |
| Windows WSL2 process management complex | Medium | Medium | All process ops go through `WslBridge`; test matrix includes Windows |
