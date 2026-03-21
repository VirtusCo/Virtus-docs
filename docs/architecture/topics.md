# ROS 2 Topic Flow

All inter-node communication in Porter uses ROS 2 topics, services, and actions. This page documents the complete topic map.

## Topic Flow Diagram

```
YDLIDAR
  └──> ydlidar_driver
         ├── /scan ──────────────────> porter_lidar_processor
         │                                └── /scan/processed ──> Nav2
         └── /diagnostics ──────────> lidar_health_monitor
                                          └── /porter/health_status ──> porter_orchestrator

Nav2
  └── /cmd_vel ──────────────────────> esp32_motor_bridge ──serial──> ESP32 #1

ESP32 #2 ──serial──> esp32_sensor_bridge
                        └── /environment ──> Nav2 (obstacle layer)

Flutter GUI
  ├── /porter/ai_query ──────────────> porter_ai_assistant
  │                                       └── /porter/ai_response ──> Flutter GUI
  └── HTTP POST /api/chat/stream ────> ai_server (SSE streaming)
```

## Topic Reference

| Topic | Message Type | Publisher | Subscriber(s) | QoS |
|-------|-------------|-----------|---------------|-----|
| `/scan` | `sensor_msgs/LaserScan` | `ydlidar_driver` | `porter_lidar_processor`, Nav2 | SensorData (RELIABLE) |
| `/scan/processed` | `sensor_msgs/LaserScan` | `porter_lidar_processor` | Nav2 | SensorData (RELIABLE) |
| `/diagnostics` | `diagnostic_msgs/DiagnosticArray` | `ydlidar_driver` | `lidar_health_monitor` | Default |
| `/porter/health_status` | `std_msgs/String` | `lidar_health_monitor` | `porter_orchestrator` | Default |
| `/cmd_vel` | `geometry_msgs/Twist` | Nav2 Controller | `esp32_motor_bridge` | Default |
| `/environment` | `sensor_msgs/Range` | `esp32_sensor_bridge` | Nav2 Costmap | SensorData |
| `/porter/ai_query` | `std_msgs/String` | Flutter GUI | `porter_ai_assistant` | Default |
| `/porter/ai_response` | `std_msgs/String` | `porter_ai_assistant` | Flutter GUI | Default |

## QoS Notes

!!! info "LIDAR QoS"
    The `/scan` topic uses `rclcpp::SensorDataQoS().reliable()` to match RViz2's RELIABLE subscription policy. Using BEST_EFFORT causes RViz2 to silently drop the subscription.

## Services

| Service | Type | Node | Purpose |
|---------|------|------|---------|
| `~/query` | `std_srvs/String` | `porter_ai_assistant` | Synchronous AI query |
| `~/get_status` | `std_srvs/Trigger` | `porter_ai_assistant` | Engine status check |

## HTTP Endpoints (AI Server)

The AI assistant also exposes a standalone HTTP server for the Flutter GUI:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/chat` | POST | Synchronous chat with multi-turn history |
| `/api/chat/stream` | POST | SSE streaming (token-by-token) |
| `/api/health` | GET | Health check |
| `/api/status` | GET | Engine status and model info |
