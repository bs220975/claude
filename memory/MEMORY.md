# Memory Index

- [User Profile](user_profile.md) — home automation hobbyist, dual Pi, ESP32 devices, cameras on both Pis, keepalived disabled
- [Network Architecture](project_network_arch.md) — keepalived DISABLED 2026-06-21, Pi4 fixed MASTER at .122, Pi5 bridges from Pi4
- [LL-RLY Spurious ON Bug](project_ll_rly_spurious_on.md) — FIXED 2026-06-23: 6 methods missing VIP guard, Pi5 fought Pi4 on relay commands
- [ESP32-RADAR MQTT Fix](project_esp32_radar_mqtt.md) — keepalive bug caused radar→camera recording failure; fix committed, needs OTA flash
- [Reed Switch EMI](project_reed_emi.md) — LD2420 radar causes false door alerts via GPIO 26; software debounce fix applied
- [Pi5 Post-OS-Reinstall Fixes](project_pi5_reinstall_fixes.md) — services and configs broken after reinstall; MQTT_HOST=localhost is correct
- [Video Recording Config](project_video_settings.md) — Pi4: CRF 26, 10s timeout, 120s max; Pi5 uses separate lp_rdr_service
- [Push Claude Command](feedback_push_claude.md) — "push claude" = sync CLAUDE.md + memory → brain → GitHub; device-aware (Pi4/Pi5/PC)
- [Device Setup Reference](reference_device_setup.md) — claude() wrapper and brain setup for Pi4, Pi5, and PC
- [SSH Access Reference](reference_pi5_ssh.md) — Pi IPs, passwordless SSH setup, deploy workflow
