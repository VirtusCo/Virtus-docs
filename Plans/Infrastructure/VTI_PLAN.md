# Virtus Test Infrastructure (VTI)
### Detailed Development Plan
**Project:** Three-layer automated test infrastructure — unit, integration, and system tests running in CI without physical hardware  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026  
**Build Priority:** #2 — Build before first deployment

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [Three-Layer Test Architecture](#2-three-layer-test-architecture)
3. [Layer 1 — Unit Tests](#3-layer-1--unit-tests)
4. [Layer 2 — Integration Tests](#4-layer-2--integration-tests)
5. [Layer 3 — System Tests](#5-layer-3--system-tests)
6. [Mock ESP32 Bridge](#6-mock-esp32-bridge)
7. [Bag Replay Test Runner](#7-bag-replay-test-runner)
8. [Scenario Library](#8-scenario-library)
9. [CI/CD Integration](#9-cicd-integration)
10. [Hardware-in-the-Loop Tests](#10-hardware-in-the-loop-tests)
11. [Test Coverage Targets](#11-test-coverage-targets)
12. [Phased Build Plan](#12-phased-build-plan)
13. [Tech Stack Summary](#13-tech-stack-summary)
14. [Risk Register](#14-risk-register)

---

## 1. Overview & Vision

### The Current State

You have 178 Ztest cases for ESP32 firmware running on `native_sim`. That's excellent for the embedded layer. What's missing:

- No automated test for "does the orchestrator FSM transition correctly when a STUCK event fires?"
- No automated test for "does the lidar processor produce valid output when given a real LiDAR scan?"
- No automated test for "does the full stack from sensor input to motor command produce safe outputs?"
- No automated way to verify a new firmware version doesn't break the RPi ↔ ESP32 protocol

### What VTI Provides

```
Layer 1 — Unit Tests        → Already exist for firmware. Extend to ROS 2 nodes.
Layer 2 — Integration Tests → NEW: nodes talking to each other with mock hardware
Layer 3 — System Tests      → NEW: full Gazebo headless simulation scenarios
```

All three layers run in GitHub Actions on every push. No physical hardware required.

### Why This Is Critical Before First Deployment

When Virtus is running at Kochi airport and a software update breaks motor response, you need to know **before you deploy**, not after a robot stops in front of a passenger. VTI is the safety net between development and production.

---

## 2. Three-Layer Test Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  Layer 3 — System Tests                                            │
│  Gazebo headless + full ROS 2 stack + scenario assertions         │
│  Runtime: ~10 minutes  Runs: on PR to main                        │
├────────────────────────────────────────────────────────────────────┤
│  Layer 2 — Integration Tests                                       │
│  ROS 2 launch_testing + mock ESP32 bridge + bag replay            │
│  Runtime: ~3 minutes   Runs: on every push                        │
├────────────────────────────────────────────────────────────────────┤
│  Layer 1 — Unit Tests                                              │
│  Ztest (firmware) + pytest (ROS 2 nodes)                          │
│  Runtime: ~90 seconds  Runs: on every push                        │
└────────────────────────────────────────────────────────────────────┘
```

**Test pyramid principle:** Many cheap fast unit tests, fewer slower integration tests, fewest slowest system tests. CI fails fast on unit test failures before running slow tests.

---

## 3. Layer 1 — Unit Tests

### Firmware (Ztest) — Extending Existing 178 Tests

The existing Ztest suite covers motor controller and sensor fusion. Gaps to fill:

```
esp32_firmware/tests/
├── test_motor_controller/     ← EXISTS: 94 tests
├── test_sensor_fusion/        ← EXISTS: 84 tests
├── test_hal_registry/         ← ADD: VHAL driver registration tests
├── test_protocol_codec/       ← ADD: CRC16-CCITT encode/decode roundtrip tests
└── test_kalman_filter/        ← ADD: Kalman filter accuracy tests with known inputs
```

**New test: Protocol codec roundtrip**

```c
// test_protocol_codec.c
ZTEST(protocol_suite, test_motor_cmd_encode_decode_roundtrip) {
    virtus_bridge_frame_t original = {
        .msg_type       = TYPE_MOTOR_CMD,
        .motor_left_speed  = 75,
        .motor_right_speed = -30,
        .lift_cmd       = 0,
    };

    uint8_t buf[64];
    size_t len = virtus_protocol_encode(&original, buf, sizeof(buf));
    zassert_true(len > 0, "Encode should succeed");

    virtus_bridge_frame_t decoded;
    int ret = virtus_protocol_decode(buf, len, &decoded);
    zassert_equal(ret, 0, "Decode should succeed");
    zassert_equal(decoded.motor_left_speed,  75, "Left speed roundtrip");
    zassert_equal(decoded.motor_right_speed, -30, "Right speed roundtrip");
}

ZTEST(protocol_suite, test_bad_crc_returns_error) {
    uint8_t buf[] = {0xAA, 0x01, 0x04, 0x00, 0xFF, 0x00, 0x00, 0x00, 0xFF, 0xFF};
    buf[8] = 0x00;  // Corrupt CRC
    virtus_bridge_frame_t decoded;
    int ret = virtus_protocol_decode(buf, sizeof(buf), &decoded);
    zassert_equal(ret, -EBADMSG, "Bad CRC should return -EBADMSG");
}
```

### ROS 2 Node Unit Tests (pytest + rclpy)

```
src/
├── porter_lidar_processor/
│   └── tests/
│       ├── test_filters.py             ← EXISTS (likely) — filter unit tests
│       └── test_processor_node.py     ← ADD: node instantiation + single message processing
├── porter_orchestrator/
│   └── tests/
│       ├── test_fsm_transitions.py    ← ADD: FSM state machine transition tests
│       └── test_recovery_logic.py    ← ADD: 3-attempt recovery counter tests
└── porter_ai_assistant/
    └── tests/
        ├── test_intent_classifier.py  ← ADD: keyword classifier unit tests
        └── test_tool_executor.py     ← ADD: mock tool call tests
```

**New test: FSM transition unit tests**

```python
# porter_orchestrator/tests/test_fsm_transitions.py
import pytest
from porter_orchestrator.fsm import OrchestratorFSM

class TestFSMTransitions:

    def test_boot_to_health_check_on_system_ready(self):
        fsm = OrchestratorFSM()
        assert fsm.state == 'BOOT'
        fsm.trigger('system_ready')
        assert fsm.state == 'HEALTH_CHECK'

    def test_health_check_to_error_on_health_fail(self):
        fsm = OrchestratorFSM()
        fsm.trigger('system_ready')
        fsm.trigger('health_fail')
        assert fsm.state == 'ERROR'

    def test_recovery_increments_attempt_counter(self):
        fsm = OrchestratorFSM()
        fsm.force_state('ERROR')
        fsm.trigger('recovery_triggered')
        assert fsm.state == 'RECOVERY'
        assert fsm.recovery_attempt == 1

    def test_max_recovery_attempts_stays_in_error(self):
        fsm = OrchestratorFSM(max_recovery_attempts=3)
        fsm.force_state('ERROR')
        for _ in range(3):
            fsm.trigger('recovery_triggered')
            fsm.trigger('reset')
        fsm.trigger('recovery_triggered')
        # 4th recovery attempt exceeds max — stays in ERROR
        assert fsm.state == 'ERROR'
        assert fsm.recovery_attempt == 3

    def test_invalid_transition_raises_no_exception(self):
        # FSM should silently ignore invalid triggers
        fsm = OrchestratorFSM()
        fsm.trigger('obstacle_detected')  # Not valid in BOOT state
        assert fsm.state == 'BOOT'        # Unchanged
```

---

## 4. Layer 2 — Integration Tests

### ROS 2 launch_testing

Integration tests launch real ROS 2 nodes in a subprocess, feed them test data, and assert on their output topics. The key tool is `launch_testing` — the standard ROS 2 integration test framework.

```python
# porter_orchestrator/tests/test_orchestrator_integration.py
import launch
import launch_ros.actions
import launch_testing
import launch_testing.actions
import rclpy
from virtus_msgs.msg import OrchestratorState, SensorFusion
import pytest, unittest

@pytest.mark.launch_test
def generate_test_description():
    # Launch the orchestrator node with test config
    orchestrator = launch_ros.actions.Node(
        package='porter_orchestrator',
        executable='orchestrator_node',
        parameters=[{'health_check_timeout': 2.0, 'recovery_attempts': 2}]
    )
    return launch.LaunchDescription([
        orchestrator,
        launch_testing.actions.ReadyToTest()
    ])

class TestOrchestratorIntegration(unittest.TestCase):

    @classmethod
    def setUpClass(cls):
        rclpy.init()
        cls.node = rclpy.create_node('test_node')
        cls.state_pub  = cls.node.create_publisher(OrchestratorState, '/orchestrator/state', 10)
        cls.sensor_pub = cls.node.create_publisher(SensorFusion, '/sensor_fusion', 10)
        cls.received_states = []
        cls.node.create_subscription(
            OrchestratorState, '/orchestrator/state',
            lambda msg: cls.received_states.append(msg.state), 10
        )

    def test_transitions_to_idle_after_health_check(self):
        # Publish healthy sensor data
        msg = SensorFusion()
        msg.health_flags = 0b111  # All sensors healthy
        for _ in range(10):
            self.sensor_pub.publish(msg)
            rclpy.spin_once(self.node, timeout_sec=0.1)

        # Should eventually reach IDLE
        import time; time.sleep(3.0)  # Wait for health_check_timeout
        rclpy.spin_once(self.node, timeout_sec=0.1)
        assert OrchestratorState.STATE_IDLE in self.received_states, \
            f"Expected IDLE in states, got: {self.received_states}"

    def test_obstacle_triggers_avoidance(self):
        # Force to NAVIGATING state
        # ... (setup)
        # Publish obstacle at 30cm (NEAR threshold)
        obstacle = SensorFusion()
        obstacle.kalman_estimate_cm = 30.0
        obstacle.obstacle_class = SensorFusion.OBSTACLE_NEAR
        self.sensor_pub.publish(obstacle)
        rclpy.spin_once(self.node, timeout_sec=0.5)
        assert self.received_states[-1] == OrchestratorState.STATE_OBSTACLE_AVOIDANCE
```

---

## 5. Layer 3 — System Tests

### Gazebo Headless System Tests

Full stack system tests launch Gazebo in headless mode (no GUI), spawn the Virtus robot, run a full scenario, and assert the robot's behaviour.

```python
# tests/system/test_navigation_scenarios.py

import subprocess, time, rclpy
from virtus_msgs.msg import OrchestratorState
from nav_msgs.msg import Odometry
from geometry_msgs.msg import PoseStamped

class TestNavigationScenarios:

    def setup_method(self):
        """Launch full simulation stack before each test"""
        self.sim_proc = subprocess.Popen([
            'ros2', 'launch', 'porter_bringup',
            'simulation_test.launch.py',
            'headless:=true', 'world:=test_corridor.world'
        ])
        time.sleep(8.0)  # Wait for Gazebo + Nav2 to initialize
        rclpy.init()
        self.node = rclpy.create_node('system_test_node')

    def teardown_method(self):
        """Kill simulation after each test"""
        self.sim_proc.terminate()
        self.sim_proc.wait(timeout=10)
        rclpy.shutdown()

    def test_robot_reaches_goal_in_clear_corridor(self):
        """Robot navigates 5m straight corridor with no obstacles"""
        goal_pub = self.node.create_publisher(PoseStamped, '/goal_pose', 10)
        goal = PoseStamped()
        goal.header.frame_id = 'map'
        goal.pose.position.x = 5.0
        goal.pose.position.y = 0.0
        goal_pub.publish(goal)

        # Wait for completion (max 30s)
        start = time.time()
        reached = False
        while time.time() - start < 30.0:
            rclpy.spin_once(self.node, timeout_sec=0.5)
            # Check if robot is within 0.3m of goal
            # ...
            if reached: break

        assert reached, "Robot should reach 5m goal within 30 seconds"

    def test_robot_recovers_from_dynamic_obstacle(self):
        """Robot navigating 5m → obstacle injected at 2m → robot replans"""
        # Start navigation
        # ... inject obstacle at t=5s
        # ... assert robot replans and still reaches goal
        pass
```

---

## 6. Mock ESP32 Bridge

The most important missing piece. A Python process that **emulates both ESP32s** — publishing realistic sensor data and consuming motor commands — so the full ROS 2 stack can be tested in CI without hardware.

```python
# tests/mocks/mock_esp32_bridge.py

import rclpy
from rclpy.node import Node
from virtus_msgs.msg import SensorFusion, OrchestratorState, MotorTelemetry
from geometry_msgs.msg import Twist
import math, time

class MockESP32Bridge(Node):
    """
    Emulates ESP32 #1 (motor controller) and ESP32 #2 (sensor fusion).
    Configurable scenarios: clear path, approaching obstacle, stall detection.
    """

    SCENARIO_CLEAR       = 'clear'         # All sensors nominal, no obstacles
    SCENARIO_OBSTACLE_50 = 'obstacle_50'   # Obstacle at 50cm and closing
    SCENARIO_STALL       = 'stall'         # Motor stall simulation
    SCENARIO_SENSOR_FAIL = 'sensor_fail'   # ToF sensor fails mid-run

    def __init__(self, scenario: str = SCENARIO_CLEAR):
        super().__init__('mock_esp32_bridge')
        self.scenario    = scenario
        self.start_time  = time.time()
        self.motor_cmd   = Twist()

        # Publishers (emulating ESP32 outputs)
        self.sensor_pub  = self.create_publisher(SensorFusion,   '/sensor_fusion',    10)
        self.motor_pub   = self.create_publisher(MotorTelemetry, '/hardware/motors',  10)

        # Subscribers (receiving commands from RPi)
        self.cmd_sub     = self.create_subscription(Twist, '/cmd_vel', self._on_cmd, 10)

        # Publish at 50Hz like real ESP32s
        self.create_timer(0.02, self._publish_sensor_data)
        self.create_timer(0.1,  self._publish_motor_telemetry)

    def _on_cmd(self, msg: Twist):
        self.motor_cmd = msg  # Track what the orchestrator is commanding

    def _publish_sensor_data(self):
        msg = SensorFusion()
        t   = time.time() - self.start_time

        if self.scenario == self.SCENARIO_CLEAR:
            msg.tof_mm              = 2000
            msg.ultrasonic_cm       = 200
            msg.microwave_detected  = False
            msg.kalman_estimate_cm  = 200.0
            msg.obstacle_class      = SensorFusion.OBSTACLE_CLEAR
            msg.health_flags        = 0b111  # All healthy

        elif self.scenario == self.SCENARIO_OBSTACLE_50:
            # Obstacle closes from 200cm to 30cm over 10 seconds
            dist_cm = max(30.0, 200.0 - t * 17.0)
            msg.tof_mm              = int(dist_cm * 10)
            msg.ultrasonic_cm       = int(dist_cm)
            msg.kalman_estimate_cm  = dist_cm
            msg.obstacle_class      = (SensorFusion.OBSTACLE_NEAR
                                       if dist_cm < 50 else SensorFusion.OBSTACLE_MEDIUM)
            msg.health_flags        = 0b111

        elif self.scenario == self.SCENARIO_SENSOR_FAIL:
            msg.health_flags        = 0b110  # ToF failed (bit 0 clear)
            msg.tof_mm              = 0
            msg.ultrasonic_cm       = 180
            msg.kalman_estimate_cm  = 180.0  # Kalman runs on ultrasonic only
            msg.obstacle_class      = SensorFusion.OBSTACLE_CLEAR

        self.sensor_pub.publish(msg)

    def _publish_motor_telemetry(self):
        msg = MotorTelemetry()
        # Reflect commanded speed back as actual current
        speed = abs(self.motor_cmd.linear.x) * 100
        msg.left_current_ma  = int(speed * 12.0)
        msg.right_current_ma = int(speed * 12.0)
        msg.left_en          = True
        msg.right_en         = True

        if self.scenario == self.SCENARIO_STALL:
            # Both motors drawing max current → stall
            msg.left_current_ma  = 6500  # Above 6A threshold
            msg.right_current_ma = 6500

        self.motor_pub.publish(msg)


# Usage in integration tests:
# mock = MockESP32Bridge(scenario=MockESP32Bridge.SCENARIO_OBSTACLE_50)
# rclpy.spin(mock)
```

---

## 7. Bag Replay Test Runner

Uses recorded real `.mcap` bag files as test inputs. This is the gold standard — testing against real data from the actual robot.

```python
# tests/integration/test_bag_replay.py

class BagReplayTestRunner:
    """
    Replays a recorded ROS 2 bag file and asserts on the outputs.
    Enables testing with real sensor data from the physical robot.
    """

    def run_bag_test(self, bag_path: str, assertions: list[BagAssertion],
                     timeout_s: float = 60.0) -> BagTestResult:
        """
        1. Launches the node under test
        2. Replays the bag file
        3. Records the node's output topics
        4. Evaluates assertions against recorded output
        """
        pass

# Example test using a real recorded bag
def test_lidar_processor_with_real_scan_data():
    runner = BagReplayTestRunner()
    result = runner.run_bag_test(
        bag_path  = 'tests/bags/airport_corridor_2026_03_15.mcap',
        assertions = [
            TopicPublishedAssertion('/scan/processed', min_hz=9.0, max_hz=11.0),
            MessageValidAssertion('/scan/processed', lambda msg: len(msg.ranges) > 0),
            NoErrorsAssertion('/diagnostics', duration_s=30.0),
        ]
    )
    assert result.passed, result.failure_summary
```

### Committed Test Bags

A `tests/bags/` directory in the repo stores small (< 5MB) trimmed bag files covering key test scenarios:

```
tests/bags/
├── nominal_idle.mcap          # Robot sitting still, all sensors nominal (30s)
├── corridor_navigation.mcap   # Moving down a straight corridor (60s)
├── obstacle_avoidance.mcap    # Dynamic obstacle encounter + recovery (45s)
├── sensor_fault_tof.mcap      # ToF sensor failing mid-run (20s)
└── passenger_interaction.mcap # AI assistant conversation (30s)
```

These are the **ground truth test fixtures** — real data, real robot behaviour.

---

## 8. Scenario Library

Reusable test scenarios for both integration and system tests:

```python
# tests/scenarios/__init__.py

class VirtusTestScenario:
    """Base class for all test scenarios"""
    name:        str
    description: str
    mock_config: dict           # What the mock ESP32 bridge should emit
    success_criteria: list      # What must be true at the end
    timeout_s: float

SCENARIOS = {
    'nominal_idle': VirtusTestScenario(
        name        = 'Nominal Idle',
        description = 'Robot in IDLE state with all sensors nominal',
        mock_config = {'scenario': 'clear'},
        success_criteria = [
            FSMStateAssertion(expected='IDLE', within_s=5.0),
            TopicHzAssertion('/sensor_fusion', min_hz=45.0),
            NoAlertsAssertion(duration_s=10.0),
        ],
        timeout_s = 15.0,
    ),
    'obstacle_recovery': VirtusTestScenario(
        name        = 'Obstacle Recovery',
        description = 'Obstacle approaches → robot stops → clears → resumes',
        mock_config = {'scenario': 'obstacle_then_clear'},
        success_criteria = [
            FSMTransitionAssertion('NAVIGATING', 'OBSTACLE_AVOIDANCE', within_s=5.0),
            FSMTransitionAssertion('OBSTACLE_AVOIDANCE', 'NAVIGATING', within_s=15.0),
            ZeroVelocityAssertion(during_state='OBSTACLE_AVOIDANCE'),
        ],
        timeout_s = 30.0,
    ),
    'sensor_degradation': VirtusTestScenario(
        name        = 'ToF Sensor Failure',
        description = 'ToF sensor fails — robot should continue on ultrasonic alone',
        mock_config = {'scenario': 'sensor_fail'},
        success_criteria = [
            DiagnosticsWarnAssertion('tof_sensor', within_s=3.0),
            FSMStateAssertion(expected='NAVIGATING', within_s=5.0),  # Should NOT error
            TopicValidAssertion('/sensor_fusion', lambda m: m.kalman_estimate_cm > 0),
        ],
        timeout_s = 20.0,
    ),
}
```

---

## 9. CI/CD Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Virtus Test Suite

on: [push, pull_request]

jobs:

  unit-tests-firmware:
    name: Unit Tests — Firmware (Ztest)
    runs-on: ubuntu-latest
    container: zephyrprojectrtos/ci:latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and run Ztest suite
        run: |
          cd esp32_firmware
          west build -p -b native_sim tests/
          ./build/zephyr/zephyr.exe --no-color 2>&1 | tee ztest_output.txt
          grep -q "PASSED" ztest_output.txt

  unit-tests-ros2:
    name: Unit Tests — ROS 2 (pytest)
    runs-on: ubuntu-latest
    container: ros:jazzy
    steps:
      - uses: actions/checkout@v4
      - name: Build workspace
        run: |
          source /opt/ros/jazzy/setup.bash
          colcon build --packages-up-to porter_orchestrator porter_lidar_processor
      - name: Run pytest unit tests
        run: |
          source /opt/ros/jazzy/setup.bash
          source install/setup.bash
          pytest src/porter_orchestrator/tests/test_fsm_transitions.py -v
          pytest src/porter_lidar_processor/tests/test_filters.py -v

  integration-tests:
    name: Integration Tests (launch_testing + mock bridge)
    runs-on: ubuntu-latest
    container: ros:jazzy
    needs: [unit-tests-firmware, unit-tests-ros2]
    steps:
      - uses: actions/checkout@v4
      - name: Build full workspace
        run: |
          source /opt/ros/jazzy/setup.bash
          colcon build
      - name: Run integration tests
        run: |
          source /opt/ros/jazzy/setup.bash
          source install/setup.bash
          colcon test --packages-select porter_orchestrator_tests
          colcon test-result --verbose

  system-tests:
    name: System Tests (Gazebo headless)
    runs-on: ubuntu-latest
    container: ros:jazzy-desktop  # Needs Gazebo
    needs: [integration-tests]
    if: github.event_name == 'pull_request' && github.base_ref == 'main'
    steps:
      - uses: actions/checkout@v4
      - name: Install Gazebo
        run: sudo apt-get install -y ros-jazzy-gazebo-ros-pkgs
      - name: Run system scenario tests
        run: |
          source /opt/ros/jazzy/setup.bash
          source install/setup.bash
          pytest tests/system/ -v --timeout=120
```

### Test Result Reporting

```yaml
      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Test Results
          path: 'test-results/**/*.xml'
          reporter: java-junit
```

Test results appear directly in GitHub PR checks — green checkmarks per test suite.

---

## 10. Hardware-in-the-Loop Tests

These run on a physical robot, not in CI — triggered manually before a release:

```python
# tests/hil/test_motor_response.py
# Requires physical robot connected via USB

class TestMotorResponseHIL:
    """
    Hardware-in-the-loop tests — require physical robot.
    Run before every release with: pytest tests/hil/ --hil
    """

    def test_motor_ramp_time(self):
        """Motor should ramp from 0 to 80% duty in < 200ms"""
        # Publish cmd_vel
        # Record /hardware/motors topic
        # Assert ramp time
        pass

    def test_emergency_stop_latency(self):
        """ESTOP command → motors at 0 in < 50ms"""
        pass

    def test_sensor_fusion_accuracy(self):
        """Place obstacle at known 50cm — kalman estimate should be 50±5cm"""
        pass
```

---

## 11. Test Coverage Targets

| Component | Current | Target | How |
|---|---|---|---|
| Zephyr firmware | ~65% (178 tests) | 80% | Add protocol + VHAL tests |
| ROS 2 orchestrator FSM | 0% | 90% | FSM unit tests (pytest) |
| ROS 2 lidar processor filters | Unknown | 85% | Filter unit tests (pytest) |
| ESP32 bridge decoder | 0% | 80% | Protocol roundtrip tests |
| Full stack integration | 0% | 5 scenarios | Integration tests with mock bridge |
| System scenarios | 0% | 3 scenarios | Gazebo headless |

Coverage measured with `gcov` (firmware) and `pytest-cov` (Python).

---

## 12. Phased Build Plan

### Phase 1 — Extend Firmware Unit Tests (1 week)
- Protocol codec roundtrip tests (CRC16-CCITT)
- Kalman filter accuracy tests with known inputs
- VHAL registry tests (once VHAL is built)
- Target: 220+ firmware tests, all passing on native_sim
- **Deliverable:** Stronger firmware test suite in CI

### Phase 2 — ROS 2 Unit Tests (1 week)
- FSM transition tests (all 14 transitions covered)
- Filter unit tests (all 6 filters with edge cases)
- Intent classifier unit tests
- Tool executor mock tests
- **Deliverable:** ROS 2 node unit tests in CI

### Phase 3 — Mock ESP32 Bridge (1 week)
- `MockESP32Bridge` with 4 scenarios (clear, obstacle, stall, sensor_fail)
- Integration with `launch_testing`
- First 3 integration test scenarios using mock bridge
- **Deliverable:** ROS 2 integration tests run without hardware

### Phase 4 — Bag Replay Test Runner (3 days)
- `BagReplayTestRunner` class
- 5 committed test bags (trimmed real data)
- Bag replay tests for lidar processor + orchestrator
- **Deliverable:** Real-data regression tests

### Phase 5 — Gazebo System Tests (1 week)
- Headless Gazebo launch in CI
- 3 system scenarios: nominal, obstacle recovery, sensor degradation
- **Deliverable:** Full stack automated system tests

### Phase 6 — HIL Test Suite (ongoing)
- HIL tests for motor response, ESTOP latency, sensor accuracy
- Run manually before each release
- **Deliverable:** Pre-release hardware validation checklist automated

---

## 13. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| Firmware testing | Ztest on native_sim | Existing infrastructure, no hardware needed |
| ROS 2 unit testing | pytest + rclpy | Standard Python testing, clean assertions |
| ROS 2 integration testing | launch_testing | Official ROS 2 integration test framework |
| Mock hardware | Custom Python rclpy node | Full control over emulated behaviour |
| Bag replay | `ros2 bag play` subprocess | Standard ROS 2 tooling |
| System testing | Gazebo headless + pytest | Full simulation without display |
| CI runner | GitHub Actions | Already in use for build |
| Coverage | gcov (C) + pytest-cov (Python) | Standard tools for each language |

---

## 14. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Mock bridge diverges from real ESP32 behaviour | Medium | High | HIL tests validate mock assumptions against real hardware quarterly |
| Gazebo headless CI takes too long (>15 min) | High | Medium | System tests only run on PR to main, not every push |
| Test bags become stale (robot hardware changes) | Medium | Medium | Re-record bags after each major hardware revision; version bags with robot config |
| launch_testing flakiness (race conditions) | High | Medium | Add `WaitForTopics` barriers before assertions; set generous timeouts |
| CI container missing ROS 2 packages | Medium | Medium | Pin specific ROS 2 container versions; test container separately |
| FSM test state forcing bypasses real init | Low | Medium | Clearly document `force_state()` as test-only; guard with `#ifdef TEST_BUILD` |
