# Virtus Firmware Builder

A visual, node-based firmware development tool for ESP32 and Zephyr RTOS. Drag and drop hardware peripheral nodes onto a React Flow canvas to generate complete firmware projects — DTS overlays, `prj.conf`, and C source code.

## Features

- **Visual canvas** — React Flow node graph where each node represents a Zephyr peripheral (GPIO, SPI, I2C, PWM, ADC, UART, etc.)
- **Auto-generated code** — Connecting nodes generates valid Zephyr C source, device tree overlays, and Kconfig settings
- **Zephyr API scanner** — Parses Zephyr SDK headers to auto-generate node types for any supported peripheral
- **Board support** — Targets ESP32-WROOM, ESP32-S2, ESP32-S3, and custom boards via Kconfig
- **One-click build** — Invokes `west build` from within VS Code
- **Pin conflict detection** — Warns when two peripherals share the same GPIO pin

## How It Works

```
┌──────────────────────────────────┐
│     React Flow Canvas (Webview)  │
│                                  │
│  [GPIO Node] ──> [PWM Node]     │
│  [I2C Node]  ──> [Sensor Node]  │
│  [UART Node] ──> [Protocol]     │
│                                  │
└──────────┬───────────────────────┘
           │ Node graph JSON
           v
┌──────────────────────────────────┐
│     Code Generator (Extension)   │
│                                  │
│  ├── boards/overlay.dts          │
│  ├── prj.conf                    │
│  └── src/main.c                  │
└──────────────────────────────────┘
```

## Usage

1. Open the command palette and run **Virtus: Open Firmware Builder**
2. Select a target board from the dropdown
3. Drag peripheral nodes from the sidebar onto the canvas
4. Connect nodes to define data flow (e.g., I2C bus to sensor to processing)
5. Configure each node's properties in the inspector panel (pin assignments, frequencies, etc.)
6. Click **Generate** to produce the firmware project
7. Click **Build** to invoke `west build`

## Supported Node Types

| Category | Nodes |
|----------|-------|
| **Bus** | UART, SPI, I2C, USB CDC ACM |
| **Peripherals** | GPIO, PWM, ADC, DAC, Timer |
| **Sensors** | VL53L0x (ToF), HC-SR04, RCWL-0516, BME280, MPU6050 |
| **Motor Drivers** | BTS7960, L298N, DRV8833 |
| **Communication** | Binary Protocol, CRC16, Transport Layer |
| **Logic** | State Machine (SMF), Kalman Filter, PID Controller |

!!! info "Custom nodes"
    The Zephyr API scanner can generate new node types from any Zephyr driver header. Run **Virtus: Scan Zephyr Headers** to update the node library.
