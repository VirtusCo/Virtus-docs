# Virtus ROS 2 Studio

A Porter-specific ROS 2 development environment providing live topic monitoring, node graph visualization, state machine debugging, and visual launch file building.

## Features

### Live Topic Monitor

Real-time display of all ROS 2 topics with message previews, publish rates, and subscriber counts. Filter by package or topic pattern.

### Node Graph

Interactive visualization of the ROS 2 computation graph showing all active nodes, their topic connections, and service endpoints. Nodes are color-coded by package.

```
[ydlidar_driver] ──/scan──> [porter_lidar_processor] ──/scan/processed──> [Nav2]
                                                                            |
                                                          /cmd_vel <────────┘
                                                            |
                                                [esp32_motor_bridge] ──serial──> ESP32
```

### 9-State FSM Viewer

Visual debugger for Porter's orchestrator state machine. Displays the current state, transition history, and health conditions in real time.

| State | Description |
|-------|-------------|
| `INITIALISING` | System boot, loading parameters |
| `DRIVER_STARTING` | LIDAR driver launching |
| `HEALTH_CHECK` | Verifying sensor health |
| `PROCESSOR_STARTING` | Scan filter pipeline starting |
| `READY` | Fully operational |
| `NAVIGATING` | Actively following a path |
| `DEGRADED` | Operating with reduced capability |
| `ERROR` | Fault detected, attempting recovery |
| `SHUTDOWN` | Graceful shutdown |

### ESP32 CRC16 Bridge Debugger

Inspect raw serial frames between RPi and ESP32 in real time. Decodes the binary protocol, validates CRC16 checksums, and highlights malformed frames.

### Launch File Builder

Visual n8n-style builder for composing ROS 2 launch files. Drag nodes representing packages, configure parameters, and export as Python launch files.

!!! note "Revisit items"
    Launch code generation, Browse buttons for file paths, and live ROS 2 testing integration are planned for a future update.
