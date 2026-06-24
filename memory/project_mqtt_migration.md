---
name: project-mqtt-migration
description: "MQTT migration Option B+C — ESP32-RADAR removes AWS+Firebase stream, ESP01-RELAY adds MQTT primary path, Pi bridges radar motion to porch relay"
metadata: 
  node_type: memory
  type: project
  originSessionId: 62d22a9a-2d08-4acf-9796-4c6b0171346e
---

All three repos committed and pushed (2026-05-14).

**ESP32-RADAR (Option B):**
- Removed: AWSService (persistent TLS), Firebase RTDB SSE stream for lobby2 state
- Kept: Telegram direct, Firebase direct writes (pushRelayStateToFirebase), HTTP fallback state endpoint for ESP01
- Added: publish `home/esp32/radar1/motion` ON/OFF via shared mqttClient (same as DHT11)
- Night-time logic: kept on ESP32 for HTTP fallback path; MQTT path lets Pi check night time
- `RELAY_PROXY_TEST_MODE 1` still set in HardwareConfig.h (office testing — skips HTTP to ESP01)

**ESP01-RELAY (Option C):**
- Added: PubSubClient MQTT, subscribe `home/esp01/porch/cmd/relay` (QoS 1)
- Publishes: `home/esp01/porch/state` (retained), `home/esp01/porch/availability` (LWT)
- HTTP poll of ESP32-RADAR (5s interval): only runs when MQTT disconnected
- 3-min auto-off timer: always active regardless of control path

**RASPI4-MAIN:**
- `mqtt_bridge.py`: added porch + radar topics, `send_porch_relay()`, all callbacks
- `firebase_logger.py`: added `push_porch_relay_status()` (writes `/devices/esp01_relay/`), refactored `start_command_stream` to support multiple simultaneous SSE streams by key
- `main.py`: added `_on_radar_motion` (night check + 5-min off timer), `_on_porch_mqtt_state` (Firebase update), `_on_firebase_porch_light_cmd` / `_execute_porch_cmd` (lights/lobby2 SSE → porch relay)

**Firebase paths:**
- Lobby light (ESP01-LL-RLY at .85): `lights/living_room/state`, `/devices/ESP01-LL-RLY/`
- Porch light (ESP01-RELAY at .111): `lights/lobby2/state`, `/devices/esp01_relay/`

**Pi-down fallback:**
- ESP01-RELAY polls ESP32-RADAR HTTP `/relay/lobby/state` every 5s when MQTT unavailable
- ESP32-RADAR maintains `s_lobbyCommandedState` with night-time + 5-min timer logic
- Telegram alerts + Firebase direct writes remain on ESP32 (Pi-independent)

**Why:** ESP32-RADAR was using ~160KB heap for 5 concurrent TLS connections (AWS persistent, Firebase SSE, Firebase REST, Telegram, OTA), leaving little margin on a ~300KB free device.
