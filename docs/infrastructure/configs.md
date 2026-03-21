# Configuration Management

Porter's deployment configurations are managed in a dedicated private repository (`virtus-configs`) that handles deployment profiles, schema validation, and audit logging.

## Repository Structure

```
virtus-configs/
├── defaults/
│   ├── firmware.yaml      # Default ESP32 firmware settings
│   ├── ros2.yaml          # Default ROS 2 parameters
│   ├── nav2.yaml          # Default Nav2 parameters
│   └── ai.yaml            # Default AI assistant settings
├── configs/deployments/
│   ├── dev-laptop.yaml    # Developer workstation profile
│   ├── dev-robot.yaml     # Development robot profile
│   ├── cial-unit-001.yaml # CIAL airport deployment
│   └── kempegowda-unit-001.yaml
├── schema/
│   └── deployment.schema.json
├── tools/
│   ├── virtus-deploy.py   # Deployment automation
│   ├── virtus-validate.py # Schema + business rule validation
│   ├── virtus-diff.py     # Compare two deployment profiles
│   └── virtus-audit.py    # Query audit trail
└── audit/
    └── deployment-log.jsonl  # Append-only deployment history
```

## Deployment Profiles

Each deployment profile specifies the full configuration for a specific robot instance:

```yaml
# configs/deployments/cial-unit-001.yaml
robot_id: "CIAL-001"
airport: "Cochin International Airport"
firmware:
  motor_controller: "v0.5.0"
  sensor_fusion: "v0.5.0"
ros2:
  lidar_model: "X4Pro"
  lidar_baudrate: 128000
  scan_frequency: 10.0
nav2:
  max_vel_x: 0.5
  inflation_radius: 0.55
ai:
  model: "qwen2.5-1.5b-dpo-tool"
  n_threads: 2
  n_ctx: 1024
```

## Validation

Profiles are validated against a JSON Schema and business rules before deployment:

```bash
# Validate a profile
python tools/virtus-validate.py configs/deployments/cial-unit-001.yaml

# Compare two profiles
python tools/virtus-diff.py configs/deployments/dev-robot.yaml configs/deployments/cial-unit-001.yaml
```

## Deployment Workflow

```bash
# Deploy to a robot
python tools/virtus-deploy.py \
    --config configs/deployments/cial-unit-001.yaml \
    --target 192.168.1.100
```

The deploy tool:

1. Validates the profile against the schema
2. Pulls the correct firmware and Docker image versions
3. Pushes configuration to the target robot via SSH
4. Restarts affected services
5. Appends an entry to `audit/deployment-log.jsonl`

## Audit Trail

Every deployment is recorded in an append-only JSONL file:

```json
{"timestamp":"2026-03-15T10:30:00Z","robot_id":"CIAL-001","profile":"cial-unit-001","deployer":"antony","version":"v0.5.0","status":"success"}
```

!!! warning "Private repository"
    This repository contains SSH hostnames, network topology, and API key paths. Access is restricted to the operations team.
