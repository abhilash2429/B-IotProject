# 🔧 Hardware & Arduino Setup Guide
### NodeMCU ESP8266 + HC-SR501 Motion Sensor
**Do these steps in order — wiring first, software second.**

---

## 🔌 PART 1 — Wiring (Do This First, Board Unplugged)

**Before anything — make sure your NodeMCU is NOT plugged into USB yet.**

Your HC-SR501 has 3 pins on the bottom. Hold it with the **white dome facing you** — the pins will be at the bottom labeled:

```
┌─────────────────┐
│   (white dome)  │
└─────────────────┘
  VCC   OUT   GND
```

Make these 3 connections using jumper cables:

| HC-SR501 Pin | NodeMCU Pin | Wire Color (recommended) |
|---|---|---|
| VCC | 3V3 | Red |
| OUT | D3 | Yellow |
| GND | GND | Black |

---

### Finding the pins on your NodeMCU

Look at the text printed directly on your NodeMCU board:

```
┌─────────────────────┐
│  [USB port here]    │
│                     │
│  3V3  ←  use this   │
│  GND  ←  use this   │
│  ...                │
│  D3   ←  use this   │
│  ...                │
└─────────────────────┘
```

> ⚠️ **Do NOT use VIN or 5V** — it can damage the NodeMCU. Only use **3V3**.

Once wired, it should look like this:

```
HC-SR501          NodeMCU
─────────         ───────
  VCC  ────red────  3V3
  OUT  ───yellow──  D3
  GND  ───black──   GND
```

✅ Done wiring? **Now** plug the NodeMCU into your PC via USB.

---

## 💻 PART 2 — Arduino IDE Setup

### Step 1 — Download & Install

1. Go to **https://www.arduino.cc/en/software**
2. Click **"Windows Win 10 and newer, 64 bits"**
3. Click **"Just Download"**
4. Run the `.exe` and install (click Next → Next → Install)

---

### Step 2 — Add ESP8266 Support

Arduino IDE doesn't know about NodeMCU by default:

1. Open Arduino IDE
2. Go to **File → Preferences**
3. Find **"Additional boards manager URLs"** box
4. Paste this inside it:
   ```
   https://arduino.esp8266.com/stable/package_esp8266com_index.json
   ```
5. Click **OK**
6. Go to **Tools → Board → Boards Manager**
7. Search: `esp8266`
8. Find **"esp8266 by ESP8266 Community"** → click **Install**
9. Wait for it to finish (~150MB download)

---

### Step 3 — Install MQTT Library

1. Go to **Sketch → Include Library → Manage Libraries**
2. Search: `PubSubClient`
3. Find **"PubSubClient by Nick O'Leary"** → click **Install**
4. Close the Library Manager

---

### Step 4 — Select Your Board & Port

1. Go to **Tools → Board → esp8266 → NodeMCU 1.0 (ESP-12E Module)**
2. Go to **Tools → Port** → select the port that appeared (e.g. `COM3`)

> ⚠️ If no port appears, you need a driver. Download from:
> **https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers**
> → "CP210x Universal Windows Driver" → unzip → right-click `.inf` → Install → restart Arduino IDE

---

### Step 5 — Upload the Code

1. Go to **File → New** in Arduino IDE
2. Delete everything in the blank sketch
3. Paste the full ESP8266 code from your guide
4. Find these two lines and fill in your details:
```cpp
const char* ssid     = "YOUR_WIFI_NAME";
const char* password = "YOUR_WIFI_PASSWORD";
```
5. Make sure the topic matches your Android app exactly:
```cpp
const char* mqttTopic = "myproject/intrusiondetection";
```
6. Press **Ctrl + U** to upload
7. Watch the bottom of Arduino IDE — wait for **"Done uploading"**

---

## 🧪 PART 3 — Test It All Together

1. Go to **Tools → Serial Monitor** (top right magnifying glass icon)
2. Set baud rate to **9600** (bottom right dropdown)
3. You should see:
```
[INFO] Connecting to WiFi...
[INFO] WiFi Connected!
[INFO] Connecting to MQTT Broker... Connected!
[INFO] Calibrating sensor for 30 seconds...
..............................
[INFO] Calibration complete! Sensor is now active.
```
4. Open your Android app on your phone
5. Wait for calibration to finish, then **wave your hand** at the sensor
6. You should see in Serial Monitor:
```
[INFO] 🚨 Motion detected at 45 seconds
[INFO] Alert published!
```
7. Your Android app fires the 🚨 notification!

---

## 🔧 Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| No COM port showing | Missing driver | Install CP2102 driver (link in Step 4) |
| WiFi not connecting | Wrong credentials | Double-check ssid/password in code |
| Sensor never triggers | Bad wiring | Check OUT wire goes to D3 |
| Sensor triggers constantly | Still calibrating | Wait full 30 seconds after power on |
| App gets no notification | Topic mismatch | Make sure topic is identical in both ESP8266 code and Android app |
| "Done uploading" never appears | Wrong board/port | Recheck Tools → Board and Tools → Port |