---
name: feedback-push-claude
description: "When user says 'push claude', update CLAUDE.md and push to GitHub claude-brain repo without asking for confirmation"
metadata:
  type: feedback
---

When the user says **"push claude"**, immediately and without asking:

**On Pi5:**
1. `cp /home/pi5/pi5_drive/Git_projects/RASPI5-MAIN/CLAUDE.md /home/pi5/claude-brain/CLAUDE.md`
2. `cp /home/pi5/.claude/projects/-home-pi5-pi5-drive-Git-projects-RASPI5-MAIN/memory/*.md /home/pi5/claude-brain/memory/`
3. `git -C /home/pi5/claude-brain pull --ff-only`
4. `git -C /home/pi5/claude-brain add .`
5. `git -C /home/pi5/claude-brain commit -m "Update brain — <summary>"`
6. `git -C /home/pi5/claude-brain push origin main`

**On Pi4:**
1. `git -C /home/pi/claude-brain pull --ff-only`
2. `cp /home/pi/.claude/projects/-home-pi-pi4-drive-Git-projects-RASPI4-MAIN/memory/*.md /home/pi/claude-brain/memory/`
3. `git -C /home/pi/claude-brain add .`
4. `git -C /home/pi/claude-brain commit -m "Update brain — <summary>"`
5. `git -C /home/pi/claude-brain push origin main`

**Why:** Single command syncs brain to GitHub so all devices get it on next login.
**How to apply:** No confirmation. Just execute immediately.
