# Proposal: Move Sensor Telemetry to MQTT

Status: **Proposal (not implemented)** · Targets the *IoT Term Project* criteria, items 10–12.

## 1. Problem

The rubric asks for a message queue (e.g. MQTT) that carries data from IoT sources on **at least 3 topics**, each with a clear purpose, and a server that **receives and processes data from that queue**.

What the code does today (read from the hardware branches):

| Path | Transport today |
|------|-----------------|
| Room node → Main_mesh | Painless Mesh |
| Main_mesh → Main_lcd (gateway) | UART JSON |
| **Gateway → backend (sensor readings)** | **HTTP `POST /api/v1/devices/sensor-data`** |
| Doorbell ring, face detection, hub command nudge | MQTT (public HiveMQ broker) |

MQTT is used only for events/notifications. The backend (`Backend:config/mqtt.js`) is publish-only: it never subscribes, so the server does not receive data from the queue. Sensor telemetry, the main IoT data stream, bypasses MQTT entirely.

## 2. Proposed design

Replace the HTTP sensor upload with MQTT. Keep HTTP for request/response operations.

```
Room nodes ─mesh─▶ Main_mesh ─UART─▶ Main_lcd ─MQTT publish─▶ Broker ─▶ Backend subscriber ─▶ Database ─▶ Dashboard
                                      smarthome/sensors/<device_id>
```

Dual-publishing (HTTP **and** MQTT for the same reading) is rejected: it duplicates ingest paths and stored data.

### Topics

| Topic | Publisher | Subscriber | Purpose |
|-------|-----------|------------|---------|
| `smarthome/sensors/<device_id>` | Gateway (Main_lcd) | Backend | Periodic and averaged telemetry (temperature, humidity, light, CO, PM) |
| `smarthome/alerts/<device_id>` | Gateway (Main_lcd) | Backend | Threshold alerts flagged by room nodes (`alert: true`) so they bypass batching |
| `smarthome/doorbell/ring` | Doorbell LCD | Gateway | Event: visitor pressed the bell (exists) |
| `smarthome/face/detection` | Backend | Gateway | Event: face recognition result (exists) |
| `smarthome/device/<id>/command` | Backend | Gateway/devices | Command notification, device then fetches via HTTP (exists) |

Sensor, alert, event and command topics have distinct data sources and purposes, which satisfies item 10.

### Payload

Same JSON the gateway already builds for HTTP, plus the token:

```json
{
  "device_id": "ss_003",
  "device_type": "sensor",
  "timestamp": 1760000000,
  "forwarded_by": "hb_001",
  "api_token": "<per-device token>",
  "data": { "temperature": 28.4, "humidity": 61.0, "light_lux": 120, "co_ppm": 1.2, "battery_percent": 83 }
}
```

### Changes by component

**Main_lcd (`src/hub_network.cpp`, `src/mqtt_client.cpp`)**
- In `sendMeshSensorData`, publish to `smarthome/sensors/<device_id>` (`smarthome/alerts/<device_id>` when `alert` is set) using the existing `PubSubClient`.
- Call `mqttClient.setBufferSize(1024)`; the default 256 bytes truncates these payloads.
- Fall back to the existing HTTP POST only when MQTT is disconnected.
- Note: `PubSubClient` publishes at QoS 0 only (no delivery guarantee); acceptable for periodic telemetry, and the fallback covers outages.

**Backend (`config/mqtt.js`, `controllers/devices.js`, `server.js`)**
- Keep a **persistent** connection with auto-reconnect (today: `reconnectPeriod: 0`, connect-on-demand) and subscribe to `smarthome/sensors/+` and `smarthome/alerts/+` on connect.
- Extract the body of the `sensor-data` handler (writes `sensors/current` and `sensors/history/readings`, then `updateCache`) into a shared function used by both the HTTP route and the MQTT handler.
- Validate `api_token` against the device record exactly as the HTTP middleware does; drop messages that fail.
- De-duplicate by `device_id` + `timestamp` while the HTTP fallback exists.

**Docs**: update the README data-flow section and slides to show the MQTT telemetry path.

## 3. Security

- The public broker (`broker.hivemq.com`) has no authentication: anyone can read or spoof `smarthome/#`. Per-device tokens in the payload mitigate spoofing but not eavesdropping.
- Recommended: self-host **Mosquitto** with username/password (and TLS if the gateway memory allows). Interim option: a long random topic prefix.
- Separately: `Room_Sensors` hardcodes the mesh password and a device API token and contains `src/api_keys.txt`. Rotate these and move them out of source control.

## 4. Hosting and cost

`Backend/fly.toml` currently scales to zero (`min_machines_running = 0`, `auto_stop_machines = 'stop'`). An MQTT subscriber needs a process that is always running. Options:

| Option | Cost | Notes |
|--------|------|-------|
| A. `min_machines_running = 1` only for the demo/video period, then revert | Low (billed per second; verify current Fly pricing) | Recommended. Optionally shrink the VM from 1 GB to 256 MB after testing |
| B. Run the subscriber elsewhere (laptop, Raspberry Pi, free-tier VM) | None | Keeps the Fly API on scale-to-zero; "ingestion service consumes the queue" |
| C. Keep HTTP for telemetry | None | Fits scale-to-zero but leaves item 10 weak |

The gateway already POSTs a heartbeat to `/devices/heartbeat`; its interval determines how often the machine is awake today (not measured here).

## 5. Related rubric gap: time-series storage (item 12)

Readings are stored in Firestore (`devices/<id>/sensors/history/readings`), which is a document database, not a time-series database. Since the backend will now consume a stream, add a time-series store behind the same shared ingest function (e.g. InfluxDB or TimescaleDB, free cloud tiers exist) and have the dashboard's history charts query it. Firestore can stay for current state and devices.

## 6. Rollout

1. Pick the broker (HiveMQ vs self-hosted Mosquitto) and hosting option (§4).
2. Backend: shared ingest function, persistent subscriber, token validation, tests.
3. Gateway: publish + buffer fix + HTTP fallback.
4. Verify end to end with a room node; confirm readings appear in the dashboard live and in history.
5. Capture topic list and message examples for the slides (item 10 explanation).

## 7. Open questions

- Broker: HiveMQ public (zero setup) or Mosquitto on Fly/other (needs auth)?
- Which hosting option from §4?
- Which time-series database?
- Is the HTTP fallback worth keeping for the final demo?

## 8. Caveat

Based on reading `Main_mesh`, `Room_Sensors`, `Main_lcd`, `Doorbell_lcd` and `Backend` source. I did not run the system, and the two ingest code paths in `controllers/devices.js` (lines ~391 and ~462) should be reviewed before refactoring.
