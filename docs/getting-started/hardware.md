# Hardware Bill of Materials

This page lists every component on the Porter robot and how they connect.

## Component Inventory

| Component | Specification | Qty | Role |
|-----------|--------------|-----|------|
| **Raspberry Pi 5** | 4 GB+ RAM, Cortex-A76 @ 2.4 GHz | 1 | Master compute — ROS 2, Nav2, SLAM, AI, display |
| **YDLIDAR X4 Pro** | 360 scanner, USB serial, 128000 baud | 1 | Primary navigation sensor |
| **ESP32 DevKitC WROOM** | Dual-core 240 MHz, Zephyr RTOS | 2 | Motor control + sensor fusion |
| **BTS7960** | 43A dual H-bridge motor driver | 2 | Drive motor power stage |
| **DC Motor** | 12V, 100 RPM, 40 Kgcm torque | 2 | Differential drive (left + right) |
| **VL53L0x ToF** | Time-of-Flight, 0.02--4 m range | 1+ | Short-range obstacle detection |
| **HC-SR04 Ultrasonic** | 0.02--4 m range | 1+ | Mid-range obstacle detection |
| **RCWL-0516 Microwave** | 1--7 m range, motion/presence | 1+ | Through-material detection |
| **Touchscreen Display** | HDMI or DSI, connected to RPi | 1 | Passenger interaction (Flutter GUI) |
| **Lifting Mechanism** | Motor/actuator | 1 | Luggage height adjustment |

## Drive System

- **Type:** Differential drive (2 powered wheels + passive casters)
- **Motors:** 2x 12V DC, 100 RPM, 40 Kgcm torque
- **Drivers:** 2x BTS7960 dual H-bridge (43A capable)
- **Encoders:** Planned for future odometry feedback

## Sensor Fusion Strategy

The three environment sensors serve complementary roles and are fused via a Kalman filter on ESP32 #2:

| Sensor | Best At | Weakness | Range |
|--------|---------|----------|-------|
| **ToF** | Precise short-range distance | Narrow FOV, ambient light sensitive | 0.02--4 m |
| **Ultrasonic** | Mid-range, works in any lighting | Soft surfaces absorb sound | 0.02--4 m |
| **Microwave** | Detects through materials, motion | Binary presence, not precise distance | 1--7 m |

## Wiring Overview

```
Raspberry Pi 5
├── USB Serial ──► YDLIDAR X4 Pro (direct, /dev/ttyUSB0)
├── USB CDC   ──► ESP32 #1 — Motor Controller (/dev/esp32_motors)
│                   ├── BTS7960 #1 ──► Left Motor (PWM)
│                   └── BTS7960 #2 ──► Right Motor (PWM)
└── USB CDC   ──► ESP32 #2 — Sensor Fusion (/dev/esp32_sensors)
                    ├── VL53L0x ToF (I2C)
                    ├── HC-SR04 Ultrasonic (GPIO trigger/echo)
                    └── RCWL-0516 Microwave (ADC)
```

!!! warning "Stable device names"
    Use the provided udev rules to assign stable names (`/dev/esp32_motors`, `/dev/esp32_sensors`). Without them, USB device enumeration order is not guaranteed across reboots.
