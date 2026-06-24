---
name: project-e2e-test-pending
description: Porch relay end-to-end test pending — ESP devices not yet reflashed with MQTT firmware
metadata: 
  node_type: memory
  type: project
  originSessionId: 62d22a9a-2d08-4acf-9796-4c6b0171346e
---

As of 2026-05-14, all three repos have been updated and pushed with the Option B+C MQTT
migration (see [[project-mqtt-migration]]). The Pi (main.py) is running with the new code.

**What is NOT done yet — needs to happen before full E2E test:**
1. Flash ESP01-RELAY with new MQTT firmware (`pio run -e esp01_relay_ota -t upload` or USB)
2. Flash ESP32-RADAR with new MQTT firmware (`pio run -e esp32_radar_ota -t upload` or USB)
3. Set `RELAY_PROXY_TEST_MODE 0` in ESP32-RADAR `src/config/HardwareConfig.h` before flashing (currently `1` — skips HTTP to ESP01)

**Recommended flash order:**
1. ESP01-RELAY first (simpler; check port 80 open = relay fw, port 8266 open = bootstrap still running)
2. ESP32-RADAR second (set DEVICE_RADAR_ID + RELAY_PROXY_TEST_MODE before build)

**What CAN be tested before flashing (Pi side only):**
- Manually publish `home/esp32/radar1/motion ON` → check Pi sends `home/esp01/porch/cmd/relay ON` (night hours only)
- Manually publish `home/esp01/porch/state ON` → check Firebase `/devices/esp01_relay/` updated
- Subscribe to `home/esp01/porch/cmd/relay` to watch Pi commands

**Why:** Devices haven't been reflashed yet — old firmware still running on both ESPs.
**How to apply:** When user says "continue E2E test" or "test porch relay", start by asking if devices have been flashed, or check port 80/8266 on 192.168.1.111 to detect firmware state.
