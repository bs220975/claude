---
name: project-pi5-reinstall-fixes
description: Post-OS-reinstall fixes applied to Pi5 on 2026-06-16
metadata:
  node_type: memory
  type: project
  originSessionId: e02bd63d-90a1-4287-8927-4d3a70e52cd3
---

## Issues Found After OS Reinstall (all fixed 2026-06-16)

### 1. wifi-best-signal.sh missing
`wifi-best-signal.service` (enabled) failed with `status=203/EXEC` because `/usr/local/bin/wifi-best-signal.sh` was missing.
**Fix:** `sudo cp /home/pi5/pi5_drive/Git_projects/RASPI5-MAIN/shell_scripts/wifi-best-signal.sh /usr/local/bin/wifi-best-signal.sh && sudo chmod +x`

### 2. mqttdatainflux.service broken
- `MQTT_ADDRESS = '192.168.1.122'` was hardcoded (Pi4's IP) → should be `localhost`
- Script exited with code 0 on network errors → `Restart=on-failure` never triggered
- `StartLimitIntervalSec` was in `[Service]` section (wrong) → moved to `[Unit]`
**Fix:** Updated `influx_aws_publish/influxdb2_aws_publish.py` and service file.

### 3. Pi5 mybot MQTT_HOST wrong
`MQTT_HOST=localhost` in `/etc/systemd/system/mybot.service` meant Pi5 connected to its own (empty) broker.
**Fix:** Changed to `MQTT_HOST=192.168.1.100` (VIP). Pi5 mybot now receives all ESP32 MQTT events via Pi4's broker.

## Current Status
- mybot.service: running, `MQTT_HOST=192.168.1.100` (VIP) — verified receiving radar MQTT events
- mqttdatainflux.service: running, connected to localhost
- wifi-best-signal.service: script installed

## VIP Note
Earlier session saw stale ARP cache (`84:1f:e8:69:5f:50`) making VIP appear broken. This was NOT a real device conflict. After `sudo arp -d 192.168.1.100` and fresh ping, VIP works correctly. Pi4 (MAC `d8:3a:dd:3c:80:95`) owns VIP.
