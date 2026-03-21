# ESP32 Binary Protocol

All communication between the Raspberry Pi and the two ESP32 microcontrollers uses a custom binary protocol over USB CDC Serial.

## Frame Format

```
┌────────┬────────┬─────────┬─────────────┬──────────┐
│ Header │ Length │ Command │   Payload   │  CRC16   │
│ 2 bytes│ 1 byte │ 1 byte  │  N bytes    │ 2 bytes  │
│ 0xAA55 │        │         │             │ CCITT    │
└────────┴────────┴─────────┴─────────────┴──────────┘
```

| Field | Size | Description |
|-------|------|-------------|
| **Header** | 2 bytes | Magic bytes `0xAA 0x55` — frame synchronization |
| **Length** | 1 byte | Length of Command + Payload (excluding header and CRC) |
| **Command** | 1 byte | Command identifier (see table below) |
| **Payload** | 0--250 bytes | Command-specific data |
| **CRC16** | 2 bytes | CRC16-CCITT over Length + Command + Payload |

## CRC16-CCITT

- **Polynomial:** 0x1021
- **Initial value:** 0xFFFF
- **Implementation:** 256-byte lookup table for performance
- **Scope:** Computed over the Length, Command, and Payload fields

## Parser State Machine

The protocol parser uses a 9-state byte-by-byte state machine:

```
WAIT_HEADER_1 -> WAIT_HEADER_2 -> WAIT_LENGTH -> WAIT_COMMAND
     -> WAIT_PAYLOAD -> WAIT_CRC_HIGH -> WAIT_CRC_LOW -> FRAME_COMPLETE
```

!!! warning "Error handling"
    Any invalid byte resets the parser to `WAIT_HEADER_1`. CRC mismatches trigger a NACK response. The parser is designed to recover from corrupted or partial frames without manual intervention.

## Motor Controller Commands (ESP32 #1)

| Command | ID | Payload | Direction |
|---------|----|---------|-----------|
| `CMD_VELOCITY` | `0x01` | `int16 linear_x, int16 angular_z` | RPi to ESP32 |
| `CMD_MOTOR_STATUS` | `0x02` | `uint8 state, int16 left_rpm, int16 right_rpm` | ESP32 to RPi |
| `CMD_HEARTBEAT` | `0x03` | (empty) | RPi to ESP32 |
| `CMD_HEARTBEAT_ACK` | `0x04` | (empty) | ESP32 to RPi |
| `CMD_ESTOP` | `0x05` | (empty) | RPi to ESP32 |
| `CMD_LIFT` | `0x06` | `uint8 position` | RPi to ESP32 |
| `CMD_ACK` | `0xFE` | `uint8 cmd_acked` | Both |
| `CMD_NACK` | `0xFF` | `uint8 cmd_nacked, uint8 error_code` | Both |

## Sensor Fusion Commands (ESP32 #2)

| Command | ID | Payload | Direction |
|---------|----|---------|-----------|
| `CMD_SENSOR_DATA` | `0x10` | `float tof, float ultra, float micro, float fused` | ESP32 to RPi |
| `CMD_SENSOR_CONFIG` | `0x11` | `float threshold, uint8 enable_mask` | RPi to ESP32 |
| `CMD_SENSOR_STATUS` | `0x12` | `uint8 state, uint8 sensor_health` | ESP32 to RPi |

## Heartbeat Watchdog

The motor controller ESP32 expects a heartbeat frame (`CMD_HEARTBEAT`) every **500 ms**. If no heartbeat is received within the timeout window, the watchdog fires and the motors stop immediately. This is a deliberate safety mechanism.

!!! danger "Bridge must be C++"
    The heartbeat timer requires sub-millisecond callback latency. Python's GIL cannot guarantee this under load. The `esp32_motor_bridge` node must remain in C++. See the [Language Migration Plan](../plans/migration.md) for details.

## Transport Abstraction

The firmware transport layer supports three backends selected via Zephyr Kconfig:

| Backend | Kconfig | Use Case |
|---------|---------|----------|
| **UART** | `CONFIG_TRANSPORT_UART` | ESP32-WROOM via USB-UART bridge |
| **CDC ACM** | `CONFIG_TRANSPORT_CDC_ACM` | ESP32-S2/S3 native USB |
| **Mock** | `CONFIG_TRANSPORT_MOCK` | Unit testing on `native_sim` |
