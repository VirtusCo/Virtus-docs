---
hide:
  - toc
---

# VirtusCo Docs

**Building the future of airport autonomy.**

VirtusCo builds Porter — an autonomous luggage-carrying robot for airports. It navigates terminals using LIDAR, carries passenger luggage, and provides an on-device AI assistant — all running locally on a Raspberry Pi 5. Zero cloud. Zero latency. Full autonomy.

[Get started](getting-started/overview.md){ .md-button .md-button--primary }
[View on GitHub](https://github.com/VirtusCo){ .md-button }

---

<div class="grid cards" markdown>

-   :material-robot-industrial:{ .lg .middle } **Getting Started**

    ---

    Set up your Porter Robot development environment — hardware, software, Docker.

    [:octicons-arrow-right-24: Get started](getting-started/overview.md)

-   :material-sitemap:{ .lg .middle } **Architecture**

    ---

    System design, ROS 2 topic flow, ESP32 binary protocol, AI inference pipeline.

    [:octicons-arrow-right-24: Learn more](architecture/system.md)

-   :material-puzzle:{ .lg .middle } **VS Code Extensions**

    ---

    8 extensions for firmware, AI/ML, ROS 2, simulation, PCB, and hardware telemetry.

    [:octicons-arrow-right-24: Explore](extensions/index.md)

-   :material-api:{ .lg .middle } **Infrastructure**

    ---

    REST API + SDK, Rust fleet backend, deployment configuration management.

    [:octicons-arrow-right-24: View](infrastructure/sdk.md)

</div>

---

## Repositories

| Repository | Description | License |
|:-----------|:------------|:--------|
| [Porter-ROS](https://github.com/VirtusCo/Porter-ROS) | Robot software — ROS 2, ESP32 firmware, AI, Flutter GUI | Proprietary |
| [Virtusco-extensions](https://github.com/VirtusCo/Virtusco-extensions) | 8 VS Code extensions for robotics development | Proprietary |
| [ydlidar-ros2-driver](https://github.com/VirtusCo/ydlidar-ros2-driver) | Production-grade YDLIDAR ROS 2 Jazzy driver | Apache 2.0 |
| [Virtus-sdk](https://github.com/VirtusCo/Virtus-sdk) | REST API + Python/Dart client libraries | Proprietary |
| [Virtus-fleet](https://github.com/VirtusCo/Virtus-fleet) | Rust fleet management backend | Proprietary |
| [Virtus-configs](https://github.com/VirtusCo/Virtus-configs) | Deployment profiles + configuration management | Proprietary |

## Tech Stack

```
Raspberry Pi 5 (Master) ── ROS 2 Jazzy, Nav2, Virtue AI, Touchscreen
 ├── YDLIDAR X4 Pro 360  (USB serial, 128000 baud)
 ├── ESP32 #1 (USB CDC)  Motor controller, BTS7960 H-bridge, differential drive
 └── ESP32 #2 (USB CDC)  Sensor fusion: ToF + Ultrasonic + Microwave (Kalman)
```

## Design Principles

| Principle | Description |
|:----------|:------------|
| **Edge-first** | Everything runs locally. No cloud. No latency. No data leaves the device. |
| **Real-time** | Motor control at 100Hz. Sensor fusion at 50Hz. Watchdogs everywhere. |
| **Fail-safe** | 9-state FSM with graceful degradation. Hardware watchdogs. CRC validation. |
| **Modular** | Each component is a ROS 2 node. Swap, upgrade, or disable independently. |
| **Production-grade** | 178+ tests. CI on every push. Automated releases. |
