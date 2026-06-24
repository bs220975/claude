---
name: project-video-settings
description: "Video recording config tuned for smaller Telegram files — CRF 26 re-encode, 10s motion timeout, 120s max duration"
metadata: 
  node_type: memory
  type: project
  originSessionId: 62d22a9a-2d08-4acf-9796-4c6b0171346e
---

Committed 2026-05-14. Changes in RASPI4-MAIN:

- `config.py` `VideoConfig.bitrate` 2M → 1M
- `config.py` `VideoConfig.motion_timeout` 5 → 10 (wired to `_check_stop_recording`)
- `config.py` `VideoConfig.max_duration` kept at 120s (was briefly changed to 60s, reverted per user)
- `video_recorder.py` ffmpeg: `-c copy` → `-c:v libx264 -crf 26 -preset fast` for smaller MP4
- `main.py` `_check_stop_recording`: hardcoded `5` → `self.config.video.motion_timeout`
