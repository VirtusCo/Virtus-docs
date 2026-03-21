# VirtusCo Docs

**Building the future of airport autonomy.**

VirtusCo builds Porter — an autonomous luggage-carrying robot for airports. It navigates terminals using LIDAR, carries passenger luggage, and provides an on-device AI assistant — all running locally on a Raspberry Pi 5.

<div class="grid cards" markdown>

-   :material-robot:{ .lg .middle } **Getting Started**

    ---

    Set up your Porter Robot development environment

    [:octicons-arrow-right-24: Get started](getting-started/overview.md)

-   :material-sitemap:{ .lg .middle } **Architecture**

    ---

    System design, ROS 2 topics, ESP32 protocol, AI pipeline

    [:octicons-arrow-right-24: Learn more](architecture/system.md)

-   :material-puzzle:{ .lg .middle } **VS Code Extensions**

    ---

    8 extensions for the full robotics dev lifecycle

    [:octicons-arrow-right-24: Explore](extensions/index.md)

-   :material-api:{ .lg .middle } **Infrastructure**

    ---

    SDK, fleet backend, configuration management

    [:octicons-arrow-right-24: View](infrastructure/sdk.md)

</div>

## Quick Links

| Repository | Description |
|:-----------|:------------|
| [Porter-ROS](https://github.com/VirtusCo/Porter-ROS) | Robot software — ROS 2, ESP32 firmware, AI, Flutter GUI |
| [Virtusco-extensions](https://github.com/VirtusCo/Virtusco-extensions) | 8 VS Code extensions |
| [Virtus-sdk](https://github.com/VirtusCo/Virtus-sdk) | REST API + Python/Dart clients |
| [Virtus-fleet](https://github.com/VirtusCo/Virtus-fleet) | Rust fleet management backend |
| [ydlidar-ros2-driver](https://github.com/VirtusCo/ydlidar-ros2-driver) | Open-source LIDAR driver (Apache 2.0) |

## Tech Stack

```
Raspberry Pi 5 (Master) — ROS 2 Jazzy, Nav2, Virtue AI, Touchscreen
├── YDLIDAR X4 Pro 360° (USB serial, 128000 baud)
├── ESP32 #1 (USB CDC) — Motor controller, BTS7960 H-bridge
└── ESP32 #2 (USB CDC) — Sensor fusion: ToF + Ultrasonic + Microwave
```

## Design Principles

- **Edge-first** — everything runs locally, no cloud, no latency
- **Real-time** — motor control at 100Hz, sensor fusion at 50Hz
- **Fail-safe** — 9-state FSM with graceful degradation
- **Modular** — each component is a ROS 2 node
- **Production-grade** — 178+ tests, CI on every push, automated releases
