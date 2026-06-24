---
name: project-esp32-radar-mqtt
description: ESP32-RADAR firmware bug — MQTT keepalive timeout caused radar motion to never reach Pi5 camera
metadata: 
  node_type: memory
  type: project
  originSessionId: e02bd63d-90a1-4287-8927-4d3a70e52cd3
---

## Root Cause
ESP32-RADAR (`DHT11_ESP32-10B37A-V1` client) was connecting to MQTT broker but immediately timing out (~23 s = 1.5 × 15 s keepalive). `AWSService::loop()` TLS operations block 20-30 s, and `mqttClient.loop()` was only called at the very END of the main loop — after all AWS ops. The broker timeout expired before loop() could send a ping.

Result: ESP32 never published `home/esp32/radar2/motion = ON` → Pi camera never triggered.

## Fix Applied (2026-06-16)
In `/home/pi5/pi5_drive/Git_projects/ESP32-RADAR/src/main.cpp`:
1. `mqttClient.setKeepAlive(60)` in setup() — raises broker timeout to 90 s
2. `mqttClient.loop()` called at START of loop() (before AWS ops)
3. `mqttClient.loop()` called again immediately after `AWSService::loop()`

**Committed as commit `107bfbb` in ESP32-RADAR repo.**

## Next Step Required
OTA flash the ESP32-RADAR via Telegram command `/ota2` after the firmware is built and uploaded to Firebase Storage. Without reflashing, the fix is only in the repo — the device still runs old firmware.

## Test After Reflash
Use `/test_cam2` Telegram command to ESP32-RADAR bot (suffix 2) — it simulates motion ON for 5 s then OFF. Pi5 should receive MQTT event, record video, and send clip to Pi5 bot.
