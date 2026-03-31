# 🤖 Xiaomi Mi Robot Vacuum-Mop Essential G1 — WiFi Fix Guide

> **Fix for:** Mi Robot Vacuum-Mop Essential G1 (MJSTG1 / SKV4136GL)  
> **Problem:** Vacuum not appearing on WiFi / Mi Home after disassembly  
> **Solution:** Directly flash WiFi credentials to ESP32 via UART  
> **Author:** Based on real hardware hacking experience  

---

## 📋 Table of Contents
- [The Problem](#the-problem)
- [Root Cause](#root-cause)
- [Hardware Required](#hardware-required)
- [WIFIPCB Pinout](#wifipcb-pinout)
- [Software Setup](#software-setup)
- [Step by Step Fix](#step-by-step-fix)
- [Verify Connection](#verify-connection)
- [Local Control Without Mi Home](#local-control-without-mi-home)

---

## ❓ The Problem

After disassembling the G1 vacuum, the robot:
- Does not appear on WiFi networks
- Does not show up in Mi Home app
- Shows two white blinking lights (pairing mode) but never connects

---

## 🔍 Root Cause

The G1 uses an **ESP32-ROOM-32D** module (R1_WIFIPCB V1.1) for WiFi connectivity.

After disassembly and reassembly, the WiFi credentials stored in the ESP32 flash memory can be wiped or corrupted.

**Key finding:** Reading the ESP32 flash revealed:
```
sta.ssid = (empty)
```
The SSID field was completely empty — the vacuum had no WiFi network to connect to.

---

## 🛠️ Hardware Required

| Item | Purpose | Approx. Cost |
|------|---------|--------------|
| CP2102 USB to TTL UART | Flash ESP32 | $2–4 |
| 4x Dupont jumper wires | Connections | $1 |
| Windows/Linux/Mac PC | Run esptool | — |

**CP2102 module example:**
```
CP2102 USB to TTL UART STC Cable
(any CP2102 or CH340 module works)
```

---
![photo_1_2026-03-31_04-39-41](https://github.com/user-attachments/assets/617ec2bd-e227-4f25-9de4-dd2d3ba543a3)
![photo_2_2026-03-31_04-39-41](https://github.com/user-attachments/assets/7431dc37-f086-43b3-a2d2-5831000f7439)
![photo_3_2026-03-31_04-39-41](https://github.com/user-attachments/assets/5d339bad-2e74-46e5-9b34-9ffd74c63fe1)

## 📌 WIFIPCB Pinout

The **R1_WIFIPCB V1.1** board has labeled test pads on the back:

```
┌─────────────────────────────┐
│  ESP32-ROOM-32D             │
│                             │
│  Back side pads:            │
│  T1 = TX0  (UART0 TX)      │
│  T2 = RX0  (UART0 RX)      │
│  T3 = IO0  (GPIO0 Boot)    │
│  T6 = GND                  │
│  T7 = VCC  (3.3V)          │
│  T4 = TX2  (to MainBoard)  │
│  T5 = RX2  (to MainBoard)  │
└─────────────────────────────┘
```

> ⚠️ **Important:** Use 3.3V only — never 5V, it will damage the ESP32!

---

## 💻 Software Setup

### Windows

**1. Install CP2102 Driver**
Download from: https://www.silabs.com/developer-tools/usb-to-uart-bridge-vcp-drivers

Install `silabser.inf` (right-click → Install)

**2. Install Python & esptool**
```bash
pip install esptool
```

Or if using Python Launcher:
```bash
py -m pip install esptool
```

**3. Find COM port**

Open Device Manager → Ports (COM & LPT) → look for `Silicon Labs CP210x`

Note the COM number (e.g., COM4)

---

## 🔧 Step by Step Fix

### Step 1 — Wire the connections

```
CP2102        →    WIFIPCB (T pads)
──────────────────────────────────
GND           →    T6 (GND)
3.3V          →    T7 (VCC)
TX            →    T2 (RX0)
RX            →    T1 (TX0)
(wire)        →    T3 (IO0) → T6 (GND)   ← Flash mode!
```

> 🔑 **Critical:** T3 (IO0) must be connected to GND **before** powering on to enter flash mode.

### Step 2 — Verify connection

```bash
py -m esptool --port COM4 chip-id
```

Expected output:
```
Connected to ESP32 on COM4:
Chip type: Unknown ESP32 (revision v1.0)
MAC: 58:b6:23:03:43:16
```

### Step 3 — Backup original firmware (Important!)

```bash
py -m esptool --port COM4 --baud 115200 read-flash 0x0 0x400000 backup.bin
```

> 💾 Always keep this backup! It contains your original firmware.

### Step 4 — Create WiFi config file

Create a Python script `write_wifi.py`:

```python
ssid = b'YourWiFiName'        # 2.4GHz only, no special characters
password = b'YourPassword'

data = bytearray(4096)
data[0:len(ssid)] = ssid
data[64:64+len(password)] = password

with open('wifi_config.bin', 'wb') as f:
    f.write(data)

print('Done! wifi_config.bin created.')
```

Run it:
```bash
py write_wifi.py
```

### Step 5 — Flash WiFi credentials

```bash
py -m esptool --port COM4 --baud 115200 write-flash 0x9000 wifi_config.bin
```

Expected output:
```
Wrote 4096 bytes (43 compressed) at 0x00009000
Hash of data verified.
```

### Step 6 — Boot normally

1. Disconnect T3 (IO0) from GND
2. Disconnect CP2102
3. Reassemble the vacuum
4. Power on

The vacuum will now connect to your WiFi network! ✅

---

## ✅ Verify Connection

After powering on, find the vacuum's IP:

```bash
# Windows
arp -a

# Or scan the network
for /l %i in (1,1,254) do ping -n 1 -w 100 192.168.x.%i | find "Reply"
```

Then verify with python-miio:

```bash
pip install python-miio
```

```python
from miio import MiotDevice

mapping = {
    "status":    {"siid": 2, "piid": 4},
    "fault":     {"siid": 2, "piid": 2},
    "battery":   {"siid": 3, "piid": 1},
    "fan_speed": {"siid": 2, "piid": 6},
}

vac = MiotDevice('YOUR_VACUUM_IP', 'YOUR_TOKEN', mapping=mapping)
props = vac.get_properties_for_mapping()
for p in props:
    print(p)
```

Expected output:
```python
{'did': 'status',   'code': 0, 'value': 1}   # 1 = Standby
{'did': 'fault',    'code': 0, 'value': 0}   # 0 = No fault
{'did': 'battery',  'code': 0, 'value': 36}  # 36%
{'did': 'fan_speed','code': 0, 'value': 1}   # 1 = Silent
```

---

## 🏠 Local Control Without Mi Home

### Get Token

Use [Xiaomi Cloud Token Extractor](https://github.com/PiotrMachowski/Xiaomi-cloud-tokens-extractor):

```
NAME:   Mi Robot Vacuum-Mop Essential
MODEL:  mijia.vacuum.v2
IP:     172.20.10.x
TOKEN:  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Control via Python

```python
from miio import MiotDevice

vac = MiotDevice('192.168.x.x', 'your_token_here', mapping=mapping)

# Start cleaning
vac.call_action_by(2, 1)

# Stop
vac.call_action_by(2, 2)

# Return to dock
vac.call_action_by(2, 3)

# Set fan speed (1=Silent, 2=Standard, 3=Medium, 4=Turbo)
vac.set_property_by(2, 6, 2)
```

---

## 📊 MiOT Property Map (mijia.vacuum.v2)

| Property | SIID | PIID | Values |
|----------|------|------|--------|
| Status | 2 | 4 | 1=Standby, 5=Cleaning, 6=Returning, 8=Charging |
| Fault | 2 | 2 | 0=No fault |
| Fan Speed | 2 | 6 | 1=Silent, 2=Standard, 3=Medium, 4=Turbo |
| Battery | 3 | 1 | 0–100% |

| Action | SIID | AIID |
|--------|------|------|
| Start Clean | 2 | 1 |
| Stop Clean | 2 | 2 |
| Return Dock | 2 | 3 |

---

## ⚠️ Important Notes

- WiFi must be **2.4GHz** — 5GHz is not supported
- SSID must use **English characters only** — no spaces or special chars
- Always backup firmware before flashing
- The token is required for local control — extract it while connected to Mi Home at least once

---

## 🔗 Related Projects

- [Valetudo](https://github.com/Hypfer/Valetudo) — Local only vacuum control
- [python-miio](https://github.com/rytilahti/python-miio) — Python library for Xiaomi devices
- [Xiaomi Cloud Token Extractor](https://github.com/PiotrMachowski/Xiaomi-cloud-tokens-extractor)

---

## 📝 License

MIT License — feel free to use and share!

---

*Tested on: MJSTG1 / SKV4136GL (Global version)*  
*Firmware: mijia.vacuum.v2*  
*ESP32 MAC: 58:B6:23:03:43:16 (your MAC will differ)*
