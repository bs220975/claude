---
name: project-pi5-reinstall-fixes
description: Post-OS-reinstall fixes applied to Pi5 on 2026-06-16 — services and configs broken after reinstall
metadata:
  type: project
---

## Issues Found After OS Reinstall (all fixed 2026-06-16)

### 1. wifi-best-signal.sh missing
`wifi-best-signal.service` (enabled) failed with `status=203/EXEC` — `/usr/local/bin/wifi-best-signal.sh` was missing.
**Fix:** `sudo cp RASPI5-MAIN/shell_scripts/wifi-best-signal.sh /usr/local/bin/wifi-best-signal.sh && sudo chmod +x`

### 2. mqttdatainflux.service broken
- `MQTT_ADDRESS = '192.168.1.122'` hardcoded (Pi4's IP) → should be `localhost`
- Script exited with code 0 on network errors → `Restart=on-failure` never triggered
- `StartLimitIntervalSec` was in `[Service]` section (wrong) → moved to `[Unit]`
**Fix:** Updated `influx_aws_publish/influxdb2_aws_publish.py` and service file.

### 3. Pi5 mybot MQTT_HOST
**Correct value:** `MQTT_HOST=localhost` in `/etc/systemd/system/mybot.service`
Pi5 has its own Mosquitto broker that bridges from Pi4 — so `localhost` is correct.
(An earlier session briefly set it to VIP .100 — that was wrong. localhost is the correct architecture.)

### 4. Python venv must include system site-packages
picamera2 is installed via apt (system-wide). Venv must be created with:
```bash
python3 -m venv --system-site-packages /home/pi5/myenv
```
NEVER use `python3 -m venv /home/pi5/myenv` alone — breaks video recording.

**Why:** After reinstall all these configs reset to defaults or were missing from the new OS image.
**How to apply:** If mybot.service fails after a Pi5 reinstall, check these four items first.
