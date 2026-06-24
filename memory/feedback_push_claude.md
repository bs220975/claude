---
name: feedback-push-claude
description: "When user says 'push claude', sync CLAUDE.md + memory to GitHub claude-brain repo from whichever device is active — no confirmation needed"
metadata:
  type: feedback
---

When the user says **"push claude"**, immediately and without asking, detect the active device by home directory and run the matching sequence:

**Why:** User wants a single short command to sync their brain to GitHub so any device pulls fresh context on next launch. No confirmation needed.

---

## Device: Pi5 (working dir starts with /home/pi5)

```bash
BRAIN=/home/pi5/claude-brain
PROJECT_MD=/home/pi5/pi5_drive/Git_projects/RASPI5-MAIN/CLAUDE.md
MEM_SRC=/home/pi5/.claude/projects/-home-pi5-pi5-drive-Git-projects-RASPI5-MAIN/memory

git -C $BRAIN pull --ff-only --quiet 2>/dev/null
cp $PROJECT_MD $BRAIN/CLAUDE.md
cp $MEM_SRC/*.md $BRAIN/memory/ 2>/dev/null
git -C $BRAIN add .
git -C $BRAIN commit -m "Update brain — Pi5: <brief summary>"
git -C $BRAIN push origin main
```

## Device: Pi4 (working dir starts with /home/pi/)

```bash
BRAIN=/home/pi/claude-brain
PROJECT_MD=/home/pi/pi4_drive/Git_projects/RASPI4-MAIN/CLAUDE.md
MEM_SRC=/home/pi/.claude/projects/-home-pi-pi4-drive-Git-projects-RASPI4-MAIN/memory

git -C $BRAIN pull --ff-only --quiet 2>/dev/null
cp $PROJECT_MD $BRAIN/CLAUDE.md
cp $MEM_SRC/*.md $BRAIN/memory/ 2>/dev/null
git -C $BRAIN add .
git -C $BRAIN commit -m "Update brain — Pi4: <brief summary>"
git -C $BRAIN push origin main
```

## Device: PC (working dir is neither above)

```bash
BRAIN=~/claude-brain
git -C $BRAIN pull --ff-only --quiet 2>/dev/null
cp ~/.claude/CLAUDE.md $BRAIN/CLAUDE.md
CWD_ENCODED=$(pwd | sed 's|/|-|g')
cp ~/.claude/projects/${CWD_ENCODED}/memory/*.md $BRAIN/memory/ 2>/dev/null
git -C $BRAIN add .
git -C $BRAIN commit -m "Update brain — PC: <brief summary>"
git -C $BRAIN push origin main
```

**How to apply:** Check the working directory at the start of the conversation (`/home/pi5/` → Pi5, `/home/pi/` → Pi4, else PC). Execute the matching block. No confirmation prompt.
