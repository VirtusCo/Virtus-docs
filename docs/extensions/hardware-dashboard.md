# Virtus Hardware Dashboard

Live hardware telemetry for the Porter robot. Monitors power rails, motor current, sensor readings, and system health from within VS Code.

## Features

### Power Monitoring

- **Voltage rails** — Real-time 5V (RPi), 3.3V (ESP32), 12V (motors) readings
- **Current draw** — Per-motor current via BTS7960 IS pins
- **Battery state** — Voltage, estimated capacity, charge rate
- **Power events** — Surge, brownout, and overcurrent event log with timestamps

### Motor Telemetry

| Metric | Source | Update Rate |
|--------|--------|-------------|
| Left motor RPM | ESP32 #1 | 50 Hz |
| Right motor RPM | ESP32 #1 | 50 Hz |
| Left motor current | BTS7960 IS pin | 50 Hz |
| Right motor current | BTS7960 IS pin | 50 Hz |
| Motor temperature | Estimated from current | 1 Hz |

### Sensor Readings

- **ToF distance** — VL53L0x range in millimeters
- **Ultrasonic distance** — HC-SR04 range
- **Microwave presence** — RCWL-0516 detection state
- **Fused estimate** — Kalman filter output from ESP32 #2

### Threshold Alerts

Configure alert thresholds for any metric. Alerts trigger VS Code notifications and are logged to the power event log.

```yaml
alerts:
  battery_voltage_low: 11.0V
  motor_current_high: 15A
  motor_temperature_high: 70C
  sensor_timeout: 2000ms
```

### Schematic Cross-Reference

Click any sensor or actuator in the dashboard to jump to its location in the KiCad schematic (requires [PCB Studio](pcb-studio.md)).

!!! tip "Data source"
    The dashboard reads telemetry from the `esp32_motor_bridge` and `esp32_sensor_bridge` ROS 2 nodes via topic subscription. The robot must be running for live data.
