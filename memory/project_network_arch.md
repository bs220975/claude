---
name: project-network-arch
description: "Dual Pi network architecture — keepalived DISABLED 2026-06-21, Pi4 is fixed MQTT hub at .122, Pi5 bridges from Pi4"
metadata:
  node_type: memory
  type: project
  originSessionId: e02bd63d-90a1-4287-8927-4d3a70e52cd3
---

## Current Architecture (as of 2026-06-24)

- **Keepalived DISABLED** 2026-06-21 — JioFiber IP source guard blocked VIP traffic (router only forwards from Pi4's DHCP MAC/IP lease; VIP replies were dropped)
- VIP `192.168.1.100` still exists as a **static secondary IP** on Pi4's wlan0 (via NetworkManager) for backward compat with app configs — but it is NOT VRRP anymore
- **Pi4 (192.168.1.122)** is the fixed primary MQTT broker — all ESP devices connect here
- **Pi5 (192.168.1.108)** has `MQTT_HOST=localhost` in mybot.service; its mosquitto bridges specific topics from Pi4

## Pi5 Mosquitto Bridge (bridge_pi4.conf)

- IN from Pi4: `home/esp32/radar1/#` — radar motion for Pi5 camera
- IN from Pi4: `home/switches/L-Porch-Light/state` + `availability` — LP relay state
- OUT to Pi4: `home/switches/L-Porch-Light/cmd` — LP relay commands
- OUT to Pi4: `home/esp01/upper-lobby/cmd/#`, `home/esp01/lower-lobby/cmd/#` — relay commands

## SSH access

- Pi4: `ssh pi@192.168.1.122` password `22`
- Pi5: local or `ssh pi5@192.168.1.108`

## Dual Pi design intent

- Pi4 handles: MQTT broker, light control logic, LD2420 radar, Firebase SSE, relay commands
- Pi5 handles: camera recording (picamera2), triggered by `home/pi5/lp-rdr/motion` from ESP32-LP-RDR1 direct, and `home/esp32/radar1/motion` via bridge
- Both mybots run simultaneously; Pi5 has VIP master guard (`is_master` check) to avoid fighting Pi4 on relay/Firebase writes
- Radar motion commands (ON/OFF for lights) are sent by both Pis — idempotent by design
