---
name: reference-pi5-ssh
description: Pi5 SSH access credentials, IPs, and passwordless key auth setup
metadata:
  node_type: memory
  type: reference
  originSessionId: ff3f88c1-7010-4ad4-8e82-d3d69558991c
---

## Pi addresses
- Pi4: `192.168.1.122` user=`pi`
- Pi5: `192.168.1.108` user=`pi5` password=`22`

## Passwordless SSH (set up 2026-06-18)
Both directions are configured with SSH key auth — no password needed:
- Pi4 → Pi5: `ssh pi5@192.168.1.108` (Pi4's ed25519 key in Pi5 authorized_keys)
- Pi5 → Pi4: `ssh pi@192.168.1.122` (Pi5's ed25519 key in Pi4 authorized_keys)

To SSH from Claude Code session (running on Pi4): `ssh pi5@192.168.1.108 "<command>"`
No `-o StrictHostKeyChecking=no` or `sshpass` needed — keys are trusted.

## Project files
- Pi4 project: `/home/pi/pi4_drive/Git_projects/RASPI4-MAIN/`
- Pi5 project (local copy on Pi4 drive): `/home/pi/pi4_drive/Git_projects/RASPI5-MAIN/`
- Pi5 project (live on Pi5): `/home/pi5/pi5_drive/Git_projects/RASPI5-MAIN/`

## Deploy workflow
1. Edit files in `/home/pi/pi4_drive/Git_projects/RASPI5-MAIN/`
2. SCP to Pi5: `scp /home/pi/pi4_drive/Git_projects/RASPI5-MAIN/<file> pi5@192.168.1.108:/home/pi5/pi5_drive/Git_projects/RASPI5-MAIN/`
3. Restart: `ssh pi5@192.168.1.108 "sudo systemctl restart mybot.service"`
4. Commit on Pi5: `ssh pi5@192.168.1.108 "cd /home/pi5/pi5_drive/Git_projects/RASPI5-MAIN && git add <files> && git commit -m '...'"`
5. Push from Pi4 copy: `git -C /home/pi/pi4_drive/Git_projects/RASPI5-MAIN push origin main`

## Logs
- Pi4 logs: `/home/pi/pi4_drive/Git_projects/RASPI4-MAIN/logs/error_log.txt`
- Pi5 logs: `ssh pi5@192.168.1.108 "tail -40 /home/pi5/pi5_drive/Git_projects/RASPI5-MAIN/logs/error_log.txt"`

Related: [[project_mqtt_migration]]
