
# 🧑‍💻 **Porter Robotics – Developer Log**

### **Date:** 25 Feb 2026

### **Engineer:** *Antony Austin*

### **Subsystem:** ROS2 Jazzy Migration / Lidar Integration / Repo Setup

---

# ✅ **1. YDLIDAR Investigation**

### ✔ Confirmed hardware identity

* Lidar model identified via SDK test tool: **S2PRO / YD-47** (not X4 Pro).
* Native SDK baudrate working: **128000**
* Verified using:

  ```
  ./tri_test
  ```

### ✔ SDK functionality verified

* `tri_test` from **YDLidar SDK** successfully:

  * Connected
  * Retrieved firmware
  * Produced valid scan data
  * Scan points ~1320
  * Scan frequency ~3.7 Hz

⚠ **But ROS2 Jazzy driver did NOT work**

* Driver crashed with:

  ```
  Error, cannot retrieve Yd Lidar health code: ffffffff
  ```
* Concluded: **Official ydlidar_ros2 driver is outdated and incompatible with Jazzy.**
* Decision: **We will build our own ROS2 Jazzy–compatible driver.**

---

# ✅ **2. Architecture Plan for Custom ROS2 Driver**

We evaluated 3 approaches to fix the Jazzy incompatibility:

| Option | Description                                    | Decision             |
| ------ | ---------------------------------------------- | -------------------- |
| A      | Patch the old driver                           | ❌ Too brittle        |
| B      | Rewrite full open-source driver                | ✅ Will do eventually |
| C      | Build company-specific production driver first | ⭐ **Chosen**         |

### Why Option C?

* Faster to production for Porter Robot
* Control over performance-critical C++ path
* Clean architecture
* Later contribute stable version to ROS community

---

# ✅ **3. Final Project Architecture**

You approved the following structure:

```
porter_robot/
│
├── docker/
│   ├── Dockerfile.dev
│   ├── Dockerfile.prod
│   ├── docker-compose.dev.yml
│   └── docker-compose.prod.yml
│
├── lidar/
│   ├── ydlidar_driver/             # open-source style driver rewrite
│   ├── porter_lidar_processor/     # internal high-level Lidar logic
│
├── orchestration/
│   └── porter_orchestrator/        # Python state machine, monitoring
│
├── nav/
│   └── nav2_config/                # Navigation configs
│
├── simulation/
│   └── porter_robot_urdf/          # Later
│
├── tools/                          # Debug tools
│
└── docs/
```

This is the **official folder layout** going forward.

---

# ✅ **4. Dockerization Setup**

You requested to use a *correct* ROS2 Jazzy Docker baseline.
We selected the best canonical one:

Base image:

```
osrf/ros:jazzy-desktop
```

Created working files:

* `docker-compose.dev.yml`
* `Dockerfile.dev`
* Build verified up to rosdep step.

---

# ⚠ Build Error Solved

Error:

```
rosdep install --from-paths src --ignore-src -y
given path 'src' does not exist
```

Cause:
Docker was built from `docker/` directory, *not root*, so no `src/` folder visible.

Fix:
Run Docker from **porter_robot root**:

```
cd porter_robot
docker compose -f docker/docker-compose.dev.yml up -d
```

---

# ✅ **5. GitHub Setup**

* You set up SSH authentication successfully:

```
ssh -T git@github.com
Hi austin207! You've successfully authenticated
```

* Confirmed repo:
  `github.com/austin207/Porter-ROS`

* Found existing branch: **prototype**

* Created new branch for development:

```
git checkout -b prototype
```

* Copied new project structure from workspace into repo.

### Issue: Git ignored empty folders

Solution: Added `.gitkeep` files so structure appears on GitHub.

---

# 🎯 **6. Current Repository Status**

Your `prototype` branch on GitHub now contains:

```
porter_robot/
  docker/
  docs/
  src/
    nav/
    orchestration/
    porter_lidar_processor/
    simulation/
    tools/
    ydlidar_driver/
```

Everything is clean and ready for coding.

---

# 🚀 **7. Tomorrow’s Development Plan**

### **Phase 1: YDLIDAR C++ Driver Rewrite (Jazzy Compatible)**

* Create a minimal driver using:

  * Serial interface (hardware-agnostic)
  * C++17
  * ROS2 Jazzy rclcpp
* Implement pipeline:

  * Connect → Health → Start Scan → Parse packets → Publish LaserScan
* Add diagnostics & logging

### **Phase 2: Lidar Processing Layer**

* Noise filtering
* Median smoothing
* Scan limiter
* Health monitoring

### **Phase 3: Python Orchestration Layer**

* Battery + Lidar + State Machine
* Integration with Nav2

### **Phase 4: Docker Workspace Stabilization**

* Multi-stage builds
* GPU passthrough (if needed)
* CI integration