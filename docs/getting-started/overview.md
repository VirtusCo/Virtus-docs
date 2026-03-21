# What is Porter?

Porter is an autonomous luggage-carrying robot built by VirtusCo for airport terminals. It navigates using LIDAR and sensor fusion, carries up to 120 kg of passenger luggage, and provides an on-device AI assistant ("Virtue") for wayfinding, flight info, and check-in assistance — all running locally on a Raspberry Pi with zero cloud dependency.

## Key Capabilities

| Feature | Description |
|---------|-------------|
| **120 kg payload** | Robust chassis for multiple heavy bags |
| **Autonomous navigation** | 360 LIDAR + triple-sensor Kalman fusion + Nav2 |
| **On-device AI** | Qwen 2.5 1.5B running locally — no internet required |
| **Interactive display** | Flutter touchscreen with chat, maps, flight status |
| **Dual-mode operation** | Fixed-path autonomous + semi-autonomous for busy areas |
| **Airport integration** | Boarding pass scanning for gate-aware navigation |

## Prerequisites

| Requirement | Version |
|-------------|---------|
| Ubuntu | 24.04 LTS (Noble) |
| ROS 2 | Jazzy Jalisco |
| Docker | 24.0+ with Compose V2 |
| Python | 3.12+ |
| Raspberry Pi | 5 (4 GB+ RAM) |
| YDLIDAR | X4 Pro 360 (or compatible model) |
| ESP32 | 2x DevKitC WROOM |

## Quick Start

```bash
# Clone the repository
git clone https://github.com/austin207/Porter-ROS.git
cd Porter-ROS/porter_robot

# Build and launch the dev container
docker compose -f docker/docker-compose.dev.yml build
docker compose -f docker/docker-compose.dev.yml up -d
docker exec -it porter_dev bash

# Inside the container
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install --cmake-args -Wno-dev
source install/setup.bash

# Launch the robot
ros2 launch porter_orchestrator porter_bringup.launch.py
```

!!! tip "First-time setup"
    See the [Software Setup](software.md) page for Docker installation, ESP32 flashing, and detailed build instructions.

## Project Status

The prototype is in active development on the `prototype` branch. The core software stack — LIDAR driver, scan processing, state machine, ESP32 firmware, AI assistant, and Flutter GUI — is complete with 398+ passing tests. Nav2 integration is the current milestone.
