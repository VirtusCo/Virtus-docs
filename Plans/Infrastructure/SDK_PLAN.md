# Virtus SDK
### Detailed Development Plan
**Project:** Clean Python + REST API surface exposing Virtus robot capabilities for airport operator integrations, internal tools, and the Flutter GUI  
**Company:** Virtusco | virtusco.in  
**Author:** Antony Austin  
**Version:** 1.0  
**Last Updated:** March 2026  
**Build Priority:** #6 — Build when first external integration is requested

---

## Table of Contents

1. [Overview & Vision](#1-overview--vision)
2. [Why an SDK](#2-why-an-sdk)
3. [SDK Architecture](#3-sdk-architecture)
4. [Core Client API](#4-core-client-api)
5. [REST API Layer (on-robot)](#5-rest-api-layer-on-robot)
6. [Event Streaming API](#6-event-streaming-api)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [SDK Clients by Language](#8-sdk-clients-by-language)
9. [Airport Operator Integration Scenarios](#9-airport-operator-integration-scenarios)
10. [SDK Documentation Strategy](#10-sdk-documentation-strategy)
11. [Versioning & Backwards Compatibility](#11-versioning--backwards-compatibility)
12. [Phased Build Plan](#12-phased-build-plan)
13. [Tech Stack Summary](#13-tech-stack-summary)
14. [Risk Register](#14-risk-register)

---

## 1. Overview & Vision

### What the Virtus SDK Is

A clean, versioned API surface that exposes Virtus robot capabilities to the outside world — airport operator IT systems, the Flutter GUI, the Fleet Monitor backend, and any future third-party integrations. It's the boundary between Virtusco's internal implementation and the world.

```python
# What an airport operator's IT team would write to integrate with Virtus
from virtus_sdk import VirtusClient

robot = VirtusClient(host="virtus-cial-001.local", api_key="...")

status = robot.get_status()
print(f"Battery: {status.battery_pct}%, State: {status.fsm_state}")

task = robot.escort_passenger(from_location="checkin_b", to_location="gate_c5")
task.on_progress(lambda p: print(f"Progress: {p.pct}%"))
task.on_complete(lambda: send_passenger_sms("Your bags have arrived"))
```

### What It Enables

- **Airport operator integrations:** Flight info systems, baggage tracking, passenger apps, gate management — all can interact with Virtus via a documented API
- **Internal consistency:** Flutter GUI, Fleet Monitor, and any future tools all call the same SDK — no duplicated HTTP client code
- **Future business model:** When airport operators pay for the SaaS tier, the SDK is the product they're integrating against
- **Architecture clarity:** Forces a clean boundary inside the codebase — if something can't be expressed through the SDK, it probably shouldn't be an external concern

---

## 2. Why an SDK

### The Current State

Everything that wants to know about or control the robot currently goes through ROS 2 directly:
- Flutter GUI → rosbridge WebSocket → ROS 2 topics
- Fleet Monitor → SSH → ros2 param get
- Hardware Dashboard → Serial port → raw telemetry

This works during development. It breaks down when:
- An airport operator's IT team wants to integrate — they don't run ROS 2
- You want to add a web dashboard — rosbridge isn't designed for cross-origin web clients
- Two tools implement the same HTTP wrapper independently and diverge
- The ROS 2 topic names change — every integration breaks

### The SDK Solves This

```
ROS 2 stack (internal)
    ↑ consumed by
virtus_api_server (on-robot FastAPI)  ← The SDK's server side
    ↑ served to
Virtus SDK clients (Python, Dart, REST)  ← The SDK's client side
    ↑ used by
Flutter GUI, Fleet Monitor, Airport IT, Web Dashboard
```

The ROS 2 internals can change completely. As long as the SDK API is stable, nothing outside breaks.

---

## 3. SDK Architecture

```
virtus-sdk/                           ← Standalone repository
├── server/                           ← Runs on each Virtus RPi 5
│   ├── virtus_api_server/            ← FastAPI application
│   │   ├── main.py                   ← FastAPI app factory
│   │   ├── routers/
│   │   │   ├── status.py             ← GET /status, GET /status/stream (SSE)
│   │   │   ├── commands.py           ← POST /navigate, POST /escort, POST /stop
│   │   │   ├── telemetry.py          ← GET /telemetry (hardware data)
│   │   │   └── config.py             ← GET/PATCH /config
│   │   ├── ros2_bridge.py            ← ROS 2 ↔ FastAPI bridge (rclpy)
│   │   ├── auth.py                   ← API key authentication
│   │   └── models.py                 ← Pydantic request/response models
│   └── install.sh                    ← Install as systemd service
│
├── clients/
│   ├── python/                       ← pip install virtus-sdk
│   │   ├── virtus_sdk/
│   │   │   ├── client.py             ← VirtusClient class
│   │   │   ├── models.py             ← Python dataclasses matching API models
│   │   │   ├── streaming.py          ← SSE event streaming
│   │   │   └── exceptions.py         ← SDK-specific exceptions
│   │   ├── setup.py
│   │   └── README.md
│   │
│   └── dart/                         ← Used by Flutter GUI
│       ├── lib/
│       │   ├── virtus_client.dart
│       │   ├── models.dart
│       │   └── streaming.dart
│       └── pubspec.yaml
│
├── docs/
│   ├── openapi.yaml                  ← OpenAPI 3.0 spec (auto-generated)
│   ├── quickstart.md
│   ├── integration_guide.md          ← For airport operator IT teams
│   └── examples/
│       ├── python_escort.py
│       ├── dart_status_monitor.dart
│       └── curl_examples.sh
└── tests/
    ├── test_client_python.py
    └── test_api_contract.py           ← Contract tests against OpenAPI spec
```

---

## 4. Core Client API

### Python Client

```python
# virtus_sdk/client.py

class VirtusClient:
    """
    Python client for the Virtus Robot API.

    Usage:
        client = VirtusClient(host="virtus-cial-001.local", api_key="...")
        status = client.get_status()
    """

    def __init__(self, host: str, api_key: str, port: int = 8888,
                 timeout: float = 10.0):
        self.base_url = f"http://{host}:{port}/v1"
        self.session  = httpx.Client(
            headers={"X-API-Key": api_key},
            timeout=timeout
        )

    # ── Status ────────────────────────────────────────────────────

    def get_status(self) -> RobotStatus:
        """Get current robot status snapshot"""
        resp = self.session.get(f"{self.base_url}/status")
        resp.raise_for_status()
        return RobotStatus(**resp.json())

    def stream_status(self) -> Iterator[RobotStatus]:
        """Stream real-time status updates via Server-Sent Events"""
        with self.session.stream("GET", f"{self.base_url}/status/stream") as resp:
            for line in resp.iter_lines():
                if line.startswith("data:"):
                    yield RobotStatus(**json.loads(line[5:]))

    # ── Commands ──────────────────────────────────────────────────

    def navigate_to(self, destination: str,
                    escort_mode: bool = False) -> NavigationTask:
        """
        Send robot to a named destination.

        Args:
            destination: Semantic location ID e.g. "gate_c5", "checkin_b"
            escort_mode: If True, robot waits for passenger at destination

        Returns:
            NavigationTask — awaitable task with progress callbacks
        """
        resp = self.session.post(f"{self.base_url}/navigate", json={
            "destination": destination,
            "escort_mode": escort_mode,
        })
        resp.raise_for_status()
        return NavigationTask(self, resp.json()["task_id"])

    def escort_passenger(self, from_location: str, to_location: str,
                         carry_luggage: bool = True) -> EscortTask:
        """
        Full escort task — picks up passenger and luggage, navigates to destination.
        Returns an EscortTask with progress callbacks.
        """
        resp = self.session.post(f"{self.base_url}/escort", json={
            "from":         from_location,
            "to":           to_location,
            "carry_luggage": carry_luggage,
        })
        resp.raise_for_status()
        return EscortTask(self, resp.json()["task_id"])

    def stop(self, reason: str = "") -> bool:
        """Immediately stop current task. Robot returns to IDLE."""
        resp = self.session.post(f"{self.base_url}/stop", json={"reason": reason})
        return resp.status_code == 200

    def emergency_stop(self) -> bool:
        """ESTOP — stops all motors immediately. Use only in emergencies."""
        resp = self.session.post(f"{self.base_url}/estop")
        return resp.status_code == 200

    # ── Telemetry ─────────────────────────────────────────────────

    def get_telemetry(self) -> HardwareTelemetry:
        """Get current hardware telemetry (power, motors, sensors)"""
        resp = self.session.get(f"{self.base_url}/telemetry")
        resp.raise_for_status()
        return HardwareTelemetry(**resp.json())

    def stream_telemetry(self, hz: float = 1.0) -> Iterator[HardwareTelemetry]:
        """Stream hardware telemetry at specified rate"""
        with self.session.stream("GET", f"{self.base_url}/telemetry/stream",
                                  params={"hz": hz}) as resp:
            for line in resp.iter_lines():
                if line.startswith("data:"):
                    yield HardwareTelemetry(**json.loads(line[5:]))

    # ── Task Tracking ─────────────────────────────────────────────

    def get_task_status(self, task_id: str) -> TaskStatus:
        resp = self.session.get(f"{self.base_url}/tasks/{task_id}")
        resp.raise_for_status()
        return TaskStatus(**resp.json())


# ── Task Objects ───────────────────────────────────────────────────────────

class NavigationTask:
    """Represents an active navigation task with callback support"""

    def __init__(self, client: VirtusClient, task_id: str):
        self.client  = client
        self.task_id = task_id
        self._on_progress_cb = None
        self._on_complete_cb = None
        self._on_error_cb    = None

    def on_progress(self, callback: Callable[[TaskProgress], None]) -> 'NavigationTask':
        self._on_progress_cb = callback
        return self  # Fluent API

    def on_complete(self, callback: Callable[[], None]) -> 'NavigationTask':
        self._on_complete_cb = callback
        return self

    def on_error(self, callback: Callable[[str], None]) -> 'NavigationTask':
        self._on_error_cb = callback
        return self

    def wait(self, timeout_s: float = 120.0) -> TaskResult:
        """Block until task completes or times out"""
        with self.client.session.stream(
            "GET", f"{self.client.base_url}/tasks/{self.task_id}/stream"
        ) as resp:
            for line in resp.iter_lines():
                if line.startswith("data:"):
                    event = json.loads(line[5:])
                    if event['type'] == 'progress' and self._on_progress_cb:
                        self._on_progress_cb(TaskProgress(**event['data']))
                    elif event['type'] == 'complete':
                        if self._on_complete_cb:
                            self._on_complete_cb()
                        return TaskResult(success=True)
                    elif event['type'] == 'error':
                        if self._on_error_cb:
                            self._on_error_cb(event['data']['reason'])
                        return TaskResult(success=False, reason=event['data']['reason'])
```

---

## 5. REST API Layer (on-robot)

### FastAPI Server

```python
# server/virtus_api_server/main.py

from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware

def create_app() -> FastAPI:
    app = FastAPI(
        title="Virtus Robot API",
        version="1.0.0",
        description="REST API for Virtus autonomous airport porter robot",
        docs_url="/docs",       # Swagger UI
        redoc_url="/redoc",
    )

    app.add_middleware(CORSMiddleware, allow_origins=["*"])  # Airport IT systems
    app.include_router(status_router,    prefix="/v1")
    app.include_router(commands_router,  prefix="/v1")
    app.include_router(telemetry_router, prefix="/v1")
    app.include_router(config_router,    prefix="/v1")

    return app
```

### Key Endpoints

```python
# server/virtus_api_server/routers/status.py

@router.get("/status", response_model=RobotStatusResponse)
async def get_status(ros2: ROS2Bridge = Depends(get_ros2_bridge)):
    """Get current robot status"""
    state   = await ros2.get_latest(OrchestratorState, '/orchestrator/state')
    metrics = await ros2.get_latest_metrics()
    return RobotStatusResponse(
        robot_id    = settings.ROBOT_ID,
        fsm_state   = STATE_NAMES[state.state],
        current_task = state.current_task,
        battery_pct = metrics.battery_pct,
        position    = metrics.position,
        is_serving  = state.state in SERVING_STATES,
    )

@router.get("/status/stream")
async def stream_status(ros2: ROS2Bridge = Depends(get_ros2_bridge)):
    """Server-Sent Events stream of robot status updates at 1Hz"""
    async def event_generator():
        while True:
            status = await get_status(ros2)
            yield f"data: {status.model_dump_json()}\n\n"
            await asyncio.sleep(1.0)

    return StreamingResponse(event_generator(), media_type="text/event-stream")

@router.post("/navigate", response_model=TaskCreatedResponse)
async def navigate(req: NavigateRequest,
                   ros2: ROS2Bridge = Depends(get_ros2_bridge),
                   _: None = Depends(require_operator_auth)):
    """Send robot to a named destination"""
    if not req.destination in settings.KNOWN_LOCATIONS:
        raise HTTPException(400, f"Unknown destination: {req.destination}")

    task_id = str(uuid4())
    await ros2.call_service(NavigateTo, '/navigate_to', {
        'destination_id': req.destination,
        'escort_mode':    req.escort_mode,
        'session_id':     task_id,
    })
    return TaskCreatedResponse(task_id=task_id)
```

---

## 6. Event Streaming API

All long-running tasks stream progress via SSE:

```
GET /v1/tasks/{task_id}/stream

data: {"type":"progress","data":{"pct":25.0,"phase":"NAVIGATING","eta_s":42,"dist_m":8.2}}
data: {"type":"progress","data":{"pct":60.0,"phase":"NAVIGATING","eta_s":24,"dist_m":4.1}}
data: {"type":"progress","data":{"pct":100.0,"phase":"ARRIVED","eta_s":0,"dist_m":0.0}}
data: {"type":"complete","data":{"total_time_s":38.4,"total_dist_m":12.3}}
```

```
GET /v1/status/stream

data: {"robot_id":"virtus-cial-001","fsm_state":"NAVIGATING","battery_pct":77,...}
data: {"robot_id":"virtus-cial-001","fsm_state":"NAVIGATING","battery_pct":77,...}
...
```

---

## 7. Authentication & Authorization

### API Keys

Three levels of access:

| Level | Can do | Who gets it |
|---|---|---|
| `read` | GET /status, GET /telemetry | Airport operator dashboards, display screens |
| `operator` | All above + POST /navigate, POST /stop | Airport staff apps, passenger-facing kiosks |
| `admin` | All above + POST /estop, PATCH /config | Virtusco team only |

```python
# server/virtus_api_server/auth.py

API_KEYS = {
    # Loaded from /opt/virtus/config/api_keys.yaml — managed by VCMS
    "cial-ops-readonly": {"level": "read",     "name": "CIAL Operations Dashboard"},
    "cial-kiosk-001":    {"level": "operator", "name": "CIAL Kiosk #1"},
    "virtusco-admin":    {"level": "admin",    "name": "Virtusco Admin"},
}

def require_operator_auth(x_api_key: str = Header(...)):
    if x_api_key not in API_KEYS:
        raise HTTPException(401, "Invalid API key")
    if API_KEYS[x_api_key]["level"] not in ["operator", "admin"]:
        raise HTTPException(403, "Insufficient permissions — operator level required")

def require_admin_auth(x_api_key: str = Header(...)):
    if x_api_key not in API_KEYS or API_KEYS[x_api_key]["level"] != "admin":
        raise HTTPException(403, "Admin level required")
```

---

## 8. SDK Clients by Language

### Python Client
Primary — used by Fleet Monitor backend, test automation, CI scripts.
Full async support via `httpx.AsyncClient`.

### Dart Client
Used by Flutter GUI to replace the current rosbridge dependency for status and commands. Rosbridge can remain for the map and sensor visualization (which genuinely needs ROS 2 data), but status and commands go through the SDK.

```dart
// clients/dart/lib/virtus_client.dart

class VirtusClient {
  final String host;
  final String apiKey;
  late final Dio _dio;

  VirtusClient({required this.host, required this.apiKey}) {
    _dio = Dio(BaseOptions(
      baseUrl: 'http://$host:8888/v1',
      headers: {'X-API-Key': apiKey},
    ));
  }

  Future<RobotStatus> getStatus() async {
    final resp = await _dio.get('/status');
    return RobotStatus.fromJson(resp.data);
  }

  Stream<RobotStatus> streamStatus() async* {
    final client = http.Client();
    final request = http.Request('GET', Uri.parse('http://$host:8888/v1/status/stream'));
    request.headers['X-API-Key'] = apiKey;
    final response = await client.send(request);
    await for (final chunk in response.stream.toStringStream()) {
      for (final line in chunk.split('\n')) {
        if (line.startsWith('data:')) {
          yield RobotStatus.fromJson(jsonDecode(line.substring(5)));
        }
      }
    }
  }

  Future<NavigationTask> navigateTo(String destination) async {
    final resp = await _dio.post('/navigate', data: {'destination': destination});
    return NavigationTask(client: this, taskId: resp.data['task_id']);
  }
}
```

### REST (curl/HTTP)
For airport operator IT teams who integrate via generic HTTP — no SDK needed. OpenAPI spec auto-generates documentation at `/docs` on the robot.

---

## 9. Airport Operator Integration Scenarios

### Scenario 1 — Flight Departure Board Integration

CIAL's departure board system knows when a flight is boarding. It calls Virtus to stage robots near the relevant gates:

```python
# Airport IT system code (they write this, not Virtusco)
from virtus_sdk import VirtusFleet

fleet = VirtusFleet(backend_url="https://api.virtusco.in", api_key="cial-ops-key")

def on_boarding_announced(flight: str, gate: str):
    # Move all idle robots near the gate
    idle_robots = [r for r in fleet.all_robots() if r.status.fsm_state == "IDLE"]
    for robot in idle_robots[:2]:  # Stage 2 robots
        robot.navigate_to(f"gate_staging_{gate.lower()}")
```

### Scenario 2 — Passenger App Integration

CIAL's passenger app shows "Request Porter" button. When tapped, it calls Virtus:

```python
# CIAL passenger app backend
@app.post("/request_porter")
def request_porter(passenger_id: str, checkin_counter: str, bag_count: int):
    robot = fleet.get_available_robot()
    task  = robot.escort_passenger(
        from_location = checkin_counter,
        to_location   = f"gate_{get_passenger_gate(passenger_id)}",
        carry_luggage = bag_count > 0,
    )
    task.on_complete(lambda: send_push_notification(passenger_id, "Your bags have arrived"))
    return {"task_id": task.task_id, "eta_s": task.eta_s}
```

### Scenario 3 — Baggage Tracking Integration

Airport baggage system tags bags. When a bag tag is scanned at a robot, the SDK reports the weight:

```python
robot.weigh_luggage()  # Returns weight in kg + raw ADC value
```

---

## 10. SDK Documentation Strategy

### Auto-Generated Docs (Swagger UI)

FastAPI automatically generates OpenAPI 3.0 spec and serves Swagger UI at `/docs`. Every endpoint, request model, response model, and error code documented. Airport IT teams can try API calls directly from the browser.

### Integration Guide

A human-written `integration_guide.md` covering:
- Authentication setup (getting API keys from Virtusco)
- Location IDs for the airport (gate names, zone names)
- Webhook setup for task completion events
- Rate limits (max 10 requests/second)
- Error codes and what to do for each

### SDK Reference (auto-generated from docstrings)
Python SDK docs generated with `pdoc3`. Dart SDK docs with `dart doc`.

---

## 11. Versioning & Backwards Compatibility

### URL Versioning

All endpoints prefixed with `/v1/`. When breaking changes are needed, `/v2/` is added while `/v1/` stays alive for 6 months.

### Response Model Versioning

New fields can be added to response models at any time (additive). Existing fields can never be removed or renamed without a major version bump.

### Client SDK Versioning

SDK follows semver:
- `1.x.y` — compatible with API v1
- `2.x.y` — compatible with API v2

Airport operator IT teams pin to a major version: `virtus-sdk>=1.0,<2.0`.

---

## 12. Phased Build Plan

### Phase 1 — Server Core + Status API (1 week)
- FastAPI app factory + systemd service
- `GET /v1/status` + `GET /v1/status/stream` (SSE)
- API key authentication
- OpenAPI spec auto-generated
- **Deliverable:** Robot status readable via HTTP without ROS 2

### Phase 2 — Command API (1 week)
- `POST /v1/navigate`, `POST /v1/escort`, `POST /v1/stop`, `POST /v1/estop`
- Task tracking + SSE task progress stream
- Navigation task objects with callbacks
- **Deliverable:** Robot controllable via HTTP

### Phase 3 — Python SDK Client (3 days)
- `VirtusClient` Python class
- `NavigationTask` and `EscortTask` objects with fluent callback API
- Contract tests against OpenAPI spec
- Published to private PyPI or installable from GitHub
- **Deliverable:** Python SDK ready for Fleet Monitor + test automation

### Phase 4 — Dart SDK Client (1 week)
- `VirtusClient` Dart class
- Flutter GUI migrated from rosbridge to SDK for status + commands
- rosbridge retained for map/LiDAR visualization only
- **Deliverable:** Flutter GUI uses SDK

### Phase 5 — Telemetry + Config API (3 days)
- `GET /v1/telemetry` + stream
- `GET /v1/config` + `PATCH /v1/config` (admin only)
- Integration with VCMS for config reading
- **Deliverable:** Full API surface complete

### Phase 6 — Integration Guide + Airport Onboarding (3 days)
- Human-written integration guide
- Example scripts for 3 airport integration scenarios
- API key provisioning process documented
- **Deliverable:** Airport IT team can self-serve integrate

---

## 13. Tech Stack Summary

| Concern | Choice | Reason |
|---|---|---|
| Server framework | FastAPI | Auto OpenAPI, Pydantic models, SSE support |
| Python client | httpx | Sync + async, same API |
| Dart client | Dio + http | Flutter standard HTTP client |
| Authentication | API key (X-API-Key header) | Simple, stateless, airport-IT friendly |
| Streaming | Server-Sent Events (SSE) | Simpler than WebSocket for one-way streams |
| Docs | Swagger UI (auto) + Markdown integration guide | Auto for reference, human for onboarding |
| Versioning | URL versioning (/v1/, /v2/) | Explicit, cacheable, no header magic |
| Contract testing | schemathesis | Tests API against OpenAPI spec automatically |

---

## 14. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| API key leaked in airport IT system | Medium | High | Keys rotatable instantly; all requests logged with key ID |
| Airport IT integrates against /v1 then breaks on changes | Medium | High | Never remove fields in v1; additive only; 6-month v1 deprecation window |
| FastAPI server crash leaves robot with no external API | Low | Medium | Systemd `Restart=always`; health endpoint checked by Fleet Monitor |
| ROS 2 bridge becomes bottleneck (too many API clients polling) | Low | Medium | Cache ROS 2 topic values in FastAPI (1s TTL); don't re-subscribe per request |
| Airport IT system makes unauthorized ESTOP calls | Low | Critical | ESTOP requires admin key; admin keys held only by Virtusco; operator keys cannot ESTOP |
| SSE connection drops cause task status loss | Medium | Medium | Clients reconnect and re-poll `GET /tasks/{id}` for current status; SSE is best-effort |
