# Claude Code — Home Automation System Briefing

Auto-loaded every session. Full context about all devices, projects, and how they connect.
**Update this file whenever a device, topic, path, or architecture changes.**

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                     HOME NETWORK (192.168.1.x)                      │
│                     Router/Gateway: 192.168.1.254                   │
│                     WiFi: JioFiber-XtHwH (primary), baljeet22-2g   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────┐     ┌──────────────────────┐             │
│  │  Pi4 — 192.168.1.122 │     │  Pi5 — 192.168.1.108 │             │
│  │  PRIMARY HUB         │     │  CAMERA NODE         │             │
│  │  Mosquitto broker    │◄────│  Bridge → Pi4        │             │
│  │  mybot.service       │────►│  mybot.service       │             │
│  │  InfluxDB            │     │  Pi camera           │             │
│  └──────────┬───────────┘     └──────────┬───────────┘             │
│             │ MQTT 1883                  │ MQTT localhost           │
│             │ (all ESP devices)          │ (bridge from Pi4)        │
│             │                            │                          │
│     ┌───────┴────────────────────────────┴──────────┐              │
│     │               ESP DEVICES                     │              │
│     │                                               │              │
│     │  ESP32-LP-RDR1  192.168.1.87  (main radar)   │              │
│     │  ESP01-LL-RLY   192.168.1.85  (lower lobby)  │              │
│     │  ESP01-UL-RLY   192.168.1.111 (upper lobby)  │              │
│     │  ESP32-LP-RLY   192.168.1.89  (lower porch)  │              │
│     └───────────────────────────────────────────────┘              │
│                                                                     │
│  VIP 192.168.1.100 — static secondary IP on Pi4 wlan0              │
│  (Keepalived DISABLED 2026-06-21; .100 kept for app backward compat)│
└─────────────────────────────────────────────────────────────────────┘

                        ┌──────────────┐
                        │   CLOUD      │
                        ├──────────────┤
                        │ Firebase RTDB│ ◄── ESP32-LP-RDR1 (direct)
                        │ AWS IoT Core │ ◄── ESP32-LP-RDR1 (direct)
                        │ Telegram API │ ◄── ESP32-LP-RDR1 + Pi bots
                        └──────────────┘
                               ▲
                    Pi4 + Pi5 mybot.service
```

---

## Pi4 — Primary Hub

| Item | Value |
|------|-------|
| Hostname | `pi4` |
| IP | `192.168.1.122` |
| User | `pi` |
| SSH | `ssh pi@192.168.1.122` (password: `22`) |
| OS | Debian (aarch64) |
| Python venv | `/home/pi/myenv/` (Python 3.x) |
| Project path | `/home/pi/pi4_drive/Git_projects/RASPI4-MAIN/` |
| GitHub | `https://github.com/bs220975/RASPI4-MAIN` |

### Pi4 — GPIO Pins (BCM)

| GPIO | Device | Direction | Notes |
|------|--------|-----------|-------|
| 14 | LD2420 radar TX | Output (UART) | `/dev/serial0` TX |
| 15 | LD2420 radar RX | Input (UART) | `/dev/serial0` RX |
| 25 | PIR sensor | Input | pull_up=False |
| 27 | MMS microwave sensor | Input | pull_up=False |
| 18 | Status LED | Output | mirrors motion |
| 26 | Reed switch (door) | Input | PUD_UP — HIGH=open, LOW=closed |
| 17 | Case fan | Output | dtoverlay=gpio-fan,gpiopin=17,temp=65000 |

Serial `/dev/serial0` — LD2420 radar (115200 baud). `enable_uart=1` in `/boot/firmware/config.txt`.

### Pi4 — Services

| Service | Role | Notes |
|---------|------|-------|
| `mybot.service` | Main bot (`RASPI4-MAIN/main.py`) | MQTT_HOST=192.168.1.122 |
| `mqttdatainflux.service` | MQTT → InfluxDB bridge | `influx_aws_publish/influxdb2_aws_publish.py` |
| `mosquitto` | MQTT broker `0.0.0.0:1883` | ALL ESP devices connect here |
| `influxdb` | InfluxDB 2.x `http://[::1]:8086` | org: `pi4org`, bucket: `pi4data` |
| `keepalived` | VRRP failover | **DISABLED** 2026-06-21 — IP source guard conflict |
| `fan-shutdown.service` | Drives GPIO17 LOW on shutdown | Prevents fan spin-up after poweroff |

### Pi4 — Key Files

| File | Role |
|------|------|
| `main.py` | Main controller (`RaspberryPiController`) |
| `mqtt_bridge.py` | Paho MQTT — relays + radar subscriptions |
| `firebase_logger.py` | Firebase RTDB writes + SSE command streams |
| `sensors.py` | LD2420 radar, PIR, MMS, reed switch |
| `config.py` | All config constants |
| `bot_commands.py` | Telegram command dispatch |
| `telegram_handler.py` | Telegram API wrapper |
| `local_api_server.py` | LAN fallback HTTP API port 5757 |
| `light_scheduler.py` | APScheduler daily on/off timers |
| `influxdb_logger.py` | Motion events → InfluxDB |
| `video_recorder.py` | Pi camera (not used — Pi4 has no camera) |
| `esp_devices.py` | HTTP fallback for ESP heartbeat |

### Pi4 — Aliases

```bash
cdmain        # cd to RASPI4-MAIN
cdgit         # cd to /home/pi/pi4_drive/Git_projects
servicemybot  # journalctl -u mybot.service -f
statusmqtt    # systemctl status mosquitto
copymybot     # copy mybot.service to /etc/systemd/system/
reloadmybot   # systemctl daemon-reload
```

