# Virtus Simulation Manager

Manage Gazebo simulations, Nav2 parameter tuning, bag file recording/playback, and test scenarios from within VS Code.

## Features

### One-Click Launch Profiles

Pre-configured simulation profiles for common development tasks:

| Profile | Description |
|---------|-------------|
| **Empty World** | Basic robot in empty Gazebo world |
| **Airport Terminal** | Corridor + gate environment with obstacles |
| **Stress Test** | Dense crowd simulation for Nav2 tuning |
| **Mapping** | SLAM-enabled drive-around for map building |
| **Replay** | Playback a recorded bag file in simulation |

### URDF Preview

Live 3D preview of the Porter URDF model. Edit joint positions, link dimensions, and sensor placements with immediate visual feedback.

### Nav2 Parameter Editor

Visual editor for 27 Nav2 parameters with descriptions, valid ranges, and real-time preview of their effect on path planning.

| Category | Parameters |
|----------|-----------|
| **Controller** | max_vel_x, max_vel_theta, min_vel_x, sim_time, vx_samples |
| **Planner** | tolerance, use_astar, allow_unknown, cost_factor |
| **Costmap** | resolution, width, height, inflation_radius, cost_scaling_factor |
| **Recovery** | spin_dist, backup_dist, wait_time |
| **General** | transform_tolerance, controller_frequency, planner_frequency |

### Bag File Manager

- **Record** — Start/stop bag recording with topic filters
- **Browse** — List all recorded bags with duration, size, and topic counts
- **Playback** — Replay bags with rate control and topic remapping
- **Export** — Convert bags to CSV for offline analysis

### Test Scenario Runner

Define test scenarios as YAML files and run them in simulation:

```yaml
name: "corridor_navigation"
world: "airport_terminal"
robot_start: [0.0, 0.0, 0.0]
goals:
  - [10.0, 0.0, 0.0]
  - [10.0, 5.0, 1.57]
timeout: 120
success_criteria:
  - goal_reached: true
  - collision_count: 0
  - max_duration: 60
```

### World Manager

Create, edit, and organize Gazebo world files. Import SDF models from the Gazebo model repository or custom assets.

!!! info "Gazebo version"
    The simulation manager targets Gazebo Harmonic (the default for ROS 2 Jazzy).
