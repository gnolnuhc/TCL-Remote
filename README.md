# TCL Google TV - Free iPhone Remote (PWA)

> A self-owned, 100% free, no-cloud, no-subscription iPhone remote for TCL Google TV. Runs on your Mac, Pi, or NAS — open to your iPhone as a native-feeling PWA.

No more hunting for the physical remote. No TCL account. No ads.

![PWA](https://img.shields.io/badge/PWA-iPhone%20Ready-black) ![Python](https://img.shields.io/badge/Python-3.9%2B-blue) ![License](https://img.shields.io/badge/License-MIT-green) ![No Cloud](https://img.shields.io/badge/Cloud-None-red)

---

### Why this?

The official TCL / Google TV apps are bloated, require accounts, and break. This is:

- **Free forever** — you own the code and certs
- **Local only** — talks directly to TV over TLS on your LAN (port 6466/6467)
- **PWA** — Add to Home Screen on iPhone, feels like a native app, works offline
- **Fixes real pain points** — auto-discovery, last IP cache, Wake-on-LAN, 16s idle disconnect fix, power-cycle recovery, iOS audio unlock

### Features

- **Premium glassmorphism UI** — 390px wide, dark, iOS style
- **Trackpad D-Pad** — 280px circle, swipe >18px = direction, tap <22px = OK, real-time arrow highlight
- **Click sound + haptics** — Web Audio square-wave clicks (different freq per key), `navigator.vibrate` fallback
- **Live status** — 🟢 Connected / 🟡 Reconnecting... / 🔴 Offline dot + banner, `/status` polling every 5s
- **Auto-discovery** — mDNS `_androidtvremote2._tcp.local.` via `zeroconf`, auto-selects first TV after 3s timeout
- **Last IP memory** — saves to `tcl_last_ip.txt`, tries saved IP first with 2s reachability check
- **Wake-on-LAN** — grabs MAC via ARP, sends magic packet to multiple broadcasts (255.255.255.255, subnet .255, etc.)
- **Keepalive + idle fix** — Android TV Remote v2 kills idle connections after 16s with stale `is_on` flag. We check live transport (`transport.is_closing()`), keepalive every 10s, thread-safe `run_coroutine_threadsafe`, retry 3× (POWER 8×), auto-recreate remote after 30s offline
- **Power-cycle recovery** — detects power off/on, recreates `AndroidTVRemote` instance, increases DPAD attempts to 6× when offline >20s
- **iOS audio unlock** — global capture listeners + silent buffer unlock, sound plays on first tap/swipe even after `preventDefault`
- **No app buttons** — clean: Power, Home, Back, Vol, Mute, Input, Settings, Play/Pause, D-Pad only

---

### How it works

```
iPhone (Safari PWA) --HTTP :8000--> Flask on Mac/Pi/NAS --TLS :6466--> TCL Google TV
                              |
                              +--> zeroconf discovery + ARP MAC + WOL
```

Protocol: `androidtvremote2` Python library, correct order `AndroidTVRemote(client_name, certfile, keyfile, host)`. First run generates `cert.pem`/`key.pem`, triggers PIN on TV, then cert is whitelisted forever.

### Requirements

- Same WiFi for TV and server (Mac/Pi/NAS + iPhone)
- Python 3.9+
- TCL Google TV with **Network standby / Keep network on** enabled: Settings > Network & Internet > Keep network on (or similar)
- iPhone iOS 16+ for PWA

```bash
pip install androidtvremote2 flask flask-cors zeroconf
```

### Quick Start (MacBook)

1. Clone / download `tcl_remote.py` to `~/Downloads`

2. Run:
```bash
cd ~/Downloads
python3 tcl_remote.py
```

3. First run:
   - Script scans 10s for TVs
   - Shows list: `1. Family Room TV -> 192.168.86.43`
   - Auto-selects first after 3s, or type number / IP
   - If new, TV shows 6-digit PIN → enter in terminal
   - `cert.pem` + `key.pem` saved in same folder

4. Open on iPhone: Safari → `http://<YOUR_MAC_IP>:8000` (printed in terminal)
   - Share → **Add to Home Screen** → name it "TCL Remote" → now native app

That's it. Works forever, cert stays paired.

### Deployment — Don't run on a laptop long-term

Laptops sleep and break WOL. Best:

**Raspberry Pi Zero 2 W (recommended, $15, always on):**
```bash
# On Pi
sudo apt install python3-pip
pip3 install androidtvremote2 flask flask-cors zeroconf
# Copy script + certs
python3 tcl_remote.py
# Make service
sudo nano /etc/systemd/system/tcl-remote.service
```
Service file:
```ini
[Unit]
Description=TCL Remote
After=network.target

[Service]
WorkingDirectory=/home/pi/tcl-remote
ExecStart=/usr/bin/python3 /home/pi/tcl-remote/tcl_remote.py
Restart=always

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable --now tcl-remote
```

**Mac stay-awake workaround:**
```bash
caffeinate -dimsu python3 tcl_remote.py
```
Or create `~/Desktop/TCL Remote.command`:
```bash
#!/bin/bash
cd ~/Downloads
caffeinate -dimsu python3 tcl_remote.py
```
`chmod +x ~/Desktop/TCL\ Remote.command` → double-click to start.

### UI Guide

- **Power** ⏻ — sends WOL if offline, retries 8×, toast "Waking TV..."
- **Trackpad** — swipe = direction (plays sound mid-swipe), tap = OK
- **Volume row** — − / MUTE / + horizontal (fixed from off-screen bug)
- **Status dot** — top left, polls `/status`
- **Banner** — yellow when reconnecting, red when offline

### Troubleshooting

**Trackpad does nothing after power off/on?**
Fixed in latest version: JS `isPressing` flag no longer gets stuck true after retry, and server recreates remote after 30s offline. Update script.

**No sound until Home tapped? (old bug)**
Fixed: global audio unlock listeners with capture phase + silent buffer + no `preventDefault` on `touchstart`/`touchend`. First tap/swipe now plays.

**First swipe no sound but tap does? (old bug)**
Fixed: sound now plays in `handleMove` when direction locks (>18px), not just `handleEnd`, and `playClick()` awaits `resume()`.

**Clicks stop after 1-2 presses (idle bug)?**
AndroidTVRemote v2 has 16s idle watchdog with stale `is_on`. Fixed: `has_live_connection()` checks `transport.is_closing()`, keepalive every 10s, `command_lock` prevents race, auto-recreate remote on failure.

**TV offline after sleep?**
Enable "Keep network on" in TV settings. Ensure MAC was captured (script prints `📋 TV MAC`). If unknown, WOL disabled.

**iPhone can't connect?**
Check same WiFi, firewall on Mac: `System Settings > Firewall` allow Python. Use `http://`, not `https`.

**Pairing fails with `{"error":"","status":"error"}` or `nodename nor servname`?**
You used placeholder IP `192.168.1.XX`. Use real IP from TV: Settings > Network & Internet > WiFi > IP.

**Port 8000 in use?**
Change `PORT = 8000` at top of script.

### File Layout

```
tcl_remote.py  # single-file Flask + PWA + pairing + discovery + WOL + keepalive
cert.pem / key.pem    # generated once, keep safe, whitelisted by TV
tcl_last_ip.txt       # auto-saved last IP
```

### Security

- All traffic local LAN only, TLS to TV using your own self-signed cert
- No telemetry, no cloud, no account
- Certs are your TV's whitelist — don't commit `cert.pem`/`key.pem` to GitHub (add to `.gitignore`)

### Roadmap

- [ ] Volume slider
- [ ] Keyboard input
- [ ] Configurable buttons (app launch via `send_launch_app_command`)
- [ ] Docker image

### Contributing

PRs welcome! Please test on real TCL Google TV (not Roku). Run `python -m py_compile tcl_remote.py` before PR.

### License

MIT — free for personal and commercial use. No warranty.

---

Built with frustration for lost remotes and love for local-first software. If this saved you $20 on a replacement remote, give it a ⭐.
