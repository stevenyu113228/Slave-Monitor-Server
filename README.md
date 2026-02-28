# Slave Monitor Server

> *Your slaves don't sleep. Neither should your access to them.*

You've got AI agents, dev environments, and long-running tasks grinding away on remote machines. But you're not always at your desk. Maybe you're on the couch. Maybe you're in bed. Maybe you're on the toilet. Doesn't matter — your slaves should still be reachable.

Slave Monitor Server turns any Mac or Linux machine into a remotely controllable terminal server, accessible from your phone (or anywhere) over [Tailscale](https://tailscale.com) VPN. It wraps your terminal sessions in a mobile-friendly web UI with dictation, quick keys, and auto-reconnect. Pair it with the [Slave Monitor iOS app](https://github.com/stevenyu113228/Slave-Monitor-iOS) for a native experience, or just use any browser.

![Architecture](architecture.svg)

<p align="center">
  <img src="screenshot-mobile.jpg" alt="Terminal running on iPhone" width="300">
  <br>
  <em>Monitoring a slave from the couch</em>
</p>

## How It Works

```
Your Phone (Boss)                    Slave Machine (Mac/Linux)
┌─────────────────┐                ┌────────────────────────────┐
│  Browser / App   │  Tailscale    │  tmux (sessions persist)   │
│                 │◄──WireGuard──►│  ├── Claude Code            │
│  Voice dictation │   encrypted   │  ├── dev server             │
│  Quick keys      │               │  ├── build pipeline         │
│  Photo upload    │               │  └── whatever you want      │
└─────────────────┘                │                              │
   :8080 Web UI                    │  ttyd (:7681) → terminal    │
   :7681 Raw terminal              │  FastAPI (:8080) → Web UI   │
                                   └────────────────────────────┘
```

**[Tailscale](https://tailscale.com)** creates an encrypted WireGuard tunnel between your devices. Zero config — no port forwarding, no dynamic DNS, no firewall rules.

**[ttyd](https://github.com/tsl0922/ttyd)** serves your terminal as a web page via WebSocket. Full interactive shell in a browser.

**[tmux](https://github.com/tmux/tmux)** keeps sessions alive across disconnects. Close your browser, lose your connection — reconnect later and pick up exactly where you left off.

**Voice Wrapper** is a FastAPI app that wraps ttyd in a mobile-friendly UI with dictation support, quick-action buttons, photo upload, and auto-reconnect.

Everything binds exclusively to your Tailscale IP — **nothing is exposed to the public internet**.

## Cost

Free. Every tool is open-source or has a free tier:

| Tool | Cost |
|------|------|
| [Tailscale](https://tailscale.com/pricing) | Free for personal use (up to 3 users, 100 devices) |
| [ttyd](https://github.com/tsl0922/ttyd) | Free and open-source |
| [tmux](https://github.com/tmux/tmux) | Free and open-source |
| [FastAPI](https://fastapi.tiangolo.com) + [Uvicorn](https://www.uvicorn.org) | Free and open-source |

The remote access layer adds zero cost. You only pay for whatever's running inside the terminal (Claude API, etc).

## iOS App

For a native experience, use the companion app:

**[Slave Monitor iOS](https://github.com/stevenyu113228/Slave-Monitor-iOS)** — Native terminal emulator with multi-device management, tmux tab control, quick keys, and auto-reconnect. Sideload via AltStore (no Apple Developer account needed).

## Prerequisites

- **macOS** (Apple Silicon or Intel) or **Linux** (x86_64 / arm64)
- [Tailscale](https://tailscale.com) on your machine and phone (free plan works)
- Python 3.9+

## Quick Install

### Linux (Debian/Ubuntu/Kali)

```bash
./scripts/install-linux.sh
```

Installs ttyd (from GitHub releases), tmux, Python packages, and Tailscale.

### macOS

```bash
brew install ttyd tmux
pip3 install fastapi uvicorn python-multipart
```

## Start Your Slave

```bash
./scripts/start-remote-cli.sh
```

Output:

```
Tailscale IP: 100.x.y.z
ttyd running (PID: 12346) on http://100.x.y.z:7681
voice wrapper running (PID: 12347) on http://100.x.y.z:8080

=== Remote CLI Ready ===
Terminal:  http://100.x.y.z:7681
Voice UI:  http://100.x.y.z:8080
```

Open the Voice UI URL on your phone (with Tailscale active). Your slave is now under surveillance.

To release the slave (temporarily):

```bash
./scripts/stop-remote-cli.sh
```

## Auto-Start on Boot

Because good slaves start working before you even wake up.

<details>
<summary>macOS (launchd)</summary>

1. Edit `scripts/remote-cli.plist` — replace `YOUR_USERNAME` with your username
2. Copy scripts (launchd can't access `~/Documents/` due to TCC):

```bash
mkdir -p ~/.local/bin/remote-cli
cp scripts/* ~/.local/bin/remote-cli/
chmod +x ~/.local/bin/remote-cli/*.sh
```

3. Install:

```bash
cp scripts/remote-cli.plist ~/Library/LaunchAgents/com.user.remote-cli.plist
launchctl load ~/Library/LaunchAgents/com.user.remote-cli.plist
```
</details>

<details>
<summary>Linux (systemd)</summary>

1. Edit `scripts/remote-cli.service` — replace `YOUR_USERNAME` with your username
2. Copy scripts:

```bash
mkdir -p ~/.local/bin/remote-cli
cp scripts/* ~/.local/bin/remote-cli/
chmod +x ~/.local/bin/remote-cli/*.sh
```

3. Install and enable:

```bash
mkdir -p ~/.config/systemd/user
cp scripts/remote-cli.service ~/.config/systemd/user/remote-cli.service
systemctl --user daemon-reload
systemctl --user enable --now remote-cli
```
</details>

## Usage Tips

| Action | How |
|--------|-----|
| **Voice dictation** | Use the text input in the Web UI — iOS dictation works natively there |
| **Quick keys** | Tap buttons for arrow keys, Tab, Esc, Ctrl+C, Enter — keys that suck on a phone keyboard |
| **Mirror to desktop** | Run `tmux attach -t claude` on any terminal to see the same session |
| **Resume session** | Hit **Resume** in the Web UI to reconnect to a previous conversation |
| **Copy output** | Hit **Copy** to get a scrollable text view of the full terminal output |
| **Auto-reconnect** | Terminal reloads automatically when your phone wakes from sleep |

## Security

All services bind to the Tailscale interface IP only — unreachable from the public internet and your local network. Tailscale uses peer-to-peer [WireGuard](https://www.wireguard.com/) encryption, and every device must authenticate via SSO.

**Be aware:** This gives full terminal access to your machine. Anyone on your Tailscale network can connect. Use [Tailscale ACLs](https://tailscale.com/kb/1018/acls) if you want to restrict which devices can reach your slave.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| ttyd won't start | Port in use? `lsof -i :7681` |
| Can't connect from phone | Tailscale active on both devices? `tailscale status` |
| Voice dictation duplicates text | Use Web UI (`:8080`), not raw terminal (`:7681`) |
| launchd can't access ~/Documents/ | Copy scripts to `~/.local/bin/remote-cli/` |
| systemd service won't start | `journalctl --user -u remote-cli` |
| Connection drops on phone sleep | Just reopen — auto-reconnect handles it |

## License

MIT — Do whatever you want. The slaves certainly can't stop you.
