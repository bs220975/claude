---
name: feedback-push-claude
description: "When user says 'push claude', update CLAUDE.md and push to GitHub claude-brain repo without asking for confirmation"
metadata:
  type: feedback
---

When the user says **"push claude"**, immediately and without asking:
1. Pull latest first: `git -C /home/pi/claude-brain pull --ff-only 2>/dev/null`
2. Copy memory files: `cp /home/pi/.claude/projects/-home-pi-pi4-drive-Git-projects-RASPI4-MAIN/memory/*.md /home/pi/claude-brain/memory/`
3. `git -C /home/pi/claude-brain add .`
4. `git -C /home/pi/claude-brain commit -m "Update brain — <brief summary of what changed>"`
5. `git -C /home/pi/claude-brain push origin main`

**Why:** User wants a single short command to sync their system brain to GitHub so all devices get the update. No confirmation needed — Bash(*) is already in global settings so no permission prompts.

**How to apply:** The moment "push claude" appears in the user's message, execute all steps above in sequence. Do not ask "are you sure" or "should I push". Just do it.
