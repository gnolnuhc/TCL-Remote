# Google TV & Android TV - Free Phone Remote (PWA)

> A self-owned, 100% free, no-cloud, no-subscription phone remote for **any Google TV and Android TV**. Runs on your Mac, Pi, or NAS — open on any phone, tablet, or laptop as a native-feeling PWA. Originally built for TCL, now generic for all Google TV & Android TV.

No more hunting for the physical remote. No manufacturer account. No ads. No cloud.

![PWA](https://img.shields.io/badge/PWA-Phone%20Ready-black) ![Platform](https://img.shields.io/badge/Platform-iPhone%20%26%20Android%20%26%20Desktop-black) ![Python](https://img.shields.io/badge/Python-3.9%2B-blue) ![License](https://img.shields.io/badge/License-MIT-green) ![No Cloud](https://img.shields.io/badge/Cloud-None-red) ![Compatible](https://img.shields.io/badge/Compatible-Google%20TV%20%26%20Android%20TV-blue)

---

<img width="1284" height="2246" alt="IMG_4651" src="https://github.com/user-attachments/assets/6e9b50e4-9ae2-4761-8dcb-eb496e20cef6" />

### Why this?

Official manufacturer apps (TCL, Sony, Hisense) and Google TV app are bloated, require accounts, and break. This is:

- **Free forever** — you own the code and certs
- **Local only** — talks directly to TV over TLS on your LAN (port 6466/6467)
- **PWA** — Add to Home Screen on any phone (iPhone & Android), feels like a native app, works offline
- **Fixes real pain points** — auto-discovery, last IP cache, Wake-on-LAN, 16s idle disconnect fix, power-cycle recovery, mobile audio unlock (iOS & Android)
- **Generic** — works on any device that speaks Android TV Remote v2, not just TCL

### Compatibility

| Device | Status | Notes |
| :--- | :--- | :--- |
| **TCL Google TV** | ✅ Tested | Original target, fully working |
| **Sony Bravia Google TV / Android TV** | ✅ Should work | Enable Network standby |
| **Hisense Google TV** | ✅ Should work | Enable Keep network on |
| **Chromecast with Google TV 4K/HD** | ✅ Should work | WOL not supported (no Ethernet) |
| **Walmart Onn 4K Box/Stick** | ✅ Should work | |
| **Nvidia Shield TV (2019+)** | ✅ Should work | Enable Network standby |
| **Any Android TV 10+ / Google TV** | ✅ Should work | Must advertise `_androidtvremote2` |

**Not supported:** Roku, Fire TV, Apple TV, LG webOS, Samsung Tizen, older Android TV v1 (`_androidtvremote._tcp`).

If your TV shows a PIN when you run the script, it’s compatible.

### Features

- **Premium glassmorphism UI** — 390px wide, dark, iOS style
- **Trackpad D-Pad** — 280px circle, swipe >18px = direction, tap <22px = OK, real-time arrow highlight
- **Click sound + haptics** — Web Audio square-wave clicks (different freq per key), `navigator.vibrate` fallback
- **Live status** — 🟢 Connected / 🟡 Reconnecting... / 🔴 Offline dot + banner, `/status` polling every 5s
- **Auto-discovery** — mDNS `_androidtvremote2._tcp.local.` via `zeroconf`, auto-selects first TV after 3s timeout
- **Last IP memory** — saves to `google_tv_last_ip.txt`, tries saved IP first with 2s reachability check
- **Wake-on-LAN** — grabs MAC via ARP, sends magic packet to multiple broadcasts (where supported by device)
- **Keepalive + idle fix** — Android TV Remote v2 kills idle connections after 16s with stale `is_on` flag. We check live transport (`transport.is_closing()`), keepalive every 10s, thread-safe `run_coroutine_threadsafe`, retry 3× (POWER 8×), auto-recreate remote after 30s offline
- **Power-cycle recovery** — detects power off/on, recreates `AndroidTVRemote` instance, increases DPAD attempts to 6× when offline >20s
- **iOS audio unlock** — global capture listeners + silent buffer unlock, sound plays on first tap/swipe even after `preventDefault`
- **Clean controls** — Power, Home, Back, Vol, Mute, Input, Settings, Play/Pause, D-Pad only

---

### How it works

```
Phone / Tablet (PWA) --HTTP :8000--> Flask on Mac/Pi/NAS --TLS :6466--> Any Google TV / Android TV
                              |
                              +--> zeroconf discovery + ARP MAC + WOL
```

Protocol: `androidtvremote2` Python library, correct order `AndroidTVRemote(client_name, certfile, keyfile, host)`. First run generates `cert.pem`/`key.pem`, triggers PIN on TV, then cert is whitelisted forever by the TV.

### Requirements

- Same WiFi for TV and server (Mac/Pi/NAS + phone/tablet)
- Python 3.9+
- Any Google TV or Android TV with **Network standby / Keep network on** enabled (brand-specific): Settings > Network & Internet > Keep network on / Network standby
- Any modern phone/tablet: iPhone iOS 16+ (Safari) or Android 8+ (Chrome) for PWA

```bash
pip install androidtvremote2 flask flask-cors zeroconf
```

### Quick Start (MacBook)

1. Clone / download `google_tv_remote.py` (or `tcl_one_mac_fixed.py`) to `~/Downloads`

2. Run:
```bash
cd ~/Downloads
python3 google_tv_remote.py
```

3. First run:
   - Script scans 10s for TVs (any brand)
   - Shows list: `1. Living Room TV -> 192.168.1.42`
   - Auto-selects first after 3s, or type number / IP
   - If new, TV shows 6-digit PIN → enter in terminal
   - `cert.pem` + `key.pem` saved in same folder

4. Open on phone:
   - **iPhone:** Safari → `http://<YOUR_SERVER_IP>:8000` → Share → **Add to Home Screen** → "TV Remote"
   - **Android:** Chrome → `http://<YOUR_SERVER_IP>:8000` → Menu ⋮ → **Add to Home screen** / **Install app** → "TV Remote"
   - **Any laptop/tablet:** Just open the URL in browser

That's it. Works forever, cert stays paired. Move `cert.pem`/`key.pem` to Pi/NAS if you change server.

### Deployment — Don't run on a laptop long-term

Laptops sleep and break WOL/keepalive. Best: always-on device.

**Raspberry Pi Zero 2 W (recommended, $15, always on, no laptop needed):**
```bash
# On Pi
sudo apt install python3-pip
pip3 install androidtvremote2 flask flask-cors zeroconf
# Copy script + certs
python3 google_tv_remote.py
# Make service
sudo nano /etc/systemd/system/googletv-remote.service
```
Service file:
```ini
[Unit]
Description=Google TV Remote
After=network.target

[Service]
WorkingDirectory=/home/pi/googletv-remote
ExecStart=/usr/bin/python3 /home/pi/googletv-remote/google_tv_remote.py
Restart=always

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable --now googletv-remote
```

**Mac stay-awake workaround:**
```bash
caffeinate -dimsu python3 google_tv_remote.py
```
Or create `~/Desktop/Google TV Remote.command`:
```bash
#!/bin/bash
cd ~/Downloads
caffeinate -dimsu python3 google_tv_remote.py
```
`chmod +x ~/Desktop/Google\ TV\ Remote.command` → double-click to start.

### UI Guide

- **Power** ⏻ — sends WOL if supported/offline, retries 8×, toast "Waking TV..."
- **Trackpad** — swipe = direction (plays sound mid-swipe), tap = OK — works on phone & desktop
- **Volume row** — − / MUTE / + horizontal
- **Status dot** — top left, polls `/status`
- **Banner** — yellow when reconnecting, red when offline

### Troubleshooting

**Trackpad does nothing after power off/on?**
Fixed in latest version: JS `isPressing` flag no longer gets stuck true after retry, and server recreates remote after 30s offline. Update script.

**No sound until Home tapped? (old mobile bug)**
Fixed: global audio unlock listeners with capture phase + silent buffer + no `preventDefault` on `touchstart`/`touchend`. First tap/swipe now plays on iPhone & Android.

**First swipe no sound but tap does? (old mobile bug)**
Fixed: sound now plays in `handleMove` when direction locks (>18px), not just `handleEnd`, and `playClick()` awaits `resume()` for both iOS and Android.

**Clicks stop after 1-2 presses (idle bug)?**
AndroidTVRemote v2 has 16s idle watchdog with stale `is_on`. Fixed: `has_live_connection()` checks `transport.is_closing()`, keepalive every 10s, `command_lock` prevents race, auto-recreate remote on failure.

**TV offline after sleep?**
Enable "Keep network on" / "Network standby" in TV settings. Ensure MAC was captured (script prints `📋 TV MAC`). If unknown, WOL disabled — expected for sticks like Chromecast.

**Phone can't connect?**
Check same WiFi, firewall on server: allow Python. Use `http://`, not `https`. Try `http://<SERVER_IP>:8000` on same WiFi.

**Pairing fails with `{"error":"","status":"error"}` or `nodename nor servname`?**
You used placeholder IP `192.168.1.XX`. Use real IP from TV: Settings > Network & Internet > WiFi > IP.

**Port 8000 in use?**
Change `PORT = 8000` at top of script.

**Works on TCL but not Sony/Hisense?**
Ensure TV advertises `_androidtvremote2`. Some older Android TVs use v1 — try `androidtvremote` library instead. Open issue with TV model + logs.

### File Layout

```
google_tv_remote.py        # single-file Flask + PWA + pairing + discovery + WOL + keepalive
cert.pem / key.pem         # generated once, keep safe, whitelisted by TV (per TV)
google_tv_last_ip.txt      # auto-saved last IP (or tcl_last_ip.txt for legacy)
```

### Security

- All traffic local LAN only, TLS to TV using your own self-signed cert
- No telemetry, no cloud, no account
- Certs are your TV's whitelist — don't commit `cert.pem`/`key.pem` to GitHub (add to `.gitignore`)

Add `.gitignore`:
```
cert.pem
key.pem
*_last_ip.txt
__pycache__/
```

### Roadmap

- [ ] Volume slider
- [ ] Keyboard input
- [ ] Configurable app launch buttons (via `send_launch_app_command` — package names vary by brand)
- [ ] Docker image
- [ ] Auto-reconnect for multiple TVs

### Contributing

PRs welcome! Please test on real Google TV / Android TV (not Roku/Fire TV). Include brand/model in PR description. Run `python -m py_compile google_tv_remote.py` before PR.

Originally built for TCL, now generic — thanks to testers on Sony, Hisense, and Chromecast.

### License

MIT — free for personal and commercial use. No warranty.

---

Built with frustration for lost remotes and love for local-first software. Tested on TCL, works on any Google TV & Android TV, from any phone (iPhone & Android). If this saved you $20 on a replacement remote, give it a ⭐.