---

## Pi5 — Camera Node

| Item | Value |
|------|-------|
| Hostname | `pi5` |
| IP | `192.168.1.108` |
| User | `pi5` |
| OS | Debian (aarch64), kernel 6.12.x |
| Python venv | `/home/pi5/myenv/` (Python 3.13, **must** have `include-system-site-packages = true`) |
| Project path | `/home/pi5/pi5_drive/Git_projects/RASPI5-MAIN/` |
| GitHub | `https://github.com/bs220975/RASPI5-MAIN` |

### Pi5 — GPIO Pins (BCM)

| GPIO | Device | Direction | Notes |
|------|--------|-----------|-------|
| 25 | PIR sensor | Input | pull_up=False |
| 27 | MMS microwave sensor | Input | pull_up=False |
| 18 | Status LED | Output | mirrors motion |
| 26 | Reed switch (door) | Input | PUD_UP — HIGH=open, LOW=closed |

Serial `/dev/serial0` — LD2420 radar (115200 baud).

### Pi5 — Services

| Service | Role | Notes |
|---------|------|-------|
| `mybot.service` | Main bot (`RASPI5-MAIN/main.py`) | MQTT_HOST=localhost (Pi5's own broker) |
| `mqttdatainflux.service` | MQTT → InfluxDB bridge | MQTT_HOST=localhost |
| `mosquitto` | MQTT broker `0.0.0.0:1883` | Bridges specific topics from Pi4 |
| `influxdb` | InfluxDB 2.x `http://[::1]:8086` | org: `pi4org`, bucket: `pi4data` |

### Pi5 — Mosquitto Bridge (to Pi4)

Config: `/etc/mosquitto/conf.d/bridge_pi4.conf`

| Direction | Topic | Purpose |
|-----------|-------|---------|
| IN (from Pi4) | `home/esp32/radar1/#` | Radar motion events → Pi5 camera trigger |
| IN (from Pi4) | `home/switches/L-Porch-Light/state` | LP relay state → Pi5 Firebase update |
| IN (from Pi4) | `home/switches/L-Porch-Light/availability` | LP relay availability |
| OUT (to Pi4) | `home/switches/L-Porch-Light/cmd` | Pi5 LP relay commands |
| OUT (to Pi4) | `home/esp01/upper-lobby/cmd/#` | Pi5 upper lobby relay commands |
| OUT (to Pi4) | `home/esp01/lower-lobby/cmd/#` | Pi5 lower lobby relay commands |

### Pi5 — Key Files

| File | Role |
|------|------|
| `main.py` | Main controller |
| `mqtt_bridge.py` | Paho MQTT — subscribes to bridged topics |
| `firebase_logger.py` | Firebase RTDB writes + SSE streams |
| `sensors.py` | LD2420 radar, PIR, MMS, reed switch |
| `config.py` | All config constants |
| `video_recorder.py` | **picamera2 → H264 → ffmpeg MP4** |
| `telegram_handler.py` | Telegram API wrapper |
| `bot_commands.py` | Telegram command dispatch |
| `influxdb_logger.py` | Motion → InfluxDB |
| `esp_devices.py` | HTTP fallback for ESP01-LL-RLY heartbeat |

### Pi5 — Python Venv

`picamera2` is **system-wide** (apt), not in venv. Venv must have `include-system-site-packages = true`.

```bash
# If venv is ever recreated:
python3 -m venv --system-site-packages /home/pi5/myenv
# NEVER use: python3 -m venv /home/pi5/myenv  ← breaks video recording
```

### Pi5 — Aliases

```bash
# Navigation
cdmain        # cd to RASPI5-MAIN
cdgit         # cd to /home/pi5/pi5_drive/Git_projects
cdapp         # cd to HOMESECURITY-APP

# mybot service
stopmybot / startmybot / restartmybot / statusmybot / servicemybot
copymybot     # copy mybot.service to /etc/systemd/system/
enablemybot / reloadmybot

# mqttdatainflux service
stopmqtt / startmqtt / restartmqtt / statusmqtt / servicemqtt

# All services together (handles VIP guard)
startall      # start keepalived + mybot + mqttdatainflux (checks VIP first)
stopall       # stop all services + release VIP
statusall     # show status of all three services
makepi4master # hand VIP to Pi4 (stops Pi5, starts Pi4 via SSH)
makepi5master # hand VIP to Pi5 (stops Pi4 via SSH, starts Pi5)

# Pi4 remote (via sshpass)
sshpi4        # SSH into Pi4 (192.168.1.122)
pi4startall   # run startall on Pi4 via SSH
pi4stopall    # run stopall on Pi4 via SSH
pi4statusall  # run statusall on Pi4 via SSH

# Misc
myenv         # source /home/pi5/myenv/bin/activate
sync          # run sync.sh backup script
backup        # run backup.sh
gitpullall    # pull all Git repos
copyalias     # copy .bash_aliases to repo backup
```

---

## ESP Devices

### ESP32-LP-RDR1 — Main Radar (Lower Porch Radar 1)

| Item | Value |
|------|-------|
| IP | `192.168.1.87` |
| Board | ESP32 Dev Module, 240MHz |
| Project | `/home/pi5/pi5_drive/Git_projects/ESP32-LP-RDR1/` |
| Telegram suffix | `2` — all commands end with `2` (e.g. `/radar_on2`) |
| Telegram bot token | `8292280421:...` |
| Telegram chat ID | `6825638285` |
| WebServerUI port | `81` |
| SwitchService port | `80` |

**GPIO pins:**

| GPIO | Device |
|------|--------|
| 19 | Radar sensor (input) |
| 18 | Radar LED (output) |
| 27 | Door sensor (input) |
| 13 | DHT11 temp/humidity |

**MQTT topics published:**

| Topic | Broker | Purpose |
|-------|--------|---------|
| `home/esp32/radar1/motion` | Pi4 VIP (192.168.1.100 → .122) | Motion ON/OFF — Pi4 light control |
| `home/esp32/radar1/availability` | Pi4 VIP | LWT online/offline |
| `home/pi5/lp-rdr/motion` | Pi5 direct (192.168.1.108) | Motion ON/OFF — Pi5 camera trigger |
| `home/esp32/radar2/motion` | Pi4 VIP | Legacy topic (may still be published) |
| DHT11 temp/humidity | Pi4 broker | Home network only |

**What it does:**
- Motion detection → turns on porch light (ESP01-UL-RLY) at night (18:00–06:00 IST)
- Sends Telegram alerts on motion (throttled 60 s)
- Acts as **Firebase gateway** for the whole system — polls Firebase every 2 s for lobby light state and serves it to ESP01-UL-RLY via HTTP on port 81
- Triggers camera ESP for photo/video (via HTTP)
- Publishes to AWS IoT Core (motion events + device shadow heartbeat every 2 min)
- OTA firmware update via `/ota2` Telegram command or Firebase Storage

**Pi-independent paths (survive Pi outage):**
- AWS IoT → Lambda → FCM push notification (app siren)
- Telegram direct alerts
- Firebase RTDB direct write (app bulb sync)

**OTA flash:**
```bash
cd /home/pi5/pi5_drive/Git_projects/ESP32-LP-RDR1
/home/pi5/.platformio/penv/bin/pio run -e esp32_lp_rdr1_ota --target upload
# Or via Telegram: /ota2
# USB environment: esp32_lp_rdr1_usb
```

**Telegram bot commands (suffix "2", chat ID `6825638285`):**

| Category | Commands |
|----------|---------|
| Status | `/start2` `/status2` `/info2` `/ping2` `/deviceid2` `/devices2` `/checkdevices2` |
| Radar | `/radar_on2` `/radar_off2` `/radar_status2` |
| WiFi | `/wifi2` `/wifi_current2` `/wifi_scan2` `/wifi_add2 SSID PASS` `/wifi_remove2 SSID` `/wifi_priority2 SSID N` `/wifi_connect2 SSID` `/wifi_clearall2` |
| Device IPs | `/setlightip2 IP` `/setcameraip2 IP` `/dellightip2` `/delcameraip2` |
| Maintenance | `/ota2` `/restart2` `/tgreset2` `/help2` |

**NVS config (persisted in `"appcfg"` namespace):**

| Key | Default | Purpose |
|-----|---------|---------|
| `radar` | true | Radar motion enabled |
| `porch_ip` | 192.168.1.111 | ESP01-UL-RLY IP |
| `camera_ip` | 192.168.1.83 | Camera ESP IP |
| `aws` | true | AWS IoT enabled |
| `tg` | true | Telegram enabled |

**Loop timers:**

| Timer | Interval | Purpose |
|-------|----------|---------|
| `pollFirebaseForLobbyState` | 2 s | Read app-commanded state |
| `checkESP01Health` | 30 s | Ping ESP01 + write heartbeat to Firebase |
| Radar debounce | 300 ms | Signal must be stable before state change |
| Motion light-off | 5 min | Turn off porch light after last motion |
| AWS shadow heartbeat | 120 s | Device status shadow update |

### ESP01-LL-RLY — Lower Lobby Light Relay

| Item | Value |
|------|-------|
| IP | `192.168.1.85` |
| Board | ESP-01 (ESP8266) |
| Project | `/home/pi5/pi5_drive/Git_projects/ESP01-LL-RLY/` |
| MQTT prefix | `home/esp01/lower-lobby/` |
| Relay pin | GPIO0 (active LOW) |

**MQTT topics:**

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `home/esp01/lower-lobby/cmd/relay` | IN | `ON` / `OFF` command |
| `home/esp01/lower-lobby/state` | OUT (retained) | Current relay state |
| `home/esp01/lower-lobby/availability` | OUT | `online` / `offline` LWT |

**HTTP API:** `GET /lighton`, `GET /lightoff`, `GET /status`
**MQTT client ID:** `esp01-lower-lobby-relay`

**Notes:**
- ESP8266 PubSubClient keepalive instability: broker drops at ~90 s; workaround in firmware (stop+delay before reconnect, WIFI_NONE_SLEEP, 10 s heartbeat)
- Pi4 has HTTP fallback (`/lighton`, `/lightoff`) for reliability
- Firmware auto-off: **DISABLED** (`LIGHT_AUTO_OFF_MS=0`) — Pi controls all timers

**OTA flash** (run from Pi4):
```bash
cd /home/pi/pi4_drive/Git_projects/ESP01-LL-RLY
~/.platformio/penv/bin/platformio run -e esp01_relay_ota --target upload
# OTA hostname: ESP01-LL-RLY  upload port: 192.168.1.85
# ⚠️ After USB flash: power cycle required before OTA works
```

### ESP01-UL-RLY — Upper Lobby Light Relay (Porch)

| Item | Value |
|------|-------|
| IP | `192.168.1.111` |
| Board | ESP-01 (ESP8266) |
| Project | `/home/pi5/pi5_drive/Git_projects/ESP01-UL-RLY/` |
| MQTT prefix | `home/esp01/upper-lobby/` |
| Relay pin | GPIO0 (active LOW) |

**MQTT topics:**

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `home/esp01/upper-lobby/cmd/relay` | IN | `ON` / `OFF` command |
| `home/esp01/upper-lobby/state` | OUT (retained) | Current relay state |
| `home/esp01/upper-lobby/availability` | OUT | `online` / `offline` LWT |

**HTTP endpoints (served on port 80):** `GET /lighton`, `GET /lightoff`, `GET /status`
**OTA hostname:** `esp01-relay`

**Notes:**
- Does NOT talk to Firebase directly — ESP32-LP-RDR1 proxies Firebase for it
- Polls ESP32-LP-RDR1 (`GET :81/relay/lobby/state`) every 1 s for commanded state
- Reports confirmed back (`GET :81/relay/lobby/confirmed?state=true`) → ESP32 writes to Firebase
- ESP32-LP-RDR1 sends motion-triggered relay commands direct via HTTP (up to 3 retries)
- 3-min firmware auto-off safety timer (safety net only — ESP32's 5-min timer manages this)

**App → relay flow (total ~3–4 s):**
```
App tap → Firebase lights/lobby/state (~200ms)
  └── ESP32-LP-RDR1 polls every 2s → caches s_lobbyCommandedState
        └── ESP01-UL-RLY polls :81/relay/lobby/state every 1s
              └── setRelay() → reportConfirmedToESP32()
                    └── ESP32 writes lights/lobby/{state,confirmed} → App bulb
```

**Motion → relay (total ~300 ms on LAN):**
```
Radar HIGH → ESP32 isPorchLightTime()? → GET http://192.168.1.111/lighton (3 retries)
  → relay clicks → ESP32 pushRelayStateToFirebase(true)
```

### ESP32-LP-RLY — Lower Porch Light Relay

| Item | Value |
|------|-------|
| IP | `192.168.1.89` |
| Board | ESP32 |
| Project | `/home/pi5/pi5_drive/Git_projects/ESP32-LP-RLY/` |
| MQTT prefix | `home/switches/L-Porch-Light/` |
| HTTP port | `1089` |
| Firebase name | `L-Porch-Light` |

**MQTT topics:**

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `home/switches/L-Porch-Light/cmd` | IN (both brokers) | `ON` / `OFF` |
| `home/switches/L-Porch-Light/state` | OUT (VIP broker) | Current state |
| `home/switches/L-Porch-Light/availability` | OUT (VIP broker) | `online`/`offline` LWT |
| `home/switches/L-Porch-Light/state/json` | OUT (VIP broker) | Full JSON status |
| `home/esp32/lp-rly/ota/cmd` | IN | Firebase OTA trigger |

**Dual MQTT client architecture:**

| Client | Broker | Purpose |
|--------|--------|---------|
| Primary (`mqtt`) | `192.168.1.100` → Pi4 | State publish + general commands |
| Secondary (`mqttPi5`) | `192.168.1.108` Pi5 direct | Pi5 commands when Pi4 holds VIP |

Both subscribe to the cmd topic. Pi5's bridge also forwards the cmd topic out to Pi4.

**HTTP endpoints:** `GET /lighton`, `GET /lightoff`, `GET /status`

**OTA flash:**
```bash
cd /home/pi5/pi5_drive/Git_projects/ESP32-LP-RLY
/home/pi5/.platformio/penv/bin/pio run --target upload
# OTA hostname: ESP32-LP-RLY  port: 3232  password: otaesp32
# Firmware version 1.0.0 — increment before OTA push (same version = skipped)
```

### Camera ESP — Motion Photo Device

| Item | Value |
|------|-------|
| Default IP | `192.168.1.83` |
| Project | `/home/pi5/pi5_drive/Git_projects/ESP32-S3-XIAO-CAMERA/` |
| Triggered by | ESP32-LP-RDR1 via `GET http://192.168.1.83/radar_trigger` (throttled 60 s) |
| Purpose | Take photo on motion → send to Telegram |
| NVS key on radar | `camera_ip` (default `192.168.1.83`, changeable via `/setcameraip2 IP`) |

---

## MQTT Topic Map (Complete)

Broker: Pi4 at `192.168.1.122:1883` — credentials `mq / mq`

```
home/esp01/lower-lobby/cmd/relay     ← Pi publishes (ON/OFF) → ESP01-LL-RLY
home/esp01/lower-lobby/state         → ESP01-LL-RLY publishes (retained)
home/esp01/lower-lobby/availability  → ESP01-LL-RLY LWT

home/esp01/upper-lobby/cmd/relay     ← Pi publishes (ON/OFF) → ESP01-UL-RLY
home/esp01/upper-lobby/state         → ESP01-UL-RLY publishes (retained)
home/esp01/upper-lobby/availability  → ESP01-UL-RLY LWT

home/switches/L-Porch-Light/cmd      ← Pi publishes (ON/OFF) → ESP32-LP-RLY
home/switches/L-Porch-Light/state    → ESP32-LP-RLY publishes (retained)
home/switches/L-Porch-Light/availability → ESP32-LP-RLY LWT
home/switches/L-Porch-Light/state/json  → ESP32-LP-RLY JSON status
home/esp32/lp-rly/ota/cmd           ← OTA trigger
home/esp32/lp-rly/ota/status        → OTA progress → Pi → Telegram
home/esp32/lp-rly/telegram          → ESP32-LP-RLY → Pi → Telegram forward

home/esp32/radar1/motion             → ESP32-LP-RDR1 (ON/OFF) → Pi4 light ctrl
home/esp32/radar1/availability       → ESP32-LP-RDR1 LWT
home/pi5/lp-rdr/motion              → ESP32-LP-RDR1 direct to Pi5 (video trigger)
home/esp32/radar2/motion             → legacy (may still be published)

home/pi5/lp-rdr/motion is on Pi5's broker directly (ESP32-LP-RDR1 dual client)
home/esp32/radar1/# bridges Pi4 → Pi5 via mosquitto bridge
```

---

## Firebase RTDB

URL: `https://home-security-app-555cf-default-rtdb.asia-southeast1.firebasedatabase.app`

### Paths written by Pi

| Path | Writer | Content |
|------|--------|---------|
| `/devices/RASPI-4/` | Pi heartbeat (60 s) | cpuTemp, uptime, motionDetected, recording |
| `/devices/ESP01-LL-RLY/` | Pi | lower lobby relay state |
| `/devices/esp01_relay/` | Pi | upper lobby relay state |
| `/devices/ESP32-LP-RLY/` | Pi | LP porch relay state |
| `/lights/living_room/confirmed` | Pi | after lower lobby MQTT state |
| `/lights/living_room/state` | Pi | current lower lobby state |
| `/lights/lobby/confirmed` | Pi | after upper lobby MQTT state |
| `/lights/lobby/state` | Pi | current upper lobby state |
| `/lights/lower_porch_light/confirmed` | Pi | after LP relay MQTT state |
| `/lights/lower_porch_light/state` | Pi | current LP relay state |

### Paths written by ESP32-LP-RDR1 (Pi-independent)

| Path | Content |
|------|---------|
| `/lights/lobby/` | state + confirmed (direct write when Pi is down) |
| AWS shadow: `radars/{radar2}` | heartbeat every 2 min |

### SSE streams read by Pi (command input)

| Path | Action |
|------|--------|
| `/lights/living_room/state` | → lower lobby relay ON/OFF |
| `/lights/lobby/state` | → upper lobby relay ON/OFF |
| `/lights/lower_porch_light/state` | → LP relay ON/OFF |
| `/schedules/` | → APScheduler light timers |

### Firestore (separate from RTDB)

| Collection | Writer | Purpose |
|------------|--------|---------|
| `door_events` | Pi (`push_door_event`) | Door open/close history for app |

---

## Data Flow — How Everything Connects

### App → Light ON (normal path)

```
Flutter app taps light switch
  ├── Firebase RTDB /lights/{id}/state = true
  │     └── Pi SSE stream fires → MQTT cmd → ESP relay
  └── [WiFi only] HTTP PUT Pi:5757/lights/{id}  (LAN fallback)
        └── Pi local_api_server → MQTT cmd → ESP relay
  (Pi deduplicates if both arrive)
```

### Motion → Light ON (night) + Video

```
ESP32-LP-RDR1 radar detects motion
  ├── MQTT home/esp32/radar1/motion=ON → Pi4 broker
  │     ├── Pi4 mybot: night? → MQTT cmd → ESP01-LL-RLY or ESP01-UL-RLY ON
  │     └── bridges to Pi5 → Pi5 mybot: camera recording + Telegram video
  ├── MQTT home/pi5/lp-rdr/motion=ON → Pi5 direct
  │     └── Pi5 mybot: camera recording + Telegram video
  ├── Telegram direct (throttled 60 s) — Pi-independent
  └── AWS IoT → Lambda → FCM (app siren) — Pi-independent
```

### Motion → Light OFF (5-min timer)

```
ESP32-LP-RDR1 radar goes LOW for 5 min
  └── MQTT home/esp32/radar1/motion=OFF → Pi
        └── 5-min timer fires → MQTT cmd → relay OFF
```

### Pi Local Sensors → Light

```
LD2420 radar (/dev/serial0, Pi4 only)
  └── Pi _trigger_light_control (VIP master only)
        └── night? → MQTT → ESP01-LL-RLY ON + video recording (Pi4 only)

PIR (GPIO25) / MMS (GPIO27) → also feed motion logic
Reed switch (GPIO26) → Telegram door alert + AWS IoT + Firestore log
```

### VIP / MASTER guard logic (Pi5 code)

Pi5 checks `ip addr show wlan0` for `192.168.1.100` before sending:
- Relay commands (except radar motion — that's always sent by both Pis, idempotent)
- Firebase updates from relay state changes
- App command responses
- Scheduler-triggered commands

---

## What Survives a Pi Outage

| Feature | Pi needed? | Path |
|---------|-----------|------|
| App siren on motion | No | ESP32-LP-RDR1 → AWS IoT → Lambda → FCM |
| Telegram motion alert | No | ESP32-LP-RDR1 → Telegram API direct |
| AWS shadow heartbeat | No | ESP32-LP-RDR1 → AWS IoT |
| App lobby bulb sync | No | ESP32-LP-RDR1 → Firebase direct |
| Video recording | Yes | Pi5 camera required |
| App light switches | Yes | Pi required for Firebase SSE → MQTT |
| Scheduled light timers | Yes | APScheduler runs on Pi |
| DHT11 logging | Yes | Pi MQTT subscription required |
| Door alerts | Yes | Pi GPIO + Telegram/Firebase |

---

## HOMESECURITY-APP — Flutter App

**Path:** `/home/pi5/pi5_drive/Git_projects/HOMESECURITY-APP/`
**Package:** `com.homesecurity.alertapp`
**Platforms:** Android (primary) + Windows
**Min Android SDK:** 21, Target: 34

### App Features

| Module | Description |
|--------|-------------|
| Lights | Toggle relays; all-on/off; daily scheduler |
| Door Status | Live open/closed per door; Door1–Door4; rename/locate/enable |
| Radar / Motion | Live alerts from ESP32 via AWS IoT Shadow + FCM |
| DHT11 | Live temp & humidity; 24h/7d graph |
| Energy Meter | Live voltage, current, power; historical graph |
| Event Logs | 4-tab live log (All/Motion/Door/Other); Firestore-backed history |
| Sensors & Devices | All IoT device heartbeat status |
| OTA | Trigger firmware OTA on ESP32 from app |
| Bot Commands | Send commands to Pi Telegram bot |
| Lights Scheduler | Daily on/off timers synced to Pi via Firebase |
| Grocery List | Shared list (Firestore) |
| Electronics Inventory | Component inventory with AI import (Firestore) |

### App Architecture — Light Control

```
App tap → Firebase RTDB lights/{id}/state
              └── Pi SSE → MQTT → relay

App tap (Wi-Fi) → HTTP PUT Pi:5757/lights/{id}   (LAN fallback, ~50ms)
              └── Pi local_api_server → MQTT → relay

Speed: local ~50–100 ms vs Firebase ~500 ms–2 s
End-to-end app toggle: ~3–4 s total (Firebase path)
End-to-end motion: ~300 ms (direct LAN)
```

### Local Wi-Fi Fallback

- Pi local API port: `5757`
- Test: `http://192.168.1.100:5757/ping` → `{"ok": true}`
- Default Pi IP stored in SharedPreferences key `local_pi_ip` (default `192.168.1.100`)
- Configure in app: Lights page → VIP chip → enter IP → Save & Test
- Connectivity banner: Green=local, Amber=Firebase, Red=offline

### AWS IoT

**Endpoint:** `a3m8azs2x620qd-ats.iot.us-east-1.amazonaws.com:8883` (mTLS)
**Thing:** `all_sensors`

| Shadow | Purpose | Key fields |
|--------|---------|------------|
| `doors` | Reed switch states | `door1.state`, `door1.timestamp`, `door1.sensor_id` |
| `radars` | Radar motion levels | `radar1.state` (low/high), `radar1.timestamp` |
| `device_status` | Universal heartbeat (all devices) | `{device_id}.timestamp`, `.type`, `.uptime` |

**Flutter subscriptions per shadow:**
```
$aws/things/all_sensors/shadow/name/{shadow}/get/accepted
$aws/things/all_sensors/shadow/name/{shadow}/update/accepted
$aws/things/all_sensors/shadow/name/{shadow}/update/delta
```

**Device heartbeat:** every 120 s → AWS IoT Shadow `device_status`
**isOnline threshold:** lastSeen < 3 min ago

**AWS Certs (not committed):** place in `assets/certs/`:
- `AmazonRootCA1.pem`, `certificate.pem.crt`, `private.pem.key`

### Door IDs (hardware)

| ID | Maps to |
|----|---------|
| `Door1` | Reed switch 1 on Raspberry Pi (GPIO 26) |
| `Door2` | Reed switch 2 on Raspberry Pi |
| `Door3` | Reed switch 3 on Raspberry Pi |
| `Door4` | Reed switch 4 (disabled by default) |

**Firestore doc_id formula** (must match Pi + Lambda + App):
```
Door{N}_{state}_{epoch_seconds}
e.g. Door1_open_1779627685
```

### Door Event Paths (3 paths, deduped)

```
Pi reed switch → Path A: Firestore REST door_events/{doc_id}  (primary persistence)
              → Path B: AWS IoT Shadow → Flutter MQTT (UI only on Android)
              → Path C: Lambda → FCM push → App (Firestore + Event Log + notification)
```

### App Firestore Collections

| Collection | Writer | Retention |
|------------|--------|-----------|
| `door_events` | Pi + Lambda + App | 50 docs, 30 days |
| `motion_events` | App (high motion only) | 30 docs, 30 days |
| `sensor_history/dht11/readings` | DHT11 service | 30 days |
| `sensor_history/energy/readings` | Energy meter | 30 days |
| `devices` | Device registry | — |

### App Firebase RTDB Paths

| Path | Purpose |
|------|---------|
| `lights/` | state + confirmed per light |
| `devices/RASPI-4/` | Pi heartbeat |
| `devices/ESP01-LL-RLY/` | Lobby relay state |
| `devices/esp01_relay/` | Porch relay state |
| `devices/ESP32-LP-RLY/` | LP relay state |
| `schedules/` | Daily on/off timer settings per light |

### Pi Local API (port 5757) — Full Endpoints

Implemented in `RASPI5-MAIN/local_api_server.py`. Runs on both Pi4 and Pi5.

| Method | Path | Request | Response | Purpose |
|--------|------|---------|----------|---------|
| GET | `/ping` | — | `{"ok": true}` | Reachability probe |
| GET | `/identity` | — | `{"hostname": "pi5"}` | VIP-holder ID (Pi4 returns `"pi"`) |
| GET | `/devices` | — | Device states JSON | All device states (same fields as Firebase) |
| PUT | `/lights/{id}` | `{"state": true\|false}` | `{"ok": true}` | Toggle named light relay |

**Notes:**
- App queries `/identity` on VIP `192.168.1.100` to know which Pi is active
- Pi4 hostname: `pi`, Pi5 hostname: `pi5` → displayed as `PI4`/`PI5` in app banner
- 404 for any unknown path, 503 if `/devices` callback not wired

### Firestore Light Device Schema

Collection: `lights/{id}`

| Field | Type | Purpose |
|-------|------|---------|
| `label` | String | Display name (e.g. "L-Porch-Light") |
| `room` | String | Room label (e.g. "Porch") |
| `on` | bool | App-commanded state (Firestore) |
| `schedulable` | bool | Appears in scheduler if true |

**Default lights** (seeded once via `seedDefaultLights()`):

| id | label | room |
|----|-------|------|
| `lobby` | L-Porch-Light | Porch |
| `living_room` | Living Room | Living Room |
| `bedroom` | Bedroom | Bedroom |
| `kitchen` | Kitchen | Kitchen |
| `bathroom` | Bathroom | Bathroom |

**State priority in app** (highest to lowest):
1. `_optimistic[id]` — set on tap, instant UI update before network
2. RTDB `lights/{id}/state` — app-commanded, arrives ~200ms
3. RTDB `lights/{id}/confirmed` — device-confirmed after relay toggle
4. Firestore `lights/{id}/on` — ground truth for all viewers

**Toggle de-dupe:** `_pending` map drops a second call for same `id` while first is in-flight.

### RTDB Light Schedule Schema

Path: `schedules/{lightId}`

| Field | Type | Example |
|-------|------|---------|
| `enabled` | bool | true |
| `on_time` | String HH:MM | "18:00" |
| `off_time` | String HH:MM | "06:00" |

Only lights with `schedulable: true` in Firestore appear in the scheduler page.
Pi reads these via SSE on `schedules/` and APScheduler fires at the set times.

### App OTA System (Firebase)

- **Firestore:** `ota_firmware/{deviceId}` — fields: `version`, `downloadUrl`, `fileName`, `uploadedAt`, `notes`
- **Storage:** `firmware/{deviceId}/{fileName}` — raw `.bin` file
- **RTDB flash command:** `commands/{deviceId}/ota` → `{"action": "start_ota", "url": ..., "version": ...}`
- **RTDB flash status (ESP writes back):** `commands/{deviceId}/ota_status` → `{"status": "downloading|installing|complete|failed", "progress": 0-100}`
- App uploads `.bin` from PC, writes Firestore record, then sends RTDB flash command
- ESP32 polls Firestore for new version; if version != current, downloads and flashes

### App Build & Debug Commands

```bash
# Build and run
flutter pub get
flutter run

# Build debug APK
flutter build apk --debug

# Install to device
adb install build/app/outputs/flutter-apk/app-debug.apk
adb install .\build\app\outputs\flutter-apk\app-debug.apk  # Windows

# Uninstall
adb uninstall com.homesecurity.alertapp

# Logs
adb logcat -s flutter
```

---

## Git Repositories

### On Pi5 (`/home/pi5/pi5_drive/Git_projects/`)

| Repo | GitHub | Purpose |
|------|--------|---------|
| `RASPI5-MAIN` | `bs220975/RASPI5-MAIN` | Pi5 main bot |
| `ESP32-LP-RDR1` | — | Main radar firmware (ESP32) |
| `ESP01-LL-RLY` | `bs220975/ESP01-LL-RLY` | Lower lobby relay firmware (ESP8266) |
| `ESP01-UL-RLY` | — | Upper lobby relay firmware (ESP8266) |
| `ESP32-LP-RLY` | — | LP porch relay firmware (ESP32) |
| `HOMESECURITY-APP` | — | Flutter Android app |
| `RASPI4-MAIN` | `bs220975/RASPI4-MAIN` | Pi4 main bot (mirror/dev) |
| `ESP32-RADAR` | — | Old/legacy radar project |
| `ESP32-DHT11` | — | Standalone DHT11 project |
| `ESP32-S3-XIAO-CAMERA` | — | Camera ESP project |

### On Pi4 only (`/home/pi/pi4_drive/Git_projects/`)

| Repo | Purpose |
|------|---------|
| `RASPI4-MAIN` | Pi4 main bot (production) |
| `ESP01-LL-RLY` | Lower lobby relay |
| `ESP01-UL-RLY` | Upper lobby relay |
| `ESP32-LP-RDR1` | Main radar |
| `ESP32-LP-RLY` | LP porch relay |
| `ESP32-RADAR` | Legacy radar |
| `HOMESECURITY-APP` | Flutter app |

---

## Known Issues / Ongoing Work

| Issue | Status | Notes |
|-------|--------|-------|
| Reed switch false door alerts | Software fix applied | Debounce 2.0 s, cooldown 5 s in `sensors.py`; hardware RC filter (10 kΩ + 100 nF on GPIO 26) not yet soldered |
| LD2420 EMI on GPIO 26 | Root cause identified | Radar ~10 cm from Pi couples RF into reed switch wire; move radar ≥30 cm or add RC filter |
| ESP01 MQTT keepalive instability | Workaround in firmware | PubSubClient on ESP8266 drops at ~90 s; permanent fix = AsyncMqttClient (not yet done) |
| Keepalived DISABLED | By design | JioFiber IP source guard blocked VIP traffic; .100 kept as static alias on Pi4 wlan0 for app backward compat |
| Pi5 MQTT_HOST=localhost | By design | Pi5 mybot connects to its own broker; bridge from Pi4 delivers ESP events |
| LL-RLY spurious ON on Pi5 start | FIXED 2026-06-23 | 6 methods missing VIP guard — firmware guard now gated on `is_master`; commit 74f996b |
| ESP32-LP-RDR1 MQTT keepalive bug | FIXED 2026-06-16 | `setKeepAlive(60)` + `mqttClient.loop()` at start of loop; commit 107bfbb (needs OTA flash) |

---

## Cloud Services

| Service | Provider | Purpose |
|---------|----------|---------|
| Firebase RTDB | Google | App ↔ Pi command channel + device state |
| Firebase Firestore | Google | Door event history |
| Firebase Storage | Google | OTA firmware files |
| AWS IoT Core | Amazon | Motion events + device shadows |
| AWS Lambda | Amazon | IoT → FCM push notification pipeline |
| Telegram Bot API | Telegram | Alerts + remote control |
| InfluxDB (local) | Self-hosted Pi4 | Motion event time-series metrics |

---

## Network Details

| Item | Value |
|------|-------|
| Router/gateway | `192.168.1.254` |
| Pi4 IP | `192.168.1.122` (DHCP reserved) |
| Pi5 IP | `192.168.1.108` (DHCP reserved) |
| VIP alias | `192.168.1.100` static secondary on Pi4 wlan0 (not VRRP) |
| MQTT broker | `192.168.1.122:1883` (Pi4) credentials `mq / mq` |
| InfluxDB | `http://[::1]:8086` org: `pi4org` bucket: `pi4data` |
| Firebase RTDB | `home-security-app-555cf-default-rtdb.asia-southeast1.firebasedatabase.app` |
| Local API | `Pi4:5757` + `Pi5:5757` — Flutter app LAN fallback |

---

## Quick Reference Commands

```bash
# ── MQTT ─────────────────────────────────────────────────────────────
# Watch all MQTT traffic
mosquitto_sub -h localhost -p 1883 -u mq -P mq -t "home/#" -v

# Watch specific device
mosquitto_sub -h 192.168.1.122 -p 1883 -u mq -P mq -t "home/esp01/lower-lobby/#" -v
mosquitto_sub -h 192.168.1.122 -p 1883 -u mq -P mq -t "home/esp32/radar1/#" -v
mosquitto_sub -h 192.168.1.122 -p 1883 -u mq -P mq -t "home/switches/L-Porch-Light/#" -v

# Manually command relays
mosquitto_pub -h localhost -p 1883 -u mq -P mq -t "home/esp01/lower-lobby/cmd/relay" -m "ON"
mosquitto_pub -h localhost -p 1883 -u mq -P mq -t "home/esp01/upper-lobby/cmd/relay" -m "ON"
mosquitto_pub -h localhost -p 1883 -u mq -P mq -t "home/switches/L-Porch-Light/cmd" -m "ON"

# ── Pi5 Services ─────────────────────────────────────────────────────
sudo systemctl restart mybot.service
journalctl -u mybot.service -f          # live log (alias: servicemybot)
systemctl status mosquitto              # alias: statusmqtt

# Run bot manually (debug)
cd /home/pi5/pi5_drive/Git_projects/RASPI5-MAIN
source /home/pi5/myenv/bin/activate
python3 main.py

# ── Pi4 Commands (via SSH) ───────────────────────────────────────────
ssh pi@192.168.1.122
sudo systemctl restart mybot.service
journalctl -u mybot.service -f

# ── GPIO ─────────────────────────────────────────────────────────────
watch -n 0.5 "raspi-gpio get 26"       # reed switch live monitor
vcgencmd get_throttled                  # 0x0 = healthy
vcgencmd measure_volts core             # normal ~0.866V

# ── Camera / Video ───────────────────────────────────────────────────
ls -lh /home/pi5/raspi_camera_videos/

# ── Logs ─────────────────────────────────────────────────────────────
journalctl -u mybot.service --since "30 min ago" | grep -i door
journalctl -u mybot.service --since "1 hour ago" | grep -i "relay\|light\|motion"

# ── PlatformIO (ESP firmware) ────────────────────────────────────────
cd /home/pi5/pi5_drive/Git_projects/ESP32-LP-RLY
/home/pi5/.platformio/penv/bin/pio run --target upload

# ── Navigation ───────────────────────────────────────────────────────
# cdmain  → RASPI5-MAIN (Pi5) or RASPI4-MAIN (Pi4)
# cdgit   → Git_projects folder on either Pi
```

---

## Pi4 vs Pi5 Comparison

| Item | Pi4 (Primary Hub) | Pi5 (Camera Node) |
|------|-------------------|-------------------|
| Role | Main MQTT broker, all logic | Camera recording, BACKUP logic |
| IP | 192.168.1.122 | 192.168.1.108 |
| User | `pi` | `pi5` |
| Camera | No | Yes (picamera2, system-wide) |
| MQTT broker | Primary (ESP devices connect here) | Bridges from Pi4 |
| mybot MQTT_HOST | 192.168.1.122 | localhost |
| InfluxDB | Yes (main) | Yes (secondary) |
| LD2420 radar | Yes (GPIO 14/15) | Yes (GPIO 14/15) |
| Firebase key | `RASPI-4` | `RASPI-5` (but heartbeat still PATCHes `/devices/RASPI-4/`) |
| Drive folder | `pi4_drive` | `pi5_drive` |
| Keepalived | DISABLED | BACKUP role (was 100) — also disabled |
| VIP .100 | Static secondary alias on wlan0 | Not assigned |
