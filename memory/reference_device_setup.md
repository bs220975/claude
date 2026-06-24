---
name: reference-device-setup
description: "Setup instructions for claude-brain on each device — Pi4, Pi5, PC — so Claude gets full context on launch from any device"
metadata:
  type: reference
---

## How It Works

On every `claude` launch:
1. Pull latest brain from GitHub
2. Copy brain `CLAUDE.md` → `~/.claude/CLAUDE.md` (global, always loaded by Claude Code)
3. Copy brain `memory/*.md` → device's project memory dir (loaded when in project dir)

On every "push claude":
1. Pull brain (get any updates from other devices)
2. Copy local project CLAUDE.md + memory → brain
3. Commit + push to GitHub

---

## Pi5 Setup (home: /home/pi5, project: RASPI5-MAIN)

Brain location: `/home/pi5/claude-brain/`
Memory path: `/home/pi5/.claude/projects/-home-pi5-pi5-drive-Git-projects-RASPI5-MAIN/memory/`

**`~/.bashrc` claude() wrapper:**
```bash
claude() {
    local BRAIN=/home/pi5/claude-brain
    local GLOBAL_MD=/home/pi5/.claude/CLAUDE.md
    local MEM_DIR=/home/pi5/.claude/projects/-home-pi5-pi5-drive-Git-projects-RASPI5-MAIN/memory
    git -C "$BRAIN" pull --ff-only --quiet 2>/dev/null
    cp "$BRAIN/CLAUDE.md" "$GLOBAL_MD" 2>/dev/null
    mkdir -p "$MEM_DIR"
    cp "$BRAIN/memory/"*.md "$MEM_DIR/" 2>/dev/null
    /home/pi5/.npm-global/bin/claude "$@"
}
```

---

## Pi4 Setup (home: /home/pi, project: RASPI4-MAIN)

Brain location: `/home/pi/claude-brain/`
Memory path: `/home/pi/.claude/projects/-home-pi-pi4-drive-Git-projects-RASPI4-MAIN/memory/`

**`~/.bashrc` claude() wrapper:**
```bash
claude() {
    local BRAIN=/home/pi/claude-brain
    local GLOBAL_MD=/home/pi/.claude/CLAUDE.md
    local MEM_DIR=/home/pi/.claude/projects/-home-pi-pi4-drive-Git-projects-RASPI4-MAIN/memory
    git -C "$BRAIN" pull --ff-only --quiet 2>/dev/null
    cp "$BRAIN/CLAUDE.md" "$GLOBAL_MD" 2>/dev/null
    mkdir -p "$MEM_DIR"
    cp "$BRAIN/memory/"*.md "$MEM_DIR/" 2>/dev/null
    /home/pi/.npm-global/bin/claude "$@"
}
```

**First-time Pi4 setup** (clone brain if not already there):
```bash
git clone https://github.com/bs220975/claude-brain /home/pi/claude-brain
```

---

## PC Setup (Linux/Mac, home: ~/  or  /home/<user>/)

Brain location: `~/claude-brain/`
Memory path: computed from current working dir at launch

**One-time setup:**
```bash
git clone https://github.com/bs220975/claude-brain ~/claude-brain
mkdir -p ~/.claude
```

**`~/.bashrc` or `~/.zshrc` claude() wrapper:**
```bash
claude() {
    local BRAIN=~/claude-brain
    local GLOBAL_MD=~/.claude/CLAUDE.md
    local CWD_ENCODED
    CWD_ENCODED=$(pwd | sed 's|/|-|g')
    local MEM_DIR=~/.claude/projects/${CWD_ENCODED}/memory
    git -C "$BRAIN" pull --ff-only --quiet 2>/dev/null
    mkdir -p ~/.claude
    cp "$BRAIN/CLAUDE.md" "$GLOBAL_MD" 2>/dev/null
    mkdir -p "$MEM_DIR"
    cp "$BRAIN/memory/"*.md "$MEM_DIR/" 2>/dev/null
    # Adjust path to wherever claude is installed:
    ~/.npm-global/bin/claude "$@"
}
```

After adding to `.bashrc`: `source ~/.bashrc`

---

## GitHub Repo

`https://github.com/bs220975/claude-brain` — private repo, all devices use same remote.

Related: [[feedback-push-claude]]
