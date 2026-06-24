---
name: project-ll-rly-spurious-on
description: Root cause analysis for lower-lobby (ESP01-LL-RLY) light turning ON spuriously when Pi5 mybot.service starts alongside Pi4
metadata: 
  node_type: memory
  type: project
  originSessionId: 6a8c3d1b-179d-47d9-8238-43046819e612
---

## Finding: Lower-lobby relay spuriously turns ON when Pi5 starts

**Why:** Two bugs in `main.py` cause Pi5 (BACKUP) to fight Pi4's relay decisions.

---

### Bug 1 — Firmware guard fires on BACKUP Pi (primary cause)

**File:** `main.py`, method `_on_ll_mqtt_state` (~line 724)

**Mechanism:**
1. Pi5 starts, seeds `_last_living_room_cmd` from Firebase (e.g. `False` = OFF)
2. Pi5 MQTT connects, subscribes → Mosquitto delivers **retained** `lower-lobby/state` topic
3. If retained state is `"ON"` (stale, from a previous session or ESP01 self-turned-on on reconnect), line 757 unconditionally sets:
   ```python
   self._last_living_room_cmd = relay_on   # → True
   ```
4. `_ll_motion_triggered` is still `False` (seeded from Firebase as OFF → not set to True)
5. Later, ESP01 firmware auto-off fires → publishes state `"OFF"`
6. Firmware guard condition at line 745:
   ```python
   elif not relay_on and not self._ll_motion_triggered and self._last_living_room_cmd:
   ```
   evaluates as `True AND True AND True` → Pi5 re-sends `send_ll_relay(True)` → **LIGHT TURNS ON**

**Root cause:** The firmware guard has no VIP (MASTER) check. BACKUP Pi5 should never re-send ON — that's Pi4's job. Pi4 already correctly gates this via `_ll_was_offline`.

**Fix:** Add `is_master` to the firmware guard condition, and consolidate the two VIP `subprocess.run` calls in the method into one at the top:

```python
def _on_ll_mqtt_state(self, payload: str) -> None:
    import json as _json
    relay_on = False
    try:
        data = _json.loads(payload)
        relay_on = bool(data.get('relay', False))
    except (ValueError, AttributeError):
        relay_on = payload.upper() == 'ON'

    # Check VIP once for both the firmware guard and Firebase write
    try:
        vip = subprocess.run(['ip', 'addr', 'show', 'wlan0'],
                             capture_output=True, text=True, timeout=2)
        is_master = '192.168.1.100' in vip.stdout
    except Exception:
        is_master = False

    if self._ll_was_offline:
        self._ll_was_offline = False
        logger.debug('LL first state after reconnect — firmware guard suppressed')
    elif not relay_on and not self._ll_motion_triggered and self._last_living_room_cmd and is_master:
        # Only MASTER re-sends: BACKUP fighting this would conflict with Pi4
        logger.info('LL firmware auto-off on manually-on light — re-sending ON')
        if self._light_cmd_timer is not None:
            self._light_cmd_timer.cancel()
            self._light_cmd_timer = None
        if self.mqtt_bridge:
            self.mqtt_bridge.send_ll_relay(True)
        return

    self._last_living_room_cmd = relay_on

    if is_master:
        def _update() -> None:
            if self.firebase:
                self.firebase.push_ll_relay_status(reachable=True, relay_on=relay_on)
                self.firebase.set_light_confirmed('living_room', relay_on)
                self.firebase.set_light_state('living_room', relay_on)
        threading.Thread(target=_update, daemon=True).start()
```

---

### Bug 2 — `_on_firebase_ul_cmd` missing VIP guard (secondary, affects upper lobby)

**File:** `main.py`, method `_on_firebase_ul_cmd` (~line 979)

Unlike `_on_firebase_light_cmd` (living_room / LL-RLY) which has a VIP guard, `_on_firebase_ul_cmd` (lobby / UL-RLY) has NO VIP check. Both Pi4 and Pi5 respond to app commands for the upper lobby simultaneously.

This does not cause the lower-lobby light issue but is a latent bug for the upper lobby.

---

### Design intent (NOT a bug)

`_do_radar_motion_on` intentionally has no VIP guard — Pi5 BACKUP is designed to control both LL-RLY and UL-RLY via ESP32-RADAR1 motion independently. Both Pis send ON/OFF for radar motion; it's idempotent and by design.

---

## Status: FULLY FIXED — committed 2026-06-23 (74f996b, RASPI5-MAIN/main.py)

Six methods were missing VIP guards (BACKUP Pi fought Pi4):

- `_on_ll_mqtt_state` — VIP check moved to top; `is_master` added to firmware guard (primary spurious-ON cause)
- `_on_ul_mqtt_state` — same firmware guard fix for upper lobby
- `_on_firebase_ul_cmd` — app commands gated on VIP (same pattern as `_on_firebase_light_cmd`)
- `_on_schedule_ll_relay` / `_on_schedule_ul_relay` — scheduler commands gated on VIP
- `_trigger_light_control` — camera motion gated on VIP

**Design intent preserved:** `_do_radar_motion_on` and its off-timers have NO VIP guard — Pi5 BACKUP controls both relays via ESP32-RADAR1 motion independently (idempotent, by design).
