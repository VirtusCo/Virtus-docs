# System Architecture

Porter's software and hardware form a layered stack running across three compute units: a Raspberry Pi 5 (master) and two ESP32 microcontrollers (slaves).

## Hardware Stack

```
Raspberry Pi 5 (Master) — ROS 2 Jazzy, Nav2, Virtue AI, Touchscreen
├── YDLIDAR X4 Pro 360 (USB serial, 128000 baud)
├── ESP32 #1 (USB CDC) — Motor controller, BTS7960 H-bridge, differential drive
└── ESP32 #2 (USB CDC) — Sensor fusion: ToF + Ultrasonic + Microwave (Kalman)
```

## Compute Hierarchy

```
┌──────────────────────────────────────────────────────┐
│           Raspberry Pi 5 (Master)                    │
│                                                      │
│  ROS 2 Jazzy (Docker)          Virtue AI (Qwen 2.5) │
│  Nav2 navigation stack         Flutter GUI           │
│  YDLIDAR driver (C++17)        AI HTTP server        │
│  Lidar processor               State machine (FSM)   │
│                                                      │
│       ┌── USB CDC ──┐    ┌── USB CDC ──┐             │
│       v             │    v             │             │
│  ┌──────────┐  ┌──────────┐                          │
│  │ ESP32 #1 │  │ ESP32 #2 │                          │
│  │ (Motors) │  │ (Sensors)│                          │
│  │          │  │          │                          │
│  │ Zephyr   │  │ Zephyr   │                          │
│  │ RTOS     │  │ RTOS     │                          │
│  │          │  │          │                          │
│  │ BTS7960  │  │ ToF      │                          │
│  │ PWM ctrl │  │ Ultrason │                          │
│  │ Watchdog │  │ Microwave│                          │
│  └──────────┘  └──────────┘                          │
└──────────────────────────────────────────────────────┘
```

## Software Layer Stack

```
┌─────────────────────────────────────────┐
│           Interactive Display           │  Flutter touchscreen UI
├─────────────────────────────────────────┤
│         Porter Orchestrator             │  9-state FSM, health monitoring
├─────────────────────────────────────────┤
│            Nav2 Stack                   │  Path planning, obstacle avoidance
├─────────────────────────────────────────┤
│      Porter LIDAR Processor             │  6-stage scan filter pipeline
├────────────────┬────────────────────────┤
│  YDLIDAR Driver│  ESP32 Bridge Nodes    │  C++17 serial drivers
├────────────────┴────────────────────────┤
│          Docker + OTA Layer             │  Container management
├─────────────────────────────────────────┤
│     Raspberry Pi / Linux / Zephyr       │  OS + firmware
└─────────────────────────────────────────┘
```

## ROS 2 Packages

| Package | Language | Purpose |
|---------|----------|---------|
| `ydlidar_driver` | C++17 | LIDAR driver, publishes `/scan` + `/diagnostics` |
| `porter_lidar_processor` | Python | 6-stage scan filter pipeline |
| `porter_orchestrator` | Python | 9-state FSM + health monitor |
| `porter_esp32_bridge` | C++17 | Motor + sensor serial bridges |
| `porter_ai_assistant` | Python | Qwen 2.5 1.5B inference, 14 tools, RAG |
| `porter_gui` | Dart | Flutter touchscreen UI |

## Data Flow

```
YDLIDAR ──serial──> ydlidar_driver ──/scan──> lidar_processor ──/scan/processed──> Nav2
                                                                                    │
ESP32 #2 ──USB──> esp32_sensor_bridge ──/environment──> Nav2 costmap                │
                                                                                    │
                                     Nav2 ──/cmd_vel──> esp32_motor_bridge ──USB──> ESP32 #1
```

## Communication

All RPi-to-ESP32 communication uses USB CDC Serial with a custom binary protocol. See the [Protocol](protocol.md) page for frame format details.

| Link | Direction | Data |
|------|-----------|------|
| LIDAR to RPi | LIDAR to RPi | Raw scan data (vendor protocol) |
| ESP32 #2 to RPi | Bidirectional | Fused sensor data / config commands |
| RPi to ESP32 #1 | Bidirectional | Velocity commands / motor status |
