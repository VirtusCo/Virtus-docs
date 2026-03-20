# Virtus Configuration Management System (VCMS)
### Detailed Development Plan
**Project:** Single versioned configuration source of truth for all Virtus deployments — firmware, ROS 2, Nav2, AI models, and Flutter app  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026  
**Build Priority:** #3 — Build before first pilot deployment

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [The Configuration Sprawl Problem](#2-the-configuration-sprawl-problem)
3. [VCMS Architecture](#3-vcms-architecture)
4. [Deployment Profile Schema](#4-deployment-profile-schema)
5. [Config Layers](#5-config-layers)
6. [Deployment Script](#6-deployment-script)
7. [Config Validation](#7-config-validation)
8. [Version Control Strategy](#8-version-control-strategy)
9. [Runtime Config Access](#9-runtime-config-access)
10. [Config Change Audit Log](#10-config-change-audit-log)
11. [Integration with DevTools Suite](#11-integration-with-devtools-suite)
12. [Phased Build Plan](#12-phased-build-plan)
13. [Tech Stack Summary](#13-tech-stack-summary)
14. [Risk Register](#14-risk-register)

---

## 1. Overview & Vision

### What VCMS Is

A versioned configuration repository and toolchain that gives every deployed Virtus robot a single, auditable, reproducible configuration profile. One YAML file per deployment. One command to apply it.

```bash
virtus-deploy --profile configs/deployments/cial-unit-001.yaml --target 192.168.1.100
# → Pushes correct firmware config, ROS 2 params, Nav2 params, AI model selection
# → Tags the deployment in Git
# → Logs the change with timestamp + operator name
```

### Why Config Management Is a Foundation

At one robot: configuration is annoying. At ten robots across three airports: configuration without VCMS is a crisis. You will have:
- Unit 3 at CIAL running a different Nav2 costmap than Unit 1
- Unit 2 at Kempegowda with the wrong AI language model for Malayalam
- No way to answer "what exact config was running when the robot stopped at Gate B12 at 14:32 last Tuesday?"

VCMS answers all of these. It's also the foundation for the OTA update pipeline in Fleet Monitor.

---

## 2. The Configuration Sprawl Problem

### Current Config Locations (Pre-VCMS)

```
Firmware config:
  esp32_firmware/motor_controller/prj.conf      ← Zephyr Kconfig flags
  esp32_firmware/motor_controller/boards/esp32.overlay  ← DTS pin assignments
  esp32_firmware/sensor_fusion/prj.conf

ROS 2 config:
  config/lidar_params.yaml                      ← LiDAR range, filter settings
  config/orchestrator_params.yaml               ← FSM timeouts, thresholds
  config/bridge_params.yaml                     ← Serial port, baud rate

Nav2 config:
  config/nav2_params.yaml                       ← 300+ line Nav2 parameter file
  config/maps/                                  ← Airport floor plan maps

AI config:
  src/porter_ai_assistant/config/models.yaml    ← Which model to load
  src/porter_ai_assistant/config/languages.yaml ← Active languages

Flutter config:
  porter_gui/lib/config/app_config.dart         ← Backend URLs, feature flags
```

**9 files in 6 directories** — all need to be consistent with each other for a given deployment. None of them reference each other. No validation that they're consistent. No record of when they were last changed.

---

## 3. VCMS Architecture

### Repository Structure

```
virtus-configs/                     ← Standalone Git repository (private)
├── README.md
├── schema/
│   ├── deployment.schema.json      ← JSON Schema for deployment profiles
│   └── validate.py                 ← Schema validation script
├── defaults/
│   ├── firmware.yaml               ← Default firmware config (all deployments start here)
│   ├── ros2.yaml                   ← Default ROS 2 params
│   ├── nav2.yaml                   ← Default Nav2 params
│   └── ai.yaml                     ← Default AI model config
├── configs/
│   └── deployments/
│       ├── dev-laptop.yaml         ← Antony's dev machine config
│       ├── dev-robot.yaml          ← Development robot config
│       ├── cial-unit-001.yaml      ← CIAL deployment #1
│       ├── cial-unit-002.yaml      ← CIAL deployment #2
│       └── kempegowda-unit-001.yaml ← Kempegowda deployment
├── tools/
│   ├── virtus-deploy.py            ← Main deployment script
│   ├── virtus-diff.py              ← Show diff between two deployment profiles
│   ├── virtus-audit.py             ← Query deployment audit log
│   └── virtus-validate.py          ← Validate a profile against schema
└── audit/
    └── deployment-log.jsonl        ← Append-only deployment audit log
```

---

## 4. Deployment Profile Schema

### Full Profile Example

```yaml
# configs/deployments/cial-unit-001.yaml
# CIAL (Kochi International Airport) — Domestic Terminal, Unit #1
# Last deployed: 2026-03-18 by Antony Austin
# Profile version: 1.3.0

meta:
  robot_id:       "virtus-cial-001"
  robot_name:     "CIAL Porter #1"
  airport:        "CIAL"
  terminal:       "domestic"
  profile_version: "1.3.0"
  base_profiles:  ["defaults/firmware.yaml", "defaults/ros2.yaml",
                   "defaults/nav2.yaml", "defaults/ai.yaml"]
  # Values here override the base profiles

# ── Firmware ──────────────────────────────────────────────────────
firmware:
  motor_controller:
    heartbeat_ms:       500
    max_speed_pct:      80          # Reduced for crowded terminal
    ramp_ms_per_step:   10
    estop_current_ma:   6000
    # DTS pins stay at defaults — same PCB rev

  sensor_fusion:
    kalman_process_noise:   0.1
    kalman_measurement_noise: 0.5
    obstacle_near_cm:       50      # Tighter threshold for narrow corridors
    obstacle_medium_cm:     150

# ── ROS 2 Nodes ───────────────────────────────────────────────────
ros2:
  lidar:
    port:         "/dev/ttyUSB0"
    baudrate:     230400
    range_max:    8.0              # CIAL terminal is wide — extend range
    frame_id:     "laser_frame"

  orchestrator:
    health_check_timeout:   5.0
    recovery_attempts:      3
    stuck_timeout_s:        4.0

  esp32_bridge:
    port_esp1:    "/dev/ttyUSB1"
    port_esp2:    "/dev/ttyUSB2"
    baud:         115200

  telemetry:
    publish_hz:   10
    telemetry_port: "/dev/ttyUSB3"

# ── Navigation (Nav2) ─────────────────────────────────────────────
nav2:
  map_file:       "maps/cial_domestic_t1.yaml"
  initial_pose:
    x: 0.0
    y: 0.0
    yaw: 0.0
    frame: "map"

  controller:
    max_vel_x:    0.4              # Slower in crowded terminal
    min_vel_x:    0.1
    max_vel_theta: 0.8

  costmap:
    local:
      width:       3.0
      height:      3.0
      resolution:  0.05
      inflation_radius: 0.35      # Slightly larger inflation for safety
    global:
      resolution:  0.05

# ── AI Model ─────────────────────────────────────────────────────
ai:
  model:          "qwen2.5-1.5b-virtus-v4"
  runtime:        "hailo"         # "hailo" or "gguf"
  gguf_fallback:  "virtus-v4.Q4_K_M.gguf"

  languages:
    primary:      "ml"            # Malayalam — dominant at CIAL
    supported:    ["ml", "en", "hi"]

  passenger_system_prompt: |
    You are Virtus, an autonomous luggage porter robot at
    Kochi International Airport. You assist passengers in
    Malayalam (preferred), English, and Hindi. Always be
    concise and accurate about airport information.

  context:
    airport_name: "Kochi International Airport"
    terminal:     "Domestic Terminal"
    airline_counters: ["Air India", "IndiGo", "SpiceJet", "GoFirst"]
    gates:        "A1-A12, B1-B8"

# ── Fleet / Connectivity ──────────────────────────────────────────
fleet:
  backend_url:    "https://api.virtusco.in"
  heartbeat_s:    30
  ssh_user:       "pi"
  ssh_host:       "virtus-cial-001.local"  # mDNS hostname

# ── Feature Flags ─────────────────────────────────────────────────
features:
  weighing_scale:   false          # Hardware not installed on this unit
  hailo_llm:        true
  escort_mode:      true
  manual_override:  true
  debug_logging:    false          # Production — disable verbose logs
```

---

## 5. Config Layers

VCMS uses an **override model** — deployment profiles only specify values that differ from defaults. This keeps profiles small and readable.

```
defaults/firmware.yaml      ← Base values for all robots
        +
defaults/ros2.yaml          ← Base ROS 2 params
        +
defaults/nav2.yaml          ← Base Nav2 params
        +
defaults/ai.yaml            ← Base AI config
        ↓ merged by virtus-deploy.py
        +
configs/deployments/cial-unit-001.yaml  ← Overrides only
        ↓
Resolved full config (in memory — never written to disk as a merged file)
        ↓
Applied to: firmware / ROS 2 params / Nav2 YAML / AI model selection
```

**Merge rules:**
- Scalar values: deployment profile wins over default
- Lists: deployment profile **replaces** the default list entirely (never merged)
- Maps: deep merge — only specified keys are overridden

---

## 6. Deployment Script

```python
#!/usr/bin/env python3
# tools/virtus-deploy.py

import argparse, yaml, json, subprocess, paramiko, sys
from datetime import datetime
from pathlib import Path

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--profile', required=True)
    parser.add_argument('--target',  help='SSH host (overrides profile fleet.ssh_host)')
    parser.add_argument('--dry-run', action='store_true')
    parser.add_argument('--operator', default='unknown')
    args = parser.parse_args()

    # 1. Load and resolve config
    config   = resolve_config(args.profile)
    robot_id = config['meta']['robot_id']
    ssh_host = args.target or config['fleet']['ssh_host']

    print(f"Deploying {robot_id} → {ssh_host} (profile v{config['meta']['profile_version']})")

    if args.dry_run:
        print("[DRY RUN] Would apply:")
        print(yaml.dump(config, indent=2))
        return

    # 2. Validate config against schema
    validate_config(config)

    # 3. Apply each layer
    apply_firmware_config(config['firmware'])
    apply_ros2_params(config['ros2'], ssh_host)
    apply_nav2_config(config['nav2'], ssh_host)
    apply_ai_config(config['ai'], ssh_host)

    # 4. Restart robot services
    restart_services(ssh_host, config)

    # 5. Health check
    health_check(ssh_host, timeout_s=30)

    # 6. Audit log
    log_deployment(robot_id, args.profile, config['meta']['profile_version'], args.operator)

    print(f"✓ Deployment complete: {robot_id}")


def apply_ros2_params(ros2_config: dict, ssh_host: str):
    """Push ROS 2 params to robot via SSH"""
    ssh = paramiko.SSHClient()
    ssh.connect(ssh_host, username='pi', key_filename=get_ssh_key())

    for node, params in ros2_config.items():
        for key, value in flatten_dict(params).items():
            cmd = f'ros2 param set /{node} {key} {value}'
            stdin, stdout, stderr = ssh.exec_command(cmd)
            if stdout.channel.recv_exit_status() != 0:
                raise DeployError(f"Failed to set {node}/{key}: {stderr.read()}")


def apply_nav2_config(nav2_config: dict, ssh_host: str):
    """Generate Nav2 YAML and push to robot"""
    nav2_yaml = yaml.dump(nav2_to_ros2_format(nav2_config))
    push_file(ssh_host, nav2_yaml.encode(), '/opt/virtus/config/nav2_params.yaml')


def log_deployment(robot_id, profile_path, version, operator):
    """Append to audit log"""
    entry = {
        't':           datetime.utcnow().isoformat(),
        'robot_id':    robot_id,
        'profile':     profile_path,
        'version':     version,
        'operator':    operator,
        'git_commit':  get_git_commit(),
    }
    with open('audit/deployment-log.jsonl', 'a') as f:
        f.write(json.dumps(entry) + '\n')
```

---

## 7. Config Validation

Before any deployment, the profile is validated against a JSON Schema:

```python
# tools/virtus-validate.py
import jsonschema, yaml, json

SCHEMA = json.load(open('schema/deployment.schema.json'))

def validate_config(config: dict):
    jsonschema.validate(config, SCHEMA)

    # Business rule validations (not expressible in JSON Schema)
    errors = []

    if config['ai']['runtime'] == 'hailo' and not config['features']['hailo_llm']:
        errors.append("ai.runtime=hailo but features.hailo_llm=false — contradiction")

    if config['nav2']['controller']['max_vel_x'] > 1.0:
        errors.append(f"max_vel_x={config['nav2']['controller']['max_vel_x']} exceeds safety limit 1.0 m/s")

    if config['firmware']['motor_controller']['estop_current_ma'] > 8000:
        errors.append("estop_current_ma exceeds BTS7960 safe limit (8000mA)")

    if 'map_file' in config['nav2']:
        map_path = Path('configs') / config['nav2']['map_file']
        if not map_path.exists():
            errors.append(f"Map file not found: {map_path}")

    if errors:
        raise ConfigValidationError('\n'.join(errors))
```

---

## 8. Version Control Strategy

### Git Workflow for Configs

```
virtus-configs/
├── main branch        ← Production-deployed configs only
├── dev branch         ← Testing new configs before deployment
└── feature branches   ← One per config change e.g. "cial-001-nav2-tuning"
```

**Every deployment is a Git commit.** The deployment script automatically commits the audit log entry and tags the commit with `deploy/{robot_id}/{timestamp}`.

```bash
# After successful deployment:
git add audit/deployment-log.jsonl
git commit -m "deploy: cial-unit-001 v1.3.0 by Antony (2026-03-18)"
git tag "deploy/virtus-cial-001/2026-03-18T14:32:00"
git push
```

This means you can always answer: "What config was running on cial-unit-001 at 14:32 on March 18?" by running `git log --tags deploy/virtus-cial-001/*`.

### Config Diff Tool

```bash
# Show what changed between two deployed versions
python3 tools/virtus-diff.py cial-unit-001 v1.2.0 v1.3.0

# Output:
# nav2.controller.max_vel_x:    0.5 → 0.4
# ai.languages.primary:         "en" → "ml"
# features.debug_logging:       true → false
```

---

## 9. Runtime Config Access

On the running robot, the resolved config is accessible to all ROS 2 nodes via a lightweight config service:

```python
# On robot — porter_config/config_node.py
# Serves the active config as a ROS 2 service

from virtus_msgs.srv import GetConfig
import yaml, rclpy

class ConfigNode(rclpy.Node):
    def __init__(self):
        super().__init__('config_node')
        self.config = yaml.safe_load(open('/opt/virtus/config/active.yaml'))
        self.srv = self.create_service(GetConfig, '/config/get', self._handle_get)

    def _handle_get(self, request, response):
        # Returns JSON-serialized value for a dot-notation key
        # e.g. request.key = "nav2.controller.max_vel_x"
        value = self._get_nested(self.config, request.key.split('.'))
        response.value_json = json.dumps(value)
        response.found = value is not None
        return response
```

Any node can query its configuration at runtime:

```python
# In porter_orchestrator
config_client = self.create_client(GetConfig, '/config/get')
req = GetConfig.Request(key='orchestrator.recovery_attempts')
resp = config_client.call(req)
self.recovery_attempts = json.loads(resp.value_json)
```

This means no hardcoded values anywhere in the ROS 2 nodes — all come from VCMS.

---

## 10. Config Change Audit Log

Every deployment appended to `audit/deployment-log.jsonl`:

```json
{"t":"2026-03-18T14:32:00Z","robot_id":"virtus-cial-001","profile":"configs/deployments/cial-unit-001.yaml","version":"1.3.0","operator":"antony","git_commit":"a3f9c12"}
{"t":"2026-03-19T09:15:00Z","robot_id":"virtus-cial-002","profile":"configs/deployments/cial-unit-002.yaml","version":"1.3.0","operator":"antony","git_commit":"a3f9c12"}
```

Query the audit log:

```bash
# What was deployed to cial-001 in the last 7 days?
python3 tools/virtus-audit.py --robot virtus-cial-001 --since 7d

# Who deployed what last week?
python3 tools/virtus-audit.py --since 7d --format table
```

---

## 11. Integration with DevTools Suite

VCMS integrates with the VS Code DevTools Suite extensions:

- **Simulation Manager** reads `configs/deployments/dev-robot.yaml` to auto-configure Nav2 params in the editor
- **Fleet Monitor** reads `audit/deployment-log.jsonl` to show deployment history per robot
- **AI Studio** reads `ai.model` from the active profile to know which model to deploy
- **Firmware Builder** reads `firmware.*` from the active profile to pre-fill DTS and Kconfig values

The DevTools Suite master extension exposes a `SharedConfig.getActiveProfile(robotId)` method that all sub-extensions use.

---

## 12. Phased Build Plan

### Phase 1 — Schema + Default Profiles (2 days)
- Define `deployment.schema.json`
- Write default profiles for all 4 layers (firmware, ros2, nav2, ai)
- Write `dev-robot.yaml` deployment profile
- `virtus-validate.py` script
- **Deliverable:** Config schema validated; dev robot has a defined profile

### Phase 2 — Deploy Script (3 days)
- `virtus-deploy.py` — merge + push ROS 2 params + Nav2 YAML + restart services
- SSH integration (paramiko)
- Dry-run mode
- Health check after deployment
- **Deliverable:** One command deploys full config to robot

### Phase 3 — Audit Log + Git Integration (1 day)
- Audit log append on deployment
- Git commit + tag automation
- `virtus-audit.py` query tool
- **Deliverable:** Every deployment is traceable

### Phase 4 — Runtime Config Service (2 days)
- `porter_config` ROS 2 package with config node
- `GetConfig` service (in VDL)
- All existing nodes migrated to use config service (no hardcoded values)
- **Deliverable:** No hardcoded configuration in ROS 2 nodes

### Phase 5 — Multi-Deployment Profiles (1 day)
- First airport deployment profile (CIAL or Kempegowda)
- `virtus-diff.py` for comparing profiles
- **Deliverable:** Production-ready config for first pilot

---

## 13. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| Profile format | YAML | Human-readable, comments supported, Git-diffable |
| Schema validation | JSON Schema + custom Python rules | jsonschema for structure, Python for business rules |
| SSH deployment | paramiko | Pure Python SSH, no subprocess dependency |
| Config merging | deepmerge (Python) | Correct deep merge semantics for nested YAML |
| Audit log | JSONL | Appendable, queryable, never edited |
| Version control | Git (existing) | Leverages existing workflow, free audit trail |
| Runtime config | ROS 2 service + YAML | Standard ROS 2 pattern |

---

## 14. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Bad config deployed to production robot | Medium | High | Schema validation + dry-run + mandatory health check after deploy |
| SSH key for deployment stored insecurely | Medium | Critical | Keys in `~/.ssh/` never in repo; `.gitignore` enforced |
| Nav2 config push fails mid-deployment | Low | Medium | Atomic deploy: backup current config, apply new, rollback on failure |
| Config drift: robot config differs from Git | Medium | High | `virtus-audit.py` drift detector: reads robot's active config hash, compares to Git |
| Two people deploy to same robot simultaneously | Low | High | Deployment takes a Git lock file on the robot's profile during deploy |
| Airport IT blocks SSH on port 22 | Medium | Medium | Support SSH on port 443 (alternate) as fallback; document firewall requirements |
