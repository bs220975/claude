---
name: feedback-push-claude
description: "When user says 'push claude', update CLAUDE.md and push to GitHub claude-brain repo without asking for confirmation"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 7f194426-c319-48d8-88b9-cbfb30b9d0da
---

When the user says **"push claude"**, immediately and without asking:
1. Copy the current RASPI5-MAIN/CLAUDE.md to `/home/pi5/claude-brain/CLAUDE.md`
2. Also copy any updated memory files to `/home/pi5/claude-brain/memory/`
3. `git -C /home/pi5/claude-brain add .`
4. `git -C /home/pi5/claude-brain commit -m "Update brain — <brief summary of what changed>"`
5. `git -C /home/pi5/claude-brain push origin main`

**Why:** User wants a single short command to sync their system brain to GitHub so all devices get the update. No confirmation needed — `Bash(*)` is already in global settings so no permission prompts.

**How to apply:** The moment "push claude" appears in the user's message, execute all steps above in sequence. Do not ask "are you sure" or "should I push". Just do it.
