# Software Setup

This guide covers building the ROS 2 workspace, flashing ESP32 firmware, and running the robot.

## Docker Development Environment

The primary development workflow uses Docker. All commands run from `porter_robot/`.

```bash
# Build the dev image
docker compose -f docker/docker-compose.dev.yml build

# Start the container (detached)
docker compose -f docker/docker-compose.dev.yml up -d

# Enter the container
docker exec -it porter_dev bash
```

!!! note "Network mode"
    All Docker services use `network_mode: host` for DDS discovery between ROS 2 nodes.

## Building the ROS 2 Workspace

Inside the dev container:

```bash
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --cmake-args -Wno-dev
source install/setup.bash
```

To build a single package:

```bash
colcon build --packages-select ydlidar_driver --symlink-install
```

To perform a clean rebuild:

```bash
rm -rf build/ install/ log/
colcon build --cmake-clean-cache --symlink-install
```

## Running Tests

```bash
# All packages
colcon test --event-handlers console_direct+
colcon test-result --verbose

# Single package
colcon test --packages-select porter_ai_assistant
```

## Flashing ESP32 Firmware

ESP32 firmware uses Zephyr RTOS v4.0.0. Build with West from a Zephyr-configured environment (not the ROS 2 container).

```bash
# Motor controller
west build -b esp32_devkitc_wroom esp32_firmware/motor_controller \
    -d build/motor -- -DBOARD_ROOT=.

# Sensor fusion
west build -b esp32_devkitc_wroom esp32_firmware/sensor_fusion \
    -d build/sensor -- -DBOARD_ROOT=.

# Flash (connect ESP32 via USB)
west flash -d build/motor
west flash -d build/sensor
```

Running firmware unit tests (no hardware needed):

```bash
cd esp32_firmware && twister -T tests/ -p native_sim
```

!!! danger "Do not mix environments"
    Never activate the Zephyr venv and ROS 2 environment in the same shell session. Their toolchains conflict.

## Installing udev Rules

For stable ESP32 device names on the Raspberry Pi:

```bash
sudo bash esp32_firmware/udev/install_rules.sh
sudo udevadm control --reload-rules
sudo udevadm trigger
```

This creates `/dev/esp32_motors` and `/dev/esp32_sensors` symlinks.

## Running the Robot

```bash
# Full bringup (LIDAR + processing + state machine + bridges)
ros2 launch porter_orchestrator porter_bringup.launch.py

# AI assistant (standalone HTTP server for Flutter GUI)
ros2 run porter_ai_assistant ai_server
```

## Flutter GUI

```bash
cd src/porter_gui
flutter analyze
flutter test
flutter build linux --release
```

The release bundle is approximately 50 MB and runs natively on ARM64 (RPi) or x64 (dev).
