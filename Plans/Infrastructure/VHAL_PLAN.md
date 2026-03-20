# Virtus Hardware Abstraction Layer (VHAL)
### Detailed Development Plan
**Project:** Zephyr RTOS application-level hardware abstraction — standard driver interfaces for all sensors and actuators on Virtus robot  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026  
**Build Priority:** #4 — Build when next hardware revision is planned

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [Why VHAL Exists](#2-why-vhal-exists)
3. [Architecture](#3-architecture)
4. [Sensor Interface](#4-sensor-interface)
5. [Actuator Interface](#5-actuator-interface)
6. [Communication Interface](#6-communication-interface)
7. [Driver Implementations](#7-driver-implementations)
8. [Driver Registry](#8-driver-registry)
9. [Health Monitoring](#9-health-monitoring)
10. [Error Handling](#10-error-handling)
11. [Testing Strategy](#11-testing-strategy)
12. [Integration with Zephyr Driver Model](#12-integration-with-zephyr-driver-model)
13. [Phased Build Plan](#13-phased-build-plan)
14. [Tech Stack Summary](#14-tech-stack-summary)
15. [Risk Register](#15-risk-register)

---

## 1. Overview & Vision

### What VHAL Is

A thin application-level hardware abstraction layer sitting **above** Zephyr's native driver model and **below** the application logic in `motor_controller` and `sensor_fusion`. It defines a standard C interface that every sensor and actuator on Virtus implements — so swapping hardware is a driver swap, not an architecture redesign.

```
Application Logic (motor_controller, sensor_fusion)
        ↓ calls virtus_hal interfaces
VHAL (virtus_hal.h) — standard interfaces
        ↓ implemented by
Hardware Drivers (bts7960.c, vl53l0x.c, hcsr04.c, rcwl.c, ...)
        ↓ use
Zephyr Native Driver Model (GPIO, PWM, I2C, UART, ADC APIs)
        ↓ maps to
Physical Hardware
```

### What VHAL Is Not

VHAL does not replace Zephyr's driver model — it adds a Virtusco-specific layer on top of it for application-level consistency. It does not abstract across operating systems — it is Zephyr/ESP32 specific.

### The Hardware Revision Problem

Virtus is a hardware startup. Hardware will change. Planned changes already known:
- VL53L0X → VL53L5CX (wider FOV, 8x8 zone ToF)
- HC-SR04 → JSN-SR04T (waterproof, for outdoor terminals)
- BTS7960 → possible replacement with dedicated motor driver PCB (Danush's next revision)
- Adding encoders to motors (for odometry)
- Adding a weighing load cell

Without VHAL, each of these changes requires Antony to understand and rewrite the sensor fusion logic and motor controller logic. With VHAL, Danush implements a new driver, registers it, and the application layer is untouched.

---

## 2. Why VHAL Exists

### Current State (No VHAL)

```c
// Current sensor_fusion/src/main.c — tightly coupled to specific hardware
#include "vl53l0x.h"   // Direct hardware library
#include "hcsr04.h"    // Direct hardware library

void sensor_task(void *a, void *b, void *c) {
    vl53l0x_init();           // Hardware-specific init
    hcsr04_init(TRIG_PIN, ECHO_PIN);  // Hardware-specific init

    while (1) {
        uint16_t tof = vl53l0x_read_mm();    // Hardware-specific read
        uint16_t sonic = hcsr04_read_cm();   // Hardware-specific read
        // ... kalman filter
    }
}
```

If VL53L0X → VL53L5CX: rewrite `sensor_task` because the init and read APIs are completely different.

### Future State (With VHAL)

```c
// Future sensor_fusion/src/main.c — hardware-agnostic
#include "virtus_hal.h"   // The only hardware header needed

void sensor_task(void *a, void *b, void *c) {
    virtus_sensor_init_all();  // VHAL initializes all registered sensors

    while (1) {
        virtus_sensor_data_t data;
        virtus_sensor_read(SENSOR_TOF, &data);    // VHAL dispatches to whatever ToF driver is registered
        virtus_sensor_read(SENSOR_ULTRASONIC, &data);
        // ... kalman filter — unchanged regardless of hardware revision
    }
}
```

If VL53L0X → VL53L5CX: implement `vl53l5cx_driver.c`, register it as `SENSOR_TOF`, done.

---

## 3. Architecture

### File Structure

```
esp32_firmware/
└── common/
    └── hal/                                    ← VHAL lives here, shared by both ESP32s
        ├── include/
        │   └── virtus_hal.h                    ← The single public header — all application code uses this
        ├── src/
        │   ├── hal_core.c                      ← Driver registry + dispatch logic
        │   ├── hal_health.c                    ← Health monitoring for all registered drivers
        │   └── hal_error.c                     ← Unified error reporting
        ├── drivers/
        │   ├── sensors/
        │   │   ├── vl53l0x_driver.c            ← ToF sensor driver (current)
        │   │   ├── hcsr04_driver.c             ← Ultrasonic driver (current)
        │   │   └── rcwl0516_driver.c           ← Microwave driver (current)
        │   └── actuators/
        │       ├── bts7960_driver.c            ← Motor driver (current)
        │       └── relay_driver.c              ← Relay bank driver (current)
        └── tests/
            ├── test_hal_registry.c             ← Ztest: driver registration/lookup
            ├── test_hal_sensor_mock.c          ← Ztest: mock sensor driver
            └── test_hal_actuator_mock.c        ← Ztest: mock actuator driver
```

---

## 4. Sensor Interface

### Core Sensor Interface (virtus_hal.h)

```c
// virtus_hal.h — Sensor interface

// Sensor IDs — hardware-independent identifiers
typedef enum {
    SENSOR_TOF         = 0,  // Time-of-Flight distance sensor
    SENSOR_ULTRASONIC  = 1,  // Ultrasonic distance sensor
    SENSOR_MICROWAVE   = 2,  // Microwave presence sensor
    SENSOR_ENCODER_L   = 3,  // Left motor encoder (future)
    SENSOR_ENCODER_R   = 4,  // Right motor encoder (future)
    SENSOR_LOADCELL    = 5,  // Luggage weighing load cell (future)
    SENSOR_MAX         = 8   // Max sensors — increase if needed
} virtus_sensor_id_t;

// Unified sensor reading — all sensors write their output here
typedef struct {
    virtus_sensor_id_t  sensor_id;
    int64_t             timestamp_us;   // Zephyr k_uptime_get() in microseconds
    bool                valid;          // False if read failed or sensor unhealthy
    union {
        struct { uint16_t mm; }         tof;        // SENSOR_TOF
        struct { uint16_t cm; }         ultrasonic; // SENSOR_ULTRASONIC
        struct { bool detected; }       microwave;  // SENSOR_MICROWAVE
        struct { int32_t ticks; float rpm; } encoder; // SENSOR_ENCODER_*
        struct { float kg; float raw_mv; } loadcell; // SENSOR_LOADCELL
    };
} virtus_sensor_data_t;

// The interface every sensor driver must implement
typedef struct {
    virtus_sensor_id_t  id;
    const char         *name;           // "VL53L0X", "HC-SR04", etc.
    const char         *version;        // Driver version e.g. "1.0.0"

    // Lifecycle
    int  (*init)(void);                 // Returns 0 on success, negative errno on failure
    int  (*deinit)(void);
    bool (*is_ready)(void);             // True if sensor is initialized and responding

    // Reading
    int  (*read)(virtus_sensor_data_t *out);  // Returns 0 on success
    int  (*read_blocking)(virtus_sensor_data_t *out, k_timeout_t timeout);

    // Health
    bool (*is_healthy)(void);           // True if last N reads were valid
    void (*get_diagnostics)(char *buf, size_t len);  // Human-readable status
} virtus_sensor_driver_t;

// VHAL API — application code calls these, never the driver directly
int virtus_sensor_register(const virtus_sensor_driver_t *driver);
int virtus_sensor_init_all(void);
int virtus_sensor_read(virtus_sensor_id_t id, virtus_sensor_data_t *out);
bool virtus_sensor_is_healthy(virtus_sensor_id_t id);
const char *virtus_sensor_name(virtus_sensor_id_t id);
```

---

## 5. Actuator Interface

```c
// virtus_hal.h — Actuator interface

typedef enum {
    ACTUATOR_MOTOR_LEFT   = 0,
    ACTUATOR_MOTOR_RIGHT  = 1,
    ACTUATOR_LIFT         = 2,
    ACTUATOR_RELAY_1      = 3,
    ACTUATOR_RELAY_2      = 4,
    ACTUATOR_RELAY_3      = 5,
    ACTUATOR_RELAY_4      = 6,
    ACTUATOR_MAX          = 8
} virtus_actuator_id_t;

// Unified actuator command
typedef struct {
    virtus_actuator_id_t id;
    union {
        struct { int16_t speed_pct; }  motor;    // -100 to +100, 0 = stop
        struct { uint8_t cmd; }        lift;     // 0=STOP, 1=UP, 2=DOWN
        struct { bool state; }         relay;    // true = energized
    };
} virtus_actuator_cmd_t;

typedef struct {
    virtus_actuator_id_t id;
    union {
        struct {
            int16_t  speed_pct;
            uint16_t current_ma;
            float    temp_est_c;
            bool     enabled;
            bool     fault;
        } motor;
        struct { uint8_t position; bool moving; }  lift;
        struct { bool state; uint32_t on_time_ms; } relay;
    };
} virtus_actuator_state_t;

typedef struct {
    virtus_actuator_id_t id;
    const char          *name;
    const char          *version;

    int  (*init)(void);
    int  (*deinit)(void);
    bool (*is_ready)(void);

    int  (*set)(const virtus_actuator_cmd_t *cmd);
    int  (*get_state)(virtus_actuator_state_t *out);
    int  (*emergency_stop)(void);    // Must be callable from any context, including ISR

    bool (*is_healthy)(void);
    void (*get_diagnostics)(char *buf, size_t len);
} virtus_actuator_driver_t;

// VHAL API
int virtus_actuator_register(const virtus_actuator_driver_t *driver);
int virtus_actuator_init_all(void);
int virtus_actuator_set(virtus_actuator_id_t id, const virtus_actuator_cmd_t *cmd);
int virtus_actuator_get_state(virtus_actuator_id_t id, virtus_actuator_state_t *out);
int virtus_actuator_emergency_stop_all(void);  // Stops ALL actuators immediately
bool virtus_actuator_is_healthy(virtus_actuator_id_t id);
```

---

## 6. Communication Interface

The serial protocol between ESP32s and RPi also goes through VHAL — the transport layer is abstracted:

```c
typedef struct {
    const char *name;             // "USB_CDC_0", "USB_CDC_1"
    int  (*init)(uint32_t baud);
    int  (*send)(const uint8_t *buf, size_t len);
    int  (*recv)(uint8_t *buf, size_t len, k_timeout_t timeout);
    bool (*is_connected)(void);
} virtus_comm_driver_t;
```

This means if Danush routes the RPi communication through SPI instead of USB CDC in a future PCB revision, nothing in the application logic changes.

---

## 7. Driver Implementations

### VL53L0X Driver

```c
// hal/drivers/sensors/vl53l0x_driver.c

static int vl53l0x_hal_init(void) {
    const struct device *dev = DEVICE_DT_GET(DT_ALIAS(tof_sensor));
    if (!device_is_ready(dev)) return -ENODEV;
    // Store device reference in module-static for read calls
    s_dev = dev;
    return 0;
}

static int vl53l0x_hal_read(virtus_sensor_data_t *out) {
    struct sensor_value distance;
    int ret = sensor_sample_fetch(s_dev);
    if (ret) return ret;
    sensor_channel_get(s_dev, SENSOR_CHAN_DISTANCE, &distance);
    out->sensor_id  = SENSOR_TOF;
    out->valid      = true;
    out->tof.mm     = (uint16_t)(sensor_value_to_double(&distance) * 1000.0);
    out->timestamp_us = k_uptime_get() * 1000;
    return 0;
}

static bool vl53l0x_hal_is_healthy(void) {
    return s_last_read_success_count >= 3;  // Last 3 reads succeeded
}

// The driver struct — registered at boot
const virtus_sensor_driver_t vl53l0x_driver = {
    .id      = SENSOR_TOF,
    .name    = "VL53L0X",
    .version = "1.0.0",
    .init    = vl53l0x_hal_init,
    .read    = vl53l0x_hal_read,
    .is_healthy = vl53l0x_hal_is_healthy,
    // ...
};
```

### BTS7960 Driver

```c
// hal/drivers/actuators/bts7960_driver.c

static int bts7960_hal_set(const virtus_actuator_cmd_t *cmd) {
    int16_t speed = cmd->motor.speed_pct;
    uint32_t period = PWM_USEC(1000);

    if (speed >= 0) {
        pwm_set_dt(&motor_left_rpwm, period, PWM_USEC(speed * 10));
        pwm_set_dt(&motor_left_lpwm, period, PWM_USEC(0));
    } else {
        pwm_set_dt(&motor_left_rpwm, period, PWM_USEC(0));
        pwm_set_dt(&motor_left_lpwm, period, PWM_USEC(-speed * 10));
    }
    s_current_speed = speed;
    return 0;
}

static int bts7960_hal_emergency_stop(void) {
    // Must work from ISR context — use raw register writes, not k_sleep
    gpio_pin_set_dt(&motor_left_en,  0);
    gpio_pin_set_dt(&motor_right_en, 0);
    s_current_speed = 0;
    return 0;
}

const virtus_actuator_driver_t bts7960_driver = {
    .id              = ACTUATOR_MOTOR_LEFT,  // Separate instance for right
    .name            = "BTS7960",
    .version         = "1.0.0",
    .set             = bts7960_hal_set,
    .emergency_stop  = bts7960_hal_emergency_stop,
    // ...
};
```

---

## 8. Driver Registry

```c
// hal/src/hal_core.c

#define MAX_SENSOR_DRIVERS  SENSOR_MAX
#define MAX_ACTUATOR_DRIVERS ACTUATOR_MAX

static const virtus_sensor_driver_t   *s_sensors[MAX_SENSOR_DRIVERS]   = {0};
static const virtus_actuator_driver_t *s_actuators[MAX_ACTUATOR_DRIVERS] = {0};

int virtus_sensor_register(const virtus_sensor_driver_t *driver) {
    if (driver->id >= SENSOR_MAX) return -EINVAL;
    if (s_sensors[driver->id] != NULL) {
        LOG_WRN("Sensor %d already registered — replacing %s with %s",
                driver->id, s_sensors[driver->id]->name, driver->name);
    }
    s_sensors[driver->id] = driver;
    LOG_INF("Registered sensor: %s (id=%d, version=%s)",
            driver->name, driver->id, driver->version);
    return 0;
}

int virtus_sensor_read(virtus_sensor_id_t id, virtus_sensor_data_t *out) {
    if (id >= SENSOR_MAX || s_sensors[id] == NULL) return -ENODEV;
    return s_sensors[id]->read(out);
}
```

### Boot Registration (main.c)

```c
// ESP32 #2 main.c — register all drivers at boot
int main(void) {
    // Register sensors
    virtus_sensor_register(&vl53l0x_driver);
    virtus_sensor_register(&hcsr04_driver);
    virtus_sensor_register(&rcwl0516_driver);

    // Initialize all registered sensors
    int ret = virtus_sensor_init_all();
    if (ret) {
        LOG_ERR("Sensor init failed: %d", ret);
        // Enter ERROR state
    }

    // Start application threads — they use VHAL, not direct hardware
    k_thread_create(&sensor_fusion_thread, ...);
}
```

---

## 9. Health Monitoring

VHAL maintains per-driver health state and exposes it via the existing orchestrator health check:

```c
// hal/src/hal_health.c

typedef struct {
    uint32_t read_count;
    uint32_t fail_count;
    uint32_t consecutive_fails;
    int64_t  last_success_us;
    bool     declared_unhealthy;
} virtus_driver_health_t;

#define UNHEALTHY_CONSECUTIVE_FAILS 5
#define UNHEALTHY_STALE_MS          500  // No successful read for 500ms = unhealthy

bool virtus_sensor_is_healthy(virtus_sensor_id_t id) {
    const virtus_driver_health_t *h = &s_sensor_health[id];
    if (h->consecutive_fails >= UNHEALTHY_CONSECUTIVE_FAILS) return false;
    if ((k_uptime_get() - h->last_success_us/1000) > UNHEALTHY_STALE_MS) return false;
    return true;
}

// VHAL wraps every read call with health tracking
int virtus_sensor_read(virtus_sensor_id_t id, virtus_sensor_data_t *out) {
    int ret = s_sensors[id]->read(out);
    if (ret == 0 && out->valid) {
        s_sensor_health[id].consecutive_fails = 0;
        s_sensor_health[id].last_success_us   = k_uptime_get() * 1000;
    } else {
        s_sensor_health[id].consecutive_fails++;
        s_sensor_health[id].fail_count++;
    }
    return ret;
}
```

The health state feeds directly into the BridgeFrame `health_flags` field (defined in VDL) that the orchestrator uses for health checks.

---

## 10. Error Handling

All VHAL functions return standard Zephyr errno codes:

| Code | Meaning | Driver action |
|---|---|---|
| `0` | Success | Normal |
| `-ENODEV` | Driver not registered | Log + return |
| `-EIO` | I/O error (I2C NACK, timeout) | Increment fail counter |
| `-EBUSY` | Hardware busy | Retry with backoff |
| `-EINVAL` | Invalid parameter | Log + assert in debug builds |
| `-ETIME` | Timeout waiting for data | Mark reading as invalid |

Emergency stop is a special case — it must **never fail silently**:

```c
int virtus_actuator_emergency_stop_all(void) {
    int worst = 0;
    for (int i = 0; i < ACTUATOR_MAX; i++) {
        if (s_actuators[i] != NULL) {
            int ret = s_actuators[i]->emergency_stop();
            if (ret) {
                // Emergency stop failed — this is a FATAL condition
                // Log to crash dump, trigger hardware watchdog reset
                LOG_ERR("FATAL: Emergency stop failed for actuator %d: %d", i, ret);
                worst = ret;
            }
        }
    }
    return worst;
}
```

---

## 11. Testing Strategy

### Mock Drivers for Ztest

```c
// tests/test_hal_sensor_mock.c

// A mock ToF sensor — always returns 500mm
static int mock_tof_read(virtus_sensor_data_t *out) {
    out->sensor_id  = SENSOR_TOF;
    out->valid      = true;
    out->tof.mm     = 500;
    out->timestamp_us = k_uptime_get() * 1000;
    return 0;
}

static const virtus_sensor_driver_t mock_tof_driver = {
    .id   = SENSOR_TOF,
    .name = "MOCK_TOF",
    .init = NULL,  // No init needed for mock
    .read = mock_tof_read,
    .is_healthy = []() { return true; },
};

ZTEST(hal_sensor_suite, test_sensor_read_dispatches_to_driver) {
    virtus_sensor_register(&mock_tof_driver);
    virtus_sensor_data_t data;
    int ret = virtus_sensor_read(SENSOR_TOF, &data);
    zassert_equal(ret, 0, "Read should succeed");
    zassert_equal(data.tof.mm, 500, "Should return mock value");
    zassert_true(data.valid, "Should be valid");
}

ZTEST(hal_sensor_suite, test_unregistered_sensor_returns_enodev) {
    virtus_sensor_data_t data;
    int ret = virtus_sensor_read(SENSOR_LOADCELL, &data);
    zassert_equal(ret, -ENODEV, "Unregistered sensor should return -ENODEV");
}
```

These tests run on `native_sim` — no hardware required, runs in CI.

---

## 12. Integration with Zephyr Driver Model

VHAL sits **above** Zephyr's driver model, not beside it. Each VHAL driver implementation uses Zephyr's native APIs internally:

```
VHAL virtus_sensor_read(SENSOR_TOF, &data)
    → vl53l0x_driver.read(&data)
        → sensor_sample_fetch(zephyr_vl53l0x_device)  ← Zephyr native API
        → sensor_channel_get(...)                      ← Zephyr native API
```

This means VHAL drivers benefit from Zephyr's existing VL53L0X, HC-SR04, and PWM drivers — they don't reimplement hardware access, they just wrap it in a consistent VHAL interface.

---

## 13. Phased Build Plan

### Phase 1 — Core Interface + Registry (2 days)
- `virtus_hal.h` — full sensor + actuator interface definitions
- `hal_core.c` — driver registry + dispatch
- Ztest suite for registry (register, read dispatch, unregistered returns -ENODEV)
- **Deliverable:** VHAL compiles and passes tests on `native_sim`

### Phase 2 — Sensor Driver Implementations (3 days)
- `vl53l0x_driver.c` — migrated from current direct usage
- `hcsr04_driver.c`
- `rcwl0516_driver.c`
- `hal_health.c` — health tracking wrapper
- **Deliverable:** ESP32 #2 sensor fusion uses VHAL internally

### Phase 3 — Actuator Driver Implementations (2 days)
- `bts7960_driver.c` — migrated from current direct usage
- `relay_driver.c`
- Emergency stop path tested on hardware
- **Deliverable:** ESP32 #1 motor controller uses VHAL internally

### Phase 4 — Communication Interface (1 day)
- `virtus_comm_driver_t` interface
- USB CDC driver implementation
- **Deliverable:** Serial communication abstracted

### Phase 5 — Mock Drivers + Full Test Suite (2 days)
- Complete mock drivers for all sensor + actuator types
- Full Ztest suite: 40+ tests
- CI integration (native_sim)
- **Deliverable:** All VHAL tests pass in CI without hardware

---

## 14. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| Language | C (C99) | Zephyr RTOS native language |
| Interface style | Function pointer structs | Standard embedded HAL pattern |
| Error codes | Zephyr errno | Consistent with Zephyr ecosystem |
| Testing | Ztest on native_sim | Runs in CI, no hardware needed |
| Health tracking | Module-static counters | No dynamic allocation |
| Emergency stop | ISR-safe raw operations | Safety-critical path |

---

## 15. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Zephyr sensor API changes between versions | Low | Medium | VHAL drivers isolate this — only driver files need updating |
| Emergency stop not ISR-safe | Medium | Critical | Code review gate: `emergency_stop` implementations audited for ISR safety |
| Mock drivers diverge from real drivers over time | Medium | Medium | Integration tests run on real hardware in addition to native_sim mocks |
| Performance overhead from function pointer dispatch | Low | Low | Single indirection — negligible on ESP32 at sensor read rates (50Hz max) |
| New hardware doesn't fit existing interface | Medium | Medium | Interface extended with versioned `capabilities` bitmask — new fields added, never removed |
