---
name: project-video-settings
description: "Pi4 video recording config — CRF 26 re-encode, 10s motion timeout, 120s max; Pi5 uses separate lp_rdr_service"
metadata:
  type: project
---

## Pi4 RASPI4-MAIN video config (applied 2026-05-14)

- `config.py` `VideoConfig.bitrate` 2M → 1M
- `config.py` `VideoConfig.motion_timeout` 5 → 10 s
- `config.py` `VideoConfig.max_duration` 120 s (kept)
- `video_recorder.py` ffmpeg: `-c copy` → `-c:v libx264 -crf 26 -preset fast` (smaller MP4)
- `main.py` `_check_stop_recording`: hardcoded `5` → `self.config.video.motion_timeout`

## Pi5 video (lp_rdr_service.py)

Pi5 does NOT use main.py for video. `lp_rdr_service.py` owns picamera2 exclusively as a separate process (`mybot-lp-rdr.service`). Its config is in `lp_rdr_service.py` directly (not shared config.py).

**Why separate processes:** picamera2 can only be owned by one process. Isolating it keeps porch recording running even if mybot.service restarts.
