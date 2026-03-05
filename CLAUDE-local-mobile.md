# CLAUDE.md - Local Claude Mobile Access

## Table of Contents

1.  [Executive Summary](#executive-summary)
2.  [Quick Start Guide](#quick-start-guide)
3.  [Project Context](#project-context)
4.  [Architecture](#architecture)
5.  [Setup Guide](#setup-guide)
6.  [Critical Safety Rules](#critical-safety-rules)
7.  [Development Environment](#development-environment)
8.  [Development Workflows](#development-workflows)
9.  [Context Management & Short Codes](#context-management--short-codes)
10. [Technical Reference](#technical-reference)
11. [Troubleshooting](#troubleshooting)

## Executive Summary

This project enables **mobile access to Claude Code running locally on a Mac** via secure tunneling. The goal is to use your iPhone/Android as a remote terminal to interact with Claude Code sessions on your Mac — no cloud hosting required.

### How It Works
```
┌──────────────┐     Tailscale/WireGuard     ┌──────────────────┐
│   Mobile      │ ◄──── secure tunnel ──────► │   Mac (Local)     │
│   (Browser/   │                             │                   │
│    SSH App)   │                             │  Claude Code CLI  │
│               │                             │  + tmux sessions  │
│               │                             │  + ttyd web term  │
└──────────────┘                             └──────────────────┘
```

### Key Features
-   Access Claude Code from mobile anywhere on the same network (or remotely via tunnel)
-   Persistent tmux sessions survive disconnections
-   Web-based terminal (ttyd) for browser access — no SSH app needed
-   Secure by default — Tailscale mesh VPN or SSH key auth

## Quick Start Guide

### Prerequisites (Mac)
```bash
# Core tools
brew install tmux           # Session persistence
brew install ttyd           # Web-based terminal
brew install tailscale      # Secure tunnel (recommended)
# OR
brew install cloudflared    # Alternative: Cloudflare Tunnel
# OR
brew install ngrok          # Alternative: ngrok

# Claude Code
npm install -g @anthropic-ai/claude-code
# Verify
claude --version

# Optional but recommended
brew install mosh           # Mobile-friendly SSH (handles network switches)
```

### Prerequisites (Mobile)
-   **Option A (Recommended)**: Any mobile browser (Safari/Chrome) — for ttyd web terminal
-   **Option B**: SSH client app — Termius, Blink Shell (iOS), or JuiceSSH (Android)
-   **Option C**: Tailscale app — install from App Store / Play Store

### 30-Second Quick Start
```bash
# On your Mac — run this single command:
tmux new-session -s claude -d 'claude' && ttyd -p 7681 tmux attach -t claude

# Then open on your mobile browser:
# http://<your-mac-ip>:7681
```

## Project Context

### Project Overview
A lightweight infrastructure project to make Claude Code accessible from mobile devices by combining:
1.  **tmux** — persistent terminal sessions
2.  **ttyd** — web terminal server
3.  **Tailscale/WireGuard** — secure remote access tunnel
4.  **Shell scripts** — automation for setup, startup, and management

### Goals
-   Zero-friction mobile access to local Claude Code
-   No cloud dependency — everything runs on your Mac
-   Survives network changes (WiFi → cellular → WiFi)
-   Minimal resource usage
-   Secure by default

## Architecture

### Components

| Component | Purpose | Port |
|-----------|---------|------|
| **Claude Code CLI** | The AI assistant | N/A (runs in terminal) |
| **tmux** | Session persistence & multiplexing | N/A (local) |
| **ttyd** | Web-based terminal server | `7681` |
| **Tailscale** | Secure mesh VPN tunnel | `41641` (UDP) |
| **mosh** (optional) | Mobile-optimized SSH | `60000-61000` (UDP) |

### Access Methods (Choose One or More)

#### Method 1: ttyd + Local Network (Simplest)
```
Mobile Browser → http://192.168.x.x:7681 → tmux → Claude Code
```
- Best for: Same WiFi network usage
- Pros: Zero setup on mobile, works in any browser
- Cons: Only works on local network

#### Method 2: ttyd + Tailscale (Recommended)
```
Mobile Browser → http://100.x.x.x:7681 → tmux → Claude Code
              (Tailscale VPN)
```
- Best for: Access from anywhere
- Pros: Encrypted, works across networks, stable IPs
- Cons: Requires Tailscale app on both devices

#### Method 3: SSH + Tailscale
```
Mobile SSH App → ssh user@100.x.x.x → tmux attach → Claude Code
              (Tailscale VPN)
```
- Best for: Power users who prefer native SSH
- Pros: Most secure, native terminal experience
- Cons: Requires SSH app on mobile

#### Method 4: mosh + Tailscale (Best for Unstable Networks)
```
Mobile (Blink/Termius) → mosh user@100.x.x.x → tmux attach → Claude Code
                       (Tailscale VPN)
```
- Best for: Moving between WiFi and cellular
- Pros: Handles network roaming, instant reconnection
- Cons: Requires mosh-compatible app

## Setup Guide

### Phase 1: Mac Setup — tmux + Claude Code

#### 1.1 Configure tmux for Mobile
Create `~/.tmux-mobile.conf`:
```bash
# Mobile-optimized tmux config
set -g status-position top
set -g status-style 'bg=#1a1a2e fg=#e0e0e0'
set -g status-left '[Claude] '
set -g status-right '%H:%M'

# Larger scrollback for mobile review
set -g history-limit 50000

# Aggressive resize for varying mobile screen sizes
setw -g aggressive-resize on

# Mouse support (for touch interaction via ttyd)
set -g mouse on

# Reduce escape time (faster response on mobile)
set -s escape-time 0

# Increase display time for messages
set -g display-time 3000
```

#### 1.2 Create Startup Script
Create `~/bin/claude-mobile`:
```bash
#!/bin/bash
# claude-mobile - Start Claude Code accessible from mobile

SESSION_NAME="claude-mobile"
TTYD_PORT="${TTYD_PORT:-7681}"
TTYD_PID_FILE="/tmp/ttyd-claude.pid"

start() {
    echo "Starting Claude Mobile Access..."

    # Create tmux session with Claude Code
    if ! tmux has-session -t "$SESSION_NAME" 2>/dev/null; then
        tmux new-session -d -s "$SESSION_NAME" -x 80 -y 24
        tmux send-keys -t "$SESSION_NAME" "claude" Enter
        echo "✓ tmux session '$SESSION_NAME' created with Claude Code"
    else
        echo "→ tmux session '$SESSION_NAME' already exists"
    fi

    # Start ttyd web terminal
    if [ -f "$TTYD_PID_FILE" ] && kill -0 "$(cat "$TTYD_PID_FILE")" 2>/dev/null; then
        echo "→ ttyd already running on port $TTYD_PORT"
    else
        ttyd \
            --port "$TTYD_PORT" \
            --writable \
            --max-clients 2 \
            --title-format "Claude Mobile" \
            tmux attach-session -t "$SESSION_NAME" &
        echo $! > "$TTYD_PID_FILE"
        echo "✓ ttyd started on port $TTYD_PORT"
    fi

    # Display access info
    echo ""
    echo "=== Access Claude from Mobile ==="
    LOCAL_IP=$(ipconfig getifaddr en0 2>/dev/null || echo "unknown")
    echo "Local network:  http://${LOCAL_IP}:${TTYD_PORT}"

    if command -v tailscale &>/dev/null; then
        TS_IP=$(tailscale ip -4 2>/dev/null || echo "not connected")
        echo "Tailscale:      http://${TS_IP}:${TTYD_PORT}"
    fi

    echo ""
    echo "SSH access:     ssh $(whoami)@${LOCAL_IP} -t 'tmux attach -t ${SESSION_NAME}'"
    echo "================================"
}

stop() {
    echo "Stopping Claude Mobile Access..."

    if [ -f "$TTYD_PID_FILE" ]; then
        kill "$(cat "$TTYD_PID_FILE")" 2>/dev/null
        rm -f "$TTYD_PID_FILE"
        echo "✓ ttyd stopped"
    fi

    # Don't kill tmux session — preserve Claude conversation
    echo "→ tmux session preserved (use 'tmux kill-session -t $SESSION_NAME' to remove)"
}

status() {
    echo "=== Claude Mobile Status ==="
    if tmux has-session -t "$SESSION_NAME" 2>/dev/null; then
        echo "tmux session: RUNNING"
    else
        echo "tmux session: STOPPED"
    fi

    if [ -f "$TTYD_PID_FILE" ] && kill -0 "$(cat "$TTYD_PID_FILE")" 2>/dev/null; then
        echo "ttyd (web):   RUNNING on port $TTYD_PORT"
    else
        echo "ttyd (web):   STOPPED"
    fi

    if command -v tailscale &>/dev/null; then
        TS_STATUS=$(tailscale status --self 2>/dev/null | head -1)
        echo "Tailscale:    $TS_STATUS"
    fi
}

case "${1:-start}" in
    start)  start ;;
    stop)   stop ;;
    status) status ;;
    restart) stop; sleep 1; start ;;
    *)      echo "Usage: claude-mobile [start|stop|status|restart]" ;;
esac
```

Make executable:
```bash
mkdir -p ~/bin
chmod +x ~/bin/claude-mobile
```

### Phase 2: Tailscale Setup (Recommended for Remote Access)

```bash
# 1. Install Tailscale on Mac
brew install tailscale
# Start Tailscale and authenticate
sudo tailscale up

# 2. Install Tailscale on Mobile
# iOS: App Store → "Tailscale"
# Android: Play Store → "Tailscale"
# Sign in with same account

# 3. Verify connectivity
tailscale status
# Both devices should appear in the list

# 4. Test from mobile browser
# Open: http://<tailscale-ip>:7681
```

### Phase 3: Security Hardening

#### ttyd Authentication
```bash
# Add basic auth to ttyd
ttyd --credential user:password --port 7681 tmux attach -t claude-mobile

# OR use Tailscale Funnel with HTTPS (recommended)
tailscale funnel --bg 7681
```

#### SSH Key Setup (for SSH access method)
```bash
# On Mac — ensure SSH is enabled
sudo systemsetup -setremotelogin on

# On Mobile — generate key pair in your SSH app
# Copy public key to Mac:
# Add to ~/.ssh/authorized_keys

# Disable password auth (keys only)
# Edit /etc/ssh/sshd_config:
#   PasswordAuthentication no
#   PubkeyAuthentication yes
```

#### Firewall Rules
```bash
# Only allow ttyd on Tailscale interface
# macOS firewall: System Preferences → Security → Firewall → Options
# Block incoming on port 7681 for public interfaces
# Allow on Tailscale interface (utun*)
```

### Phase 4: Auto-Start on Boot (Optional)

Create `~/Library/LaunchAgents/com.claude.mobile.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.mobile</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>-c</string>
        <string>~/bin/claude-mobile start</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <false/>
</dict>
</plist>
```

Load it:
```bash
launchctl load ~/Library/LaunchAgents/com.claude.mobile.plist
```

## Critical Safety Rules

### Network Security
-   **NEVER expose ttyd to the public internet without authentication**
-   **NEVER use ngrok free tier for persistent access** (URLs are public and guessable)
-   **Always use Tailscale or SSH tunnel** for remote access
-   **Enable authentication** on ttyd if accessible beyond localhost

### Claude Code Safety
-   Same rules as main CLAUDE.md — no force flags, no destructive operations
-   **Be extra careful on mobile** — small screens make it easy to miss context
-   **Review commands before confirming** — touch input is error-prone

### Session Safety
-   **Never kill tmux sessions without checking** — active Claude conversations are lost
-   **Use `tmux detach` (Ctrl+b, d)** instead of closing the browser/app
-   **Set tmux lock** for unattended sessions: `tmux set lock-after-time 300`

## Development Environment

### Project Structure
```
local-claude-mobile/
├── CLAUDE.md                   # This file
├── bin/
│   ├── claude-mobile           # Main startup script
│   ├── health-check            # Connection health monitor
│   └── setup.sh                # One-time setup script
├── config/
│   ├── tmux-mobile.conf        # Mobile-optimized tmux config
│   ├── ttyd.conf               # ttyd configuration
│   └── launchd/                # macOS auto-start configs
├── docs/
│   ├── SETUP.md                # Detailed setup guide
│   ├── TROUBLESHOOTING.md      # Common issues
│   └── SECURITY.md             # Security considerations
└── scripts/
    ├── install-deps.sh         # Install all dependencies
    └── test-connection.sh      # Test mobile connectivity
```

### Environment Variables
```bash
# .env (optional — for customization)
TTYD_PORT=7681              # Web terminal port
CLAUDE_SESSION=claude-mobile # tmux session name
TTYD_AUTH=user:password      # ttyd basic auth (optional)
TAILSCALE_FUNNEL=false       # Enable Tailscale HTTPS funnel
```

### Development Ports
| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| ttyd | `7681` | HTTP/WS | Web terminal |
| SSH | `22` | TCP | SSH access |
| mosh | `60000-61000` | UDP | Mobile SSH |
| Tailscale | `41641` | UDP | VPN tunnel |

## Development Workflows

### Daily Usage

#### Starting a Session
```bash
# On Mac — start everything
claude-mobile start

# On Mobile — open browser
# Navigate to http://<tailscale-ip>:7681
# You're now in Claude Code!
```

#### Reconnecting After Disconnect
```bash
# ttyd (browser): Just refresh the page — tmux session persists
# SSH: ssh user@ip -t 'tmux attach -t claude-mobile'
# mosh: mosh user@ip -- tmux attach -t claude-mobile
```

#### Managing Multiple Sessions
```bash
# Create a second Claude session
tmux new-session -d -s claude-project2 'claude'

# Switch sessions in tmux
# Ctrl+b, s → select session
# Ctrl+b, ( → previous session
# Ctrl+b, ) → next session
```

### Testing Checklist
Before considering setup complete:
-   [ ] `claude-mobile start` runs without errors
-   [ ] Mac browser can access `http://localhost:7681`
-   [ ] Mobile browser can access via local IP
-   [ ] Mobile browser can access via Tailscale IP (if using)
-   [ ] Claude Code responds to prompts through web terminal
-   [ ] Session survives browser close and reopen
-   [ ] Session survives WiFi disconnect and reconnect
-   [ ] Touch/tap works correctly in ttyd
-   [ ] Copy/paste works from mobile

## Context Management & Short Codes

### Quick Reference
-   `ccc` - Create context issue and compact
-   `nnn` - Smart planning (auto-runs `ccc` if needed)
-   `gogogo` - Execute the most recent plan
-   `rrr` - Create session retrospective

### Mobile-Specific Tips
-   **Use `ccc` before switching networks** — saves context in case of disconnection
-   **Prefer short commands** — mobile typing is slow, use aliases
-   **Use tmux copy mode** — `Ctrl+b, [` then scroll with touch (in ttyd)

### Recommended Shell Aliases (add to `~/.zshrc`)
```bash
# Claude Mobile shortcuts
alias cm='claude-mobile'
alias cms='claude-mobile status'
alias cma='tmux attach -t claude-mobile'
alias cmn='tmux new-session -d -s claude-mobile "claude" && claude-mobile start'
```

## Technical Reference

### ttyd Options Reference
```bash
ttyd [options] <command>
  --port, -p          Port to listen on (default: 7681)
  --writable, -W      Allow write (input) to the terminal
  --max-clients, -m   Maximum number of clients (0 = unlimited)
  --credential, -c    Basic auth (user:password)
  --ssl               Enable SSL
  --ssl-cert           SSL certificate file
  --ssl-key            SSL key file
  --title-format       Browser tab title
```

### Tailscale Useful Commands
```bash
tailscale status           # Show connected devices
tailscale ip -4            # Show your Tailscale IPv4
tailscale ping <device>    # Test connectivity to another device
tailscale funnel 7681      # Expose port via HTTPS (public)
tailscale serve 7681       # Expose port within tailnet only
```

### tmux Quick Reference (Mobile-Friendly)
```
Ctrl+b, d     Detach (safe exit)
Ctrl+b, [     Copy/scroll mode (use touch in ttyd)
Ctrl+b, c     New window
Ctrl+b, n     Next window
Ctrl+b, p     Previous window
Ctrl+b, 0-9   Switch to window N
Ctrl+b, z     Zoom/unzoom pane (useful on small screens!)
```

## Troubleshooting

### Cannot Connect from Mobile

**On same WiFi but can't reach Mac:**
```bash
# Check Mac's local IP
ipconfig getifaddr en0

# Check if ttyd is running
lsof -i :7681

# Check macOS firewall
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --listapps
```

**Tailscale connected but can't reach:**
```bash
# Check both devices are on same tailnet
tailscale status

# Ping from Mac to Mobile
tailscale ping <mobile-device-name>

# Check if port is accessible
nc -zv <tailscale-ip> 7681
```

### ttyd Issues

**Blank screen in mobile browser:**
-   Try force-refreshing the page
-   Check if tmux session exists: `tmux ls`
-   Restart ttyd: `claude-mobile restart`

**Keyboard not working:**
-   Ensure `--writable` flag is set on ttyd
-   Try a different mobile browser
-   On iOS: tap the terminal area to bring up keyboard

**Slow/laggy input:**
-   Reduce tmux status bar updates
-   Use Tailscale direct connection instead of relay
-   Check: `tailscale netcheck` for connection quality

### tmux Session Issues

**Session lost after Mac sleep:**
```bash
# Prevent Mac from sleeping when on power
sudo pmset -c disablesleep 1

# Or use caffeine
brew install keepingyouawake  # Menu bar app
```

**Can't attach to session:**
```bash
# List all sessions
tmux ls

# Force detach other clients and attach
tmux attach -t claude-mobile -d
```

### Network Switching (WiFi → Cellular)

**Connection drops when switching:**
-   Use **mosh** instead of SSH for seamless roaming
-   ttyd in browser: just refresh after network switch
-   Tailscale handles network changes automatically — wait 5-10 seconds

---

## Appendices

### A. Glossary
-   **ttyd** — Terminal emulator that runs in the browser over WebSocket
-   **tmux** — Terminal multiplexer for persistent sessions
-   **Tailscale** — Zero-config mesh VPN built on WireGuard
-   **mosh** — Mobile Shell, SSH replacement optimized for mobile connectivity
-   **tailnet** — Your private Tailscale network

### B. Alternative Tunnel Options

| Tool | Free Tier | Auth | Best For |
|------|-----------|------|----------|
| **Tailscale** | 100 devices | Built-in | Daily use (recommended) |
| **Cloudflare Tunnel** | Unlimited | Cloudflare Access | Team access |
| **ngrok** | Limited | Basic auth | Quick testing only |
| **WireGuard** | Self-hosted | Key-based | Full control |
| **Zerotier** | 25 devices | Built-in | Alternative to Tailscale |

### C. Mobile App Recommendations

**iOS:**
-   Safari/Chrome — for ttyd web terminal
-   Blink Shell — best SSH/mosh client
-   Termius — good free SSH client
-   Tailscale — VPN client

**Android:**
-   Chrome — for ttyd web terminal
-   JuiceSSH — SSH client
-   Termux — full terminal with mosh support
-   Tailscale — VPN client

### D. Environment Checklist
-   [ ] Homebrew installed on Mac
-   [ ] tmux installed and working
-   [ ] ttyd installed and working
-   [ ] Claude Code CLI installed (`claude --version`)
-   [ ] Tailscale installed on Mac
-   [ ] Tailscale installed on Mobile
-   [ ] Both devices on same tailnet
-   [ ] `claude-mobile` script in `~/bin/` and executable
-   [ ] Test: access web terminal from mobile browser
-   [ ] Optional: mosh installed for roaming
-   [ ] Optional: LaunchAgent configured for auto-start

---

## Oracle/Shadow Philosophy

This project follows the Oracle/Shadow philosophy.

Core principles:
1. **Nothing is Deleted** - Append only, timestamps = truth
2. **Patterns Over Intentions** - Observe what happens
3. **External Brain, Not Command** - Mirror reality, don't decide

---

**Last Updated**: 2026-03-05
**Version**: 1.0.0
