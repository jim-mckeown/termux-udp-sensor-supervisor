# README — WAV-Based Patient Supervision Engine (v1.0.0)

## 📌 1. Overview & Purpose

The **WAV-Based Patient Supervision Engine (v1.0.0)** is a lightweight, real-time asynchronous daemon designed for monitoring distributed medical and safety sensors (such as motion detectors, bed scale sensors, and emergency doorbells) across a local Wi-Fi network.

Operating on **UDP Port 8888**, the script acts as a central control unit that:

1. **Listens for Critical Alarms:** Listens for real-time sensor events and latched state back-ups.
2. **Plays Precise Audio Alerts:** Dynamically parses `.wav` binary headers using Python’s built-in `wave` module to calculate audio runtime down to the millisecond, preventing audio overlap or premature clipping.
3. **Supports Multi-Language Queuing:** Plays alert sounds sequentially across multiple languages (e.g., English, Thai).
4. **Resets Sensor Latches:** Automatically dispatches a clear command (`status 0`) back to the triggering device upon audio dispatch.
5. **Monitors Sensor Health:** Periodically queries connected hardware (every 10 seconds) and flags supervision loss if a device stops responding for 30 seconds.

---

## 🛠️ 2. Prerequisites & Environment Setup

### System Requirements

* **Python Version:** Python 3.7+ (Uses standard built-in modules only: `socket`, `time`, `subprocess`, `argparse`, `select`, `os`, `sys`, `wave`). No third-party `pip` packages required!
* **Supported Platforms:** Android (Termux), Linux / Raspberry Pi, Windows.

### Required Media Player Executable

The script invokes external media commands to output `.wav` sound files depending on the target system. Ensure the executable for your OS is installed:

* **Android (Termux Default):** `termux-media-player`
* Install via Termux terminal: `pkg install termux-api`


* **Linux / Raspberry Pi:** `mpv` or `aplay`
* Install via package manager: `sudo apt install mpv`


* **Windows:** `ffplay` (included with standard `ffmpeg` binaries added to system `PATH`).

---

## 📁 3. Directory Structure & File Setup

Audio files must be structured into language-specific folders relative to where the script is run. Filenames correspond to the **4-digit alert code** (Device ID + Message Type):

```text
.
├── supervision_engine.py      # Main script
├── en/                        # English sound directory
│   ├── 0101.wav               # Motion Detector (01) Alarm
│   ├── 0201.wav               # Bed Scale (02) Alarm
│   └── 0501.wav               # Doorbell (05) Alarm
└── th/                        # Thai sound directory
    ├── 0101.wav
    ├── 0201.wav
    └── 0501.wav

```

---

## ⚙️ 4. Configuration

Open `supervisor.py` in a text editor to configure device mappings and system media players prior to launching.

### A. Network & Device Mapping Matrix

Map physical IP addresses on your subnet to logical 2-digit IDs and readable display names:

```python
DEVICES = {
    "192.168.1.108": {"id": "01", "name": "Motion Detector 01"},
    "192.168.1.138": {"id": "02", "name": "Bed Scale 02"},
    "192.168.1.100": {"id": "05", "name": "Doorbell 05"}
}

```

### B. Media Player Command

Modify `PLAYER_CMD` depending on your host OS:

```python
# Termux (Android)
PLAYER_CMD = ["termux-media-player", "play"]

# Linux / Raspberry Pi (Alternative)
# PLAYER_CMD = ["mpv", "--no-terminal"]

# Windows (Alternative)
# PLAYER_CMD = ["ffplay", "-nodisp", "-autoexit"]

```

---

## 🚀 5. How to Run

Launch the script from your terminal using command-line flags to specify which language audio alerts to enable:

### Command Options

| Flag | Description |
| --- | --- |
| `--en` | Enables English WAV alerts (`en/*.wav`) |
| `--th` | Enables Thai WAV alerts (`th/*.wav`) |

### Example Launch Commands

```bash
# Run with English alerts only
python supervisor.py --en

# Run with dual English and Thai sequential alerts
python supervisor.py --en --th

# Running with no flags defaults to English (en/)
python supervisor.py

```

---

## 📡 6. Network Protocol & Payload Summary

Communication runs over **UDP Port 8888**. The table below details supported network commands:

| Payload | Type | Protocol Action |
| --- | --- | --- |
| `msg <code>` | **Inbound** | Triggers instantaneous alarm audio for `<code>` (e.g. `0101`) and returns `status 0` to sensor. |
| `OK status 2` | **Inbound** | Latched state query backup. Formats code as `{sensor_id}01`, plays audio, and returns `status 0`. |
| `alert <text>` | **Inbound** | Logs informal soft alert message to system terminal output. |
| `status 0` | **Outbound** | Clears the hardware state latch on the target IP sensor node after alarm dispatch. |
| `status ?` | **Outbound** | Polling ping sent to each configured sensor every 10 seconds to verify online health. |

---

## 💡 7. Operational & Troubleshooting Tips

1. **Android / Termux Battery Optimization:**
* Android OS aggressively puts background processes to sleep. If running under Termux, run `termux-wake-lock` in the shell prior to starting the script to prevent Wi-Fi and CPU sleep during background monitoring.


2. **Network Timeout & Recovery:**
* If a device goes un-updated for **>30 seconds** (3 polling intervals), the console logs `[CRITICAL] SUPERVISION LOST ON <DEVICE>!`. The device state will automatically recover and log `[NETWORK RECOVERED]` as soon as a packet is received again.


3. **Corrupted or Missing WAV Files:**
* If a target `.wav` file is corrupted, the system falls back to a safe `3.0s` sleep delay. Missing `.wav` files will log an `[AUDIO-ERR]` without crashing the main loop.
