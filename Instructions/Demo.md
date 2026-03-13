# 🎯 Demo Setup Guide — Hardware & Arduino
### IoT Intrusion Detection System
**For college demo — standalone, no laptop needed after setup**

---

## 🧰 What to Bring to the Demo

| Item | Why You Need It |
|------|----------------|
| NodeMCU ESP8266 | The brain of the system |
| HC-SR501 Motion Sensor | Detects motion |
| Breadboard | Holds everything together |
| Jumper Cables (x3) | Connects sensor to board |
| Power Bank (5V) | Powers NodeMCU without a laptop |
| Micro-USB Cable | Connects NodeMCU to power bank |
| Your Android Phone | Shows the alert app |

> 💡 **Pro tip:** A power bank makes your demo completely wireless and portable.
> No laptop, no cables on the table — looks much more professional.

---

## ⚙️ PART 1 — Prepare Arduino Code Before the Demo Day

Do this at home **before** you go to the demo.

### Step 1 — Decide Your WiFi Strategy

You have two options. **Option B is strongly recommended.**

---

**Option A — Use College WiFi**

Get the college WiFi name and password in advance, then update your Arduino code:

```cpp
const char* ssid     = "COLLEGE_WIFI_NAME";
const char* password = "COLLEGE_WIFI_PASSWORD";
```

⚠️ Risk: College WiFi may block MQTT port 1883. You won't know until you're there.

---

**Option B — Use Your Phone as a Hotspot ✅ Recommended**

This makes your demo 100% independent — no venue WiFi needed.

1. On your phone go to **Settings → Hotspot & Tethering → Mobile Hotspot**
2. Set a simple hotspot name and password you'll remember, for example:
   - **Name:** `DemoHotspot`
   - **Password:** `demo1234`
3. Update your Arduino code to match:

```cpp
const char* ssid     = "DemoHotspot";
const char* password = "demo1234";
```

4. Re-upload this code to your NodeMCU at home
5. Test it works with hotspot before demo day

> ✅ With this setup your NodeMCU connects to your phone's hotspot,
> then reaches the MQTT broker over your phone's mobile data.
> Everything works in one self-contained unit.

---

### Step 2 — Re-upload the Code

1. Open Arduino IDE on your PC
2. Plug NodeMCU into your PC via USB
3. Open your `intrusion_detection.ino` sketch
4. Update WiFi credentials as above
5. Go to **Tools → Board → NodeMCU 1.0 (ESP-12E Module)**
6. Go to **Tools → Port** → select your COM port
7. Press **Ctrl + U** to upload
8. Wait for **"Done uploading"**

---

### Step 3 — Verify It Works at Home First

1. Turn on your phone hotspot
2. Open Serial Monitor in Arduino IDE (**Tools → Serial Monitor**, baud rate **9600**)
3. You should see:
```
[INFO] Connecting to WiFi: DemoHotspot
....
[INFO] WiFi Connected!
[INFO] IP Address: 192.168.x.x
[INFO] Connecting to MQTT Broker... Connected!
[INFO] Calibrating sensor for 30 seconds...
..............................
[INFO] Calibration complete! Sensor is now active.
```
4. Open the app on your phone
5. Wave at the sensor — confirm 🚨 alert fires and notification appears
6. Confirm screen resets to ✅ All Clear after 5 seconds

✅ **Only go to the demo if this works perfectly at home first.**

---

## 🔌 PART 2 — Wire the Circuit

**Always wire with the NodeMCU unplugged from power.**

### Wiring Table

| HC-SR501 Pin | NodeMCU Pin | Wire Color |
|---|---|---|
| VCC | 3V3 | Red |
| OUT | D3 | Yellow |
| GND | GND | Black |

### Wiring Diagram

```
HC-SR501              NodeMCU ESP8266
─────────             ───────────────
  VCC  ────red─────▶  3V3
  OUT  ───yellow───▶  D3
  GND  ───black───▶   GND
                      │
                      └── Micro-USB ──▶ Power Bank
```

> ⚠️ Never use VIN or 5V on the NodeMCU — always use 3V3 for the sensor.

---

## 🚀 PART 3 — On Demo Day, Step by Step

Follow this exact order every time.

---

### Step 1 — Turn On Phone Hotspot First
- Go to **Settings → Hotspot → Turn On**
- Do this **before** powering the NodeMCU
- The NodeMCU needs the hotspot to already be available when it boots

---

### Step 2 — Power On the NodeMCU
- Plug the Micro-USB cable from NodeMCU into your power bank
- The blue LED on the NodeMCU will blink — this means it is booting

---

### Step 3 — Wait for Calibration (30 seconds)
- The HC-SR501 needs 30 seconds to stabilise
- **Do not wave at the sensor during this time**
- Use this time to open the app and show the audience the "● Connecting..." status

---

### Step 4 — Open the App
- Open the **Intrusion Detection** app on your phone
- Wait until it shows **"● Connected to MQTT"** in green
- This confirms the full chain is working:
```
Sensor → NodeMCU → Hotspot → Internet → MQTT Broker → App
```

---

### Step 5 — Trigger the Demo
- Wave your hand slowly in front of the white dome of the HC-SR501
- Within 1-2 seconds the app will show:
  - 🚨 **INTRUSION DETECTED** in red
  - Timestamp of detection
  - Push notification popup
  - Phone vibration
- After 5 seconds it resets to 🏠 **All Clear** in green

---

## 🔁 Resetting Between Demo Runs

If you want to trigger it again for another audience:

1. Simply wave your hand at the sensor again after the 5 second reset
2. No need to restart anything
3. The system is always listening — just keep waving to demo repeatedly

---

## 🔧 Quick Fixes If Something Goes Wrong at the Demo

| Problem | Quick Fix |
|---------|-----------|
| App shows "● Disconnected" | Toggle phone hotspot off and on, wait 10 seconds |
| NodeMCU not connecting | Unplug power bank, plug back in, wait 30 seconds |
| No notification on phone | Check Settings → Apps → Intrusion Detection → Notifications → Allow |
| Sensor not triggering | Move hand slowly — wave 20-30cm in front of the dome |
| App installed but won't open | Uninstall and reinstall the APK |
| Everything looks connected but no alert | Confirm MQTT topic is identical in both Arduino code and Android app |

---

## 📋 Pre-Demo Checklist (Night Before)

```
□ Arduino code re-uploaded with hotspot WiFi credentials
□ Tested full demo run at home — alert fires correctly
□ APK installed on phone as standalone app
□ Notifications allowed for the app
□ Power bank is fully charged
□ All wires are firmly connected on breadboard
□ Brought all components listed in the table above
□ Phone mobile data is active (for hotspot to reach MQTT broker)
```

---

## 🎤 Suggested Demo Script

> *"This is a real-time IoT intrusion detection system. The NodeMCU ESP8266
> microcontroller reads data from the HC-SR501 motion sensor and publishes
> an alert to an MQTT broker over WiFi. The Android app subscribes to that
> broker and receives the alert in real time — anywhere in the world.
> Watch what happens when I wave my hand..."*

*[wave at sensor — alert fires on phone]*

> *"The app shows the intrusion alert instantly with a timestamp and push
> notification. After 5 seconds it resets automatically — ready for the
> next detection. The whole system runs wirelessly off a power bank."*

---

*IoT Intrusion Detection System — College Demo Guide*