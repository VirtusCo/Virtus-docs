# Virtus SDK

The Virtus SDK provides a REST API server (deployed on the robot) and installable client libraries (Python and Dart) for airport operators and integrators to interact with Porter programmatically.

## Architecture

```
Airport Operator System
    |
    v
virtus-sdk (Python client)          Flutter GUI
    |                                    |
    v                                    v
┌────────────────────────────────────────────┐
│        Virtus API Server (FastAPI)         │
│        Running on Raspberry Pi 5           │
│                                            │
│  /v1/status    /v1/navigate   /v1/health   │
│  /v1/escort    /v1/battery    /v1/sensors  │
└────────────────┬───────────────────────────┘
                 │
                 v
          ROS 2 Bridge (ros2_bridge.py)
                 │
                 v
          ROS 2 Topics & Services
```

## API Server

The server is a FastAPI application running on the Raspberry Pi alongside the ROS 2 stack. It bridges HTTP requests to ROS 2 topics and services.

| Feature | Detail |
|---------|--------|
| **Framework** | FastAPI with Pydantic models |
| **Auth** | 3-tier API key authentication (read, write, admin) |
| **Spec** | OpenAPI 3.0 (580-line spec in `docs/openapi.yaml`) |
| **Deployment** | systemd service via `install.sh` |

## Python Client

```bash
pip install virtus-sdk
```

```python
from virtus_sdk import PorterClient

client = PorterClient(host="192.168.1.100", api_key="your-key")

# Get robot status
status = client.get_status()
print(f"Battery: {status.battery_pct}%, State: {status.state}")

# Send navigation goal
client.navigate(gate="C5")

# Start escort mode
client.escort(passenger_id="PAX-1234")
```

## Dart Client

For use in the Flutter GUI and third-party Dart/Flutter applications:

```dart
final client = VirtusClient(host: '192.168.1.100', apiKey: 'your-key');
final status = await client.getStatus();
```

## Versioning

The SDK follows independent semantic versioning, decoupled from the robot software version:

```
SDK v1.0.0  -- compatible with robot v0.3.0 through v0.9.x
SDK v1.1.0  -- adds new endpoint (non-breaking)
SDK v2.0.0  -- breaking change (v1 served for 6 months)
```

## Test Suite

81 tests covering client methods, Pydantic model validation, and authentication.

```bash
cd virtus-sdk && pytest tests/ -v
```

!!! info "Runtime dependency"
    The API server imports `virtus_msgs` from the ROS 2 workspace. It runs inside the same Docker container as the robot software.
