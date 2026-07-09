# IMX500 Visitor Counter

Headless exposition visitor counter built on a Raspberry Pi Zero 2W and Sony IMX500 AI camera.
Detects and logs visitors in a hallway using
on-sensor inference (SSD MobileNetV2 FPN Lite, COCO labels). Runs automatically from
set times, with a live WebSocket stream and a historical event dashboard
available 24/7.

**Hardware:** Raspberry Pi Zero 2W · Sony IMX500 AI Camera  
**Model:** SSD MobileNetV2 FPN Lite 320×320 (on-sensor, no CPU inference load)  
**Stack:** Python · picamera2 · OpenCV · websockets · systemd

---

## Requirements

### Hardware

- Raspberry Pi Zero 2W (or Pi 4 / Pi 5)
- Sony IMX500 AI Camera connected via CSI ribbon cable
- MicroSD card (16GB or larger recommended)
- Power supply appropriate for your Pi model
- WiFi or Ethernet connectivity

### Software

- Raspberry Pi OS Debian Trixie (headless, 64-bit recommended)
- Internet connection during provisioning (for package installation)
- A computer with SSH access to the Pi

### Disk Space

At least 2GB free on the Pi after OS installation. The IMX500 model firmware
and Python packages are large downloads.

---

## Architecture

The system runs as two independent systemd services:

| Service | Script | Runs | Responsibility |
|---|---|---|---|
| `imx500_server.service` | `imx500_server.py` | Always (24/7) | HTTP server, WebSocket server, frame relay |
| `imx500_capture.service` | `imx500_capture.py` | Sunrise to sunset | Camera, AI inference, event logging |

The server starts at boot and stays up permanently. The capture script runs only
during daylight hours and sends annotated JPEG frames to the server via a Unix
socket (`/tmp/imx500_frames.sock`). When the camera is offline the dashboard
remains accessible and the live view shows "waiting for camera..."

### Web Interfaces

| URL | Content |
|---|---|
| `http://<pi-ip>:8081/` | Live camera stream with bounding boxes |
| `http://<pi-ip>:8081/dashboard` | Event log summary by day and label |
| `http://<pi-ip>:8081/summary.json` | Raw summary data (JSON) |
| `ws://<pi-ip>:8080/` | WebSocket frame stream |

---

## Git Setup on Raspberry Pi OS Debian Trixie (Headless)

### Update, Install Git, and Reboot

```bash
sudo apt update && sudo apt upgrade -y && sudo apt install -y git && sudo reboot
```

### Clone the Repository
```bash
cd ~
git clone https://github.com/sezabart/imx500.git
```

### Prevent Permission Change Conflicts

Raspberry Pi OS marks shell scripts as executable with `chmod +x`, which git
tracks as a file change and blocks future pulls. Disable permission tracking
in this repo so that doesn't happen:

```bash
cd ~/imx500
git config core.fileMode false
```

This is a local repo setting — it only affects this clone on the Pi and does
not change anything in GitHub.

---

## Provisioning

Run the three provisioning scripts in order. The first script ends with a reboot.

```bash
cd ~/imx500
chmod +x *.sh
sudo ./imx500pi_provision.sh <username>
# --- reboot to apply boot config changes ---
sudo reboot
cd ~/imx500
sudo ./imx500pi_provision_python.sh
sudo ./imx500pi_provision_service.sh
```

### What each script does

| Script | Purpose |
|---|---|
| `imx500pi_provision.sh` | Base OS: camera interface, GPU memory, log directory |
| `imx500pi_provision_python.sh` | Python venv with all required packages |
| `imx500pi_provision_service.sh` | Location config, systemd service units, service alias, start everything |

### Re-Provisioning

All provisioning scripts are re-runnable and display a pre-run state check
before making any changes.

| Script | Re-run behavior |
|---|---|
| `imx500pi_provision.sh` | Skips already-applied changes, re-applies missing ones |
| `imx500pi_provision_python.sh` | Skips installed packages and existing venv; use `--reset` to rebuild venv |
| `imx500pi_provision_service.sh` | Stops services, rewrites unit files, restarts; use `--reset` to reconfigure location |

---

## Service Aliases

`systemctl --user` requires `XDG_RUNTIME_DIR` to be set when run via sudo or
from certain shell contexts. The service provisioning script adds this alias
to `~/.bashrc` automatically:

```bash
alias imx500='XDG_RUNTIME_DIR=/run/user/$(id -u) systemctl --user'
```

Open a new shell (or run `source ~/.bashrc`) after provisioning to activate it.

### Useful Service Commands

All commands below assume the `imx500` alias is configured. If not, prefix
each `systemctl --user` call with `XDG_RUNTIME_DIR=/run/user/$(id -u)`.

| Action | Command |
|---|---|
| Start server | `imx500 start imx500_server.service` |
| Stop server | `imx500 stop imx500_server.service` |
| Restart server | `imx500 restart imx500_server.service` |
| Server status | `imx500 status imx500_server.service` |
| Start capture | `imx500 start imx500_capture.service` |
| Stop capture | `imx500 stop imx500_capture.service` |
| Restart capture | `imx500 restart imx500_capture.service` |
| Capture status | `imx500 status imx500_capture.service` |
| Timer status | `imx500 list-timers imx500_capture.timer` |
| Server journal | `journalctl --user -u imx500_server.service -f` |
| Capture journal | `journalctl --user -u imx500_capture.service -f` |
| Event log | `tail -f /var/log/imx500/events.jsonl` |
| Wrapper log | `tail -f /var/log/imx500/wrapper.log` |

---

## Verify the Installation

Check both services are running:
```bash
imx500 status imx500_server.service
imx500 status imx500_capture.service
```

Check the timer is active:
```bash
imx500 list-timers imx500_capture.timer
```

Tail the event log:
```bash
tail -f /var/log/imx500/events.jsonl
```

Follow the journal for each service:
```bash
journalctl --user -u imx500_server.service -f
journalctl --user -u imx500_capture.service -f
```

---


### Controlling log file retention

The number of daily event log files kept in `/var/log/imx500/` is controlled
by `max_log_files` in `config.json`. The default is 30 days. To change it,
edit the value and restart the capture service:

```json
"logging": {
  "max_log_files": 14
}
```

```bash
imx500 restart imx500_capture.service
```

If `max_log_files` is missing from `config.json`, the capture script defaults
to 30.

---

## Dashboard

The event log dashboard is served at `http://<pi-ip>:8081/dashboard` and is
available 24/7 via `imx500_server.service`.

`summary.json` is rebuilt each morning by `build_summary.py` before the camera
starts. It can also be rebuilt manually at any time:

```bash
~/imx500_venv/bin/python3 ~/imx500/build_summary.py
```
