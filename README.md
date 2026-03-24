# kernel-idle: Advanced SDDM Power Management

A deterministic, kernel-level hardware idle monitor that enforces Display Power Management and Suspend policies on the SDDM login screen. It gracefully yields control to KDE Plasma once a user logs in, providing a seamless, feature-complete power management experience.

## Why This Exists
SDDM currently does not natively honor system power settings when no user is logged in. Existing workarounds (like `xautolock` or simple `sleep` loops) often fall short because they:
* Don't play well with Wayland.
* Cause black screens upon resuming from suspend (especially with NVIDIA).
* Trigger "false wakes" from ACPI events (like a 1% battery drop).
* Block I/O, meaning they cannot dynamically adapt if you unplug your laptop on the login screen.

## The Architecture
`kernel-idle` is written in pure Bash with a tiny Python footprint for non-blocking input polling. It acts as a smart state machine with the following core features:

* **Zero-Config (Dynamic Sync):** Automatically scans for the primary user and reads their KDE Plasma (`powerdevilrc`) settings. It seamlessly applies your actual GUI timeouts for AC, Battery, and Low Battery (<=20%).
* **Wayland & NVIDIA Safe:** Executes a specific pre-suspend sequence (saving the VT, switching to isolated VT 8, VESA blanking, and cutting physical backlight power) to prevent Wayland crash loops and black screens on resume.
* **Smart Polling (No I/O Blocking):** Uses a 5-second Python `select` heartbeat strictly listening to physical input (`/dev/input/by-path/*-event-kbd` and `*-event-mouse`). It ignores false wakes and instantly adapts to AC/Battery state changes.
* **Unobtrusive:** Once a human user logs in, the daemon yields power management back to the Desktop Environment and enters a silent sleep.

## Compatibility & Requirements
* **Distros:** Tested and confirmed working perfectly on **CachyOS (Arch Linux)** and **Kubuntu 25.10**.
* **Systemd:** Required for session tracking (`loginctl`) and suspending (`systemctl`).
* **Display Manager:** Explicitly targets SDDM (`sddm` or `plasmalogin` users).
* **Desktop Environment:** Syncs dynamically with KDE Plasma. Functions perfectly on other DEs but will safely use the fallback timeouts defined at the top of the script.
* **Hardware:** Display power-off (`/sys/class/backlight/`) is optimized for laptop internal displays. External monitors may only receive VESA blanking depending on GPU driver support.

---

## Installation

### 1. Create the Script

Save the `kernel-idle.sh` script to your local binaries. *(You can copy the code from the "The Script" section below)*.

```bash
sudo nano /usr/local/bin/kernel-idle.sh
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/kernel-idle.sh
```

### 2. Create the Systemd Service

Create the service file:

```bash
sudo nano /etc/systemd/system/kernel-idle.service
```

Paste the following configuration:

```ini
[Unit]
Description=Kernel Level Hardware Idle Monitor for SDDM
After=display-manager.service

[Service]
Type=simple
ExecStart=/usr/local/bin/kernel-idle.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3. Enable and Start

Reload the daemon and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kernel-idle.service
```

---

## Observability

The script logs its state changes clearly without spamming your journal. You can monitor it in real-time by running:

```bash
journalctl -u kernel-idle.service -f
```

### Example Logs:

Here is an example of `kernel-idle` in action: detecting the user's config, yielding to an active session, taking control back after logout, reacting to AC/Battery changes, and executing a safe suspend cycle.

```text
Mar 24 12:08:50 kubuntu-legion7 systemd[1]: Started kernel-idle.service - Kernel Level Hardware Idle Monitor for SDDM.
Mar 24 12:08:50 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: Service initializing...
Mar 24 12:08:50 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: Startup Config [KDE Plasma] -> AC[Disp:10m | Susp:60m] BAT[Disp:5m | Susp:10m] LOW[Disp:2m | Susp:5m] (User: sahindirek)
Mar 24 12:08:50 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: No active user session (SDDM). Took control with 'AC' profile. Timer starting fresh at 0s -> Display: 10m, Suspend: 60m
Mar 24 12:09:47 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: User 'sahindirek' is logged in. Yielding power management to KDE.
Mar 24 12:10:07 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: No active user session (SDDM). Took control with 'AC' profile. Timer starting fresh at 0s -> Display: 10m, Suspend: 60m
Mar 24 12:10:13 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: Power state changed to 'Battery'. Resetting timer. Timeouts -> Display: 5m, Suspend: 10m
Mar 24 12:10:18 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: Power state changed to 'AC'. Resetting timer. Timeouts -> Display: 10m, Suspend: 60m
Mar 24 12:20:30 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: Display idle timeout (10m) reached. Turning off display.
Mar 24 13:11:46 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: Suspend idle timeout (60m) reached. Preparing for suspend.
Mar 24 13:11:50 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: System resumed. Restoring display state.
Mar 24 13:13:51 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: User 'sahindirek' is logged in. Yielding power management to KDE.
Mar 24 13:14:11 kubuntu-legion7 kernel-idle.sh[2601]: kernel-idle: No active user session (SDDM). Took control with 'AC' profile. Timer starting fresh at 0s -> Display: 10m, Suspend: 60m
```

---
