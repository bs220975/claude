---
name: user-profile
description: "User is a home automation hobbyist running dual Pi setup with ESP32 devices, Firebase, MQTT, Telegram bot, and Pi camera"
metadata: 
  node_type: memory
  type: user
  originSessionId: e02bd63d-90a1-4287-8927-4d3a70e52cd3
---

- Home automation system with Pi4 (192.168.1.122) and Pi5 (192.168.1.108) running in parallel
- Pi4 is keepalived MASTER (priority 101), Pi5 is BACKUP (priority 100)
- Pi5 has a Pi camera (picamera2) — Pi4 does not
- ESP32 devices: ESP32-RADAR (192.168.1.87), ESP01-LL-RLY (192.168.1.85), ESP01-RELAY (192.168.1.111), ESP32-LP-RLY (192.168.1.89)
- Uses Firebase RTDB, AWS IoT, Telegram bot, InfluxDB, MQTT
- SSH to Pi4: `ssh pi@192.168.1.122` password `22`
- OS reinstalled on Pi5 in June 2026 — some scripts/configs needed restoration
