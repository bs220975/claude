---
name: project-reed-emi
description: LD2420 radar EMI causes false door open/close Telegram alerts via GPIO 26 reed switch
metadata: 
  node_type: memory
  type: project
  originSessionId: a7ab68dc-bdf8-4933-bc23-a2d79fd66f10
---

LD2420 24GHz radar (~10cm from Pi) causes EMI on GPIO 26 reed switch line, producing false "Door OPENED/CLOSED" Telegram alerts without any real door activity.

**Fix applied (sensors.py `_reed_poll_loop`):**
- `DEBOUNCE` increased from 0.3 s → 2.0 s (EMI bursts pass in <2 s; real door transitions always sustain >2 s)
- `MIN_INTERVAL = 5.0 s` cooldown added between consecutive events (kills false open+close pairs)
- Suppressed events now log a WARNING so interference can be monitored

**Why:** EMI from the radar couples into the GPIO wire acting as an antenna and holds the pin in the wrong state long enough to pass the old 300 ms debounce.

**How to apply:** If false events return, further increase DEBOUNCE or add hardware RC filter (see hardware section below).

**Hardware fix still needed (permanent solution):**
- Add 10 kΩ series resistor between reed switch and GPIO 26, plus 100 nF capacitor from GPIO 26 to GND
- This RC low-pass filter (~160 Hz cutoff) completely rejects RF interference
- Alternatively: move LD2420 ≥30 cm from Pi, or add ferrite bead on reed switch wire
