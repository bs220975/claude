---
name: user-profile
description: "User is a home automation hobbyist running dual Pi setup with ESP32 devices, Firebase, MQTT, Telegram bot, and Pi cameras"
metadata:
  type: user
---

- Home automation system with Pi4 (192.168.1.122) and Pi5 (192.168.1.108) running in parallel
- **Keepalived DISABLED 2026-06-21** — JioFiber IP source guard blocked VIP; VIP .100 kept as static secondary alias on Pi4 wlan0 for app backward compat only
- Pi4 is fixed primary hub: MQTT broker, LD2420 radar, light logic, Firebase SSE
- Pi5 is camera node + backup logic: bridges from Pi4 broker, runs lp_rdr_service separately
- **Both Pis have cameras:** Pi4 CSI camera records lower lobby; Pi5 CSI camera records lower porch
- ESP32 devices: ESP32-LP-RDR1 (192.168.1.87), ESP01-LL-RLY (192.168.1.85), ESP01-UL-RLY (192.168.1.111), ESP32-LP-RLY (192.168.1.89)
- Uses Firebase RTDB, AWS IoT, Telegram bot, InfluxDB, MQTT (Mosquitto)
- SSH to Pi4: `ssh pi@192.168.1.122` (passwordless from Pi5); SSH to Pi5: `ssh pi5@192.168.1.108`
- OS reinstalled on Pi5 in June 2026 — services restored
- Python venv on Pi5 must use `--system-site-packages` so picamera2 (apt) is accessible
