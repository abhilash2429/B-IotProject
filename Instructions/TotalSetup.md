# 🔐 IoT Intrusion Detection System — Complete Beginner's Guide
### NodeMCU ESP8266 + HC-SR501 Motion Sensor + Android App
**Your Setup:** Windows PC | NodeMCU ESP8266 | Android Phone

---

## 🗺️ What You're Building

A motion-triggered intrusion alarm system with 3 parts:

```
[HC-SR501 Motion Sensor]
        ↓ detects motion
[NodeMCU ESP8266]  ──WiFi──→  [MQTT Broker (Cloud)]  ──→  [Android App]
   reads sensor &                 routes messages            shows alert
   publishes alert                                           notification
```

**Key difference from the book:** The book used Arduino UNO + a separate WiFi Shield.
Your NodeMCU ESP8266 does **both jobs in one board** — it has built-in WiFi. Simpler and cheaper!

---

## 📋 PHASE 1 — Install Your Software (One-Time Setup)

### Step 1.1 — Download Arduino IDE

1. Open your browser and go to: **https://www.arduino.cc/en/software**
2. Click **"Windows Win 10 and newer, 64 bits"**
3. Click **"Just Download"** (you don't need to donate)
4. Run the downloaded `.exe` file and install it (click Next → Next → Install)
5. Open Arduino IDE when done — it will look like a text editor with a toolbar at the top

---

### Step 1.2 — Add ESP8266 Board Support to Arduino IDE

Arduino IDE doesn't know about the ESP8266 by default. We need to teach it.

1. Open Arduino IDE
2. Go to **File → Preferences** (top menu)
3. Find the box that says **"Additional boards manager URLs"**
4. Paste this URL into that box:
   ```
   https://arduino.esp8266.com/stable/package_esp8266com_index.json
   ```
5. Click **OK**
6. Now go to **Tools → Board → Boards Manager**
7. In the search box, type: `esp8266`
8. Find **"esp8266 by ESP8266 Community"** and click **Install**
9. Wait for it to finish (it downloads ~150MB, so give it a minute)

---

### Step 1.3 — Install the MQTT Library

Your ESP8266 needs a library to talk to the MQTT broker.

1. In Arduino IDE, go to **Sketch → Include Library → Manage Libraries**
2. In the search box, type: `PubSubClient`
3. Find **"PubSubClient by Nick O'Leary"**
4. Click **Install**
5. Close the Library Manager

---

### Step 1.4 — Select Your Board

1. Plug your NodeMCU ESP8266 into your PC via USB cable
2. In Arduino IDE, go to **Tools → Board → esp8266 → NodeMCU 1.0 (ESP-12E Module)**
3. Go to **Tools → Port** and select the port that appeared (e.g., `COM3` or `COM4`)
   - ⚠️ If you don't see a port, install the CP2102 driver from: https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers
   - Download "CP210x Universal Windows Driver", unzip it, right-click the `.inf` file → Install

---

## 🔌 PHASE 2 — Wire Your Circuit

### Your Components
| Component | What it does |
|-----------|-------------|
| NodeMCU ESP8266 | The brain + WiFi |
| HC-SR501 | Detects motion (infrared) |
| Breadboard | Holds everything together |
| Jumper Cables | Connects components |

---

### Wiring Diagram

The HC-SR501 has **3 pins**. Look at your sensor — it will have 3 legs labeled:

```
HC-SR501 (viewed from the front dome side, pins facing you)

   [  dome  ]
   
  VCC  OUT  GND
   |    |    |
```

Connect them like this:

| HC-SR501 Pin | Connect to NodeMCU Pin |
|-------------|------------------------|
| VCC (Power) | **3V3** (3.3V pin on NodeMCU) |
| OUT (Signal)| **D3** (Digital pin 3) |
| GND (Ground)| **GND** (any GND pin on NodeMCU) |

> ⚠️ **IMPORTANT:** The book used 5V for the sensor, but NodeMCU runs on 3.3V logic.
> The HC-SR501 actually works fine on 3.3V for this project.
> Do NOT connect VCC to the VIN/5V pin — it could damage your board.

---

### NodeMCU Pin Reference (which pins to use)

```
NodeMCU Board Layout (top view):

Left Side Pins:          Right Side Pins:
─────────────            ─────────────
  A0                       RSV
  RSV                      RSV
  SD3                      SD1
  SD2                      CMD
  SD1                      SD0
  CMD                      CLK
  SD0                      GND  ← connect HC-SR501 GND here
  CLK                      3V3  ← connect HC-SR501 VCC here
  GND                      EN
  3V3                      RST
  EN                       GND
  RST                      VIN
  GND  
  VIN                       D0
                            D1
  D8                        D2
  D7                        D3  ← connect HC-SR501 OUT here
  D6                        D4
  D5                        3V3
  GND                       GND
  3V3                       D5
  D4                        D6
  D3                        D7
  D2                        D8
  D1                        RX
  D0                        TX
  RST                      GND
  GND                      3V3
```

> 💡 Look for the text printed on your NodeMCU board — the pin labels are printed right on it!

---

## 💻 PHASE 3 — Write & Upload the ESP8266 Code

### Step 3.1 — Open a New Sketch

In Arduino IDE, go to **File → New**. You'll get a blank sketch.

---

### Step 3.2 — Copy This Complete Code

**Delete everything** in the blank sketch, then paste this:

```cpp
// ============================================================
// IoT Intrusion Detection System
// NodeMCU ESP8266 + HC-SR501 Motion Sensor + MQTT
// ============================================================

#include <ESP8266WiFi.h>      // Built-in WiFi library for ESP8266
#include <PubSubClient.h>     // MQTT library

// ============================================================
// 🔧 CHANGE THESE VALUES TO MATCH YOUR SETUP
// ============================================================
const char* ssid     = "YOUR_WIFI_NAME";       // Your WiFi network name
const char* password = "YOUR_WIFI_PASSWORD";   // Your WiFi password
const char* mqttServer = "broker.hivemq.com";  // Free public MQTT broker
const int   mqttPort = 1883;
const char* mqttTopic = "myproject/intrusiondetection"; // Change this to something unique
const char* clientId  = "esp8266_intruder";    // Unique name for your device
// ============================================================

// Motion sensor settings
const int pirPin = D3;         // HC-SR501 signal pin connected to D3
const int calibrationTime = 30; // Seconds to calibrate sensor on startup

// State tracking variables
boolean lockLow = true;
boolean takeLowTime;
long unsigned int lowIn;
long unsigned int pause = 5000; // 5 seconds before resetting detection

// WiFi and MQTT clients
WiFiClient espClient;
PubSubClient mqttClient(espClient);

// ============================================================
// SETUP — runs once when powered on
// ============================================================
void setup() {
  Serial.begin(9600);         // Start serial monitor
  delay(100);

  // Connect to WiFi
  connectToWiFi();

  // Set up MQTT broker address
  mqttClient.setServer(mqttServer, mqttPort);

  // Calibrate motion sensor
  calibrateSensor();
}

// ============================================================
// LOOP — runs forever after setup
// ============================================================
void loop() {
  // Keep MQTT connection alive
  if (!mqttClient.connected()) {
    reconnectMQTT();
  }
  mqttClient.loop();

  // Read motion sensor
  readSensorData();
}

// ============================================================
// Connect to WiFi
// ============================================================
void connectToWiFi() {
  Serial.println("");
  Serial.print("[INFO] Connecting to WiFi: ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("[INFO] WiFi Connected!");
  Serial.print("[INFO] IP Address: ");
  Serial.println(WiFi.localIP());
}

// ============================================================
// Connect/Reconnect to MQTT broker
// ============================================================
void reconnectMQTT() {
  while (!mqttClient.connected()) {
    Serial.print("[INFO] Connecting to MQTT Broker...");

    if (mqttClient.connect(clientId)) {
      Serial.println(" Connected!");
    } else {
      Serial.print(" Failed. Error code: ");
      Serial.println(mqttClient.state());
      Serial.println("[INFO] Retrying in 5 seconds...");
      delay(5000);
    }
  }
}

// ============================================================
// Calibrate the HC-SR501 sensor (give it time to stabilize)
// ============================================================
void calibrateSensor() {
  pinMode(pirPin, INPUT);
  digitalWrite(pirPin, LOW);

  Serial.print("[INFO] Calibrating sensor for ");
  Serial.print(calibrationTime);
  Serial.println(" seconds. Please don't move...");

  for (int i = 0; i < calibrationTime; i++) {
    Serial.print(".");
    delay(1000);
  }

  Serial.println("");
  Serial.println("[INFO] Calibration complete! Sensor is now active.");
  delay(50);
}

// ============================================================
// Read motion sensor and publish if motion detected
// ============================================================
void readSensorData() {
  if (digitalRead(pirPin) == HIGH) {
    if (lockLow) {
      lockLow = false;
      Serial.print("[INFO] 🚨 Motion detected at ");
      Serial.print(millis() / 1000);
      Serial.println(" seconds since start");

      // Publish alert to MQTT broker
      publishAlert();
      delay(50);
    }
    takeLowTime = true;
  }

  if (digitalRead(pirPin) == LOW) {
    if (takeLowTime) {
      lowIn = millis();
      takeLowTime = false;
    }

    if (!lockLow && millis() - lowIn > pause) {
      lockLow = true;
      Serial.print("[INFO] Motion ended at ");
      Serial.print((millis() - pause) / 1000);
      Serial.println(" seconds since start");
    }
  }
}

// ============================================================
// Publish intrusion alert to MQTT broker
// ============================================================
void publishAlert() {
  if (mqttClient.connected()) {
    Serial.println("[INFO] Publishing alert to MQTT broker...");
    mqttClient.publish(mqttTopic, "Intrusion Detected");
    Serial.println("[INFO] Alert published!");
  } else {
    Serial.println("[ERROR] Not connected to MQTT broker, cannot publish.");
  }
}
```

---

### Step 3.3 — Edit Your WiFi Credentials

Find these two lines near the top and replace the text:

```cpp
const char* ssid     = "YOUR_WIFI_NAME";       // ← type your WiFi name here
const char* password = "YOUR_WIFI_PASSWORD";   // ← type your WiFi password here
```

Also **change the topic** to something unique (so you don't mix up with others):
```cpp
const char* mqttTopic = "myproject/intrusiondetection"; // ← change "myproject" to anything unique
```
Use the **exact same topic string** later in your Android app.

---

### Step 3.4 — Upload the Code

1. Go to **Sketch → Upload** (or press `Ctrl + U`)
2. You'll see a progress bar at the bottom — wait for "Done uploading"
3. If you get an error, make sure the correct **Port** is selected under **Tools → Port**

---

### Step 3.5 — Test with Serial Monitor

1. Go to **Tools → Serial Monitor**
2. Set baud rate to **9600** (bottom-right dropdown of the Serial Monitor window)
3. You should see:
   ```
   [INFO] Connecting to WiFi: YourWiFiName
   ....
   [INFO] WiFi Connected!
   [INFO] IP Address: 192.168.x.x
   [INFO] Connecting to MQTT Broker... Connected!
   [INFO] Calibrating sensor for 30 seconds. Please don't move...
   ..............................
   [INFO] Calibration complete! Sensor is now active.
   ```
4. After calibration, wave your hand in front of the sensor. You should see:
   ```
   [INFO] 🚨 Motion detected at 45 seconds since start
   [INFO] Publishing alert to MQTT broker...
   [INFO] Alert published!
   ```

✅ **If you see this — your hardware is working perfectly!**

---

## 📱 PHASE 4 — Build the Android App

### Step 4.1 — Install Android Studio

1. Go to: **https://developer.android.com/studio**
2. Click **Download Android Studio** and install it
3. On first launch, follow the Setup Wizard — click Next through all the defaults
4. It will download some SDK components (this takes a few minutes)

---

### Step 4.2 — Create a New Project

1. Open Android Studio
2. Click **New Project**
3. Select **"Empty Views Activity"** → click Next
4. Fill in:
   - **Name:** `Intrusion Detection`
   - **Package name:** `com.myapp.intrusiondetection`
   - **Language:** `Java`
   - **Minimum SDK:** API 21 (Android 5.0)
5. Click **Finish** — Android Studio generates your project files

---

### Step 4.3 — Add the MQTT Library

1. Open the file: `app/build.gradle` (the one inside the `app` folder, not the root one)
2. Find the `dependencies { }` block
3. Add this line inside it:
   ```groovy
   implementation 'org.eclipse.paho:org.eclipse.paho.client.mqttv3:1.2.5'
   implementation 'org.eclipse.paho:org.eclipse.paho.android.service:1.1.1'
   ```
4. Click **"Sync Now"** in the yellow bar that appears at the top
5. Also add to `dependencies`:
   ```groovy
   implementation 'androidx.localbroadcastmanager:localbroadcastmanager:1.1.0'
   ```

---

### Step 4.4 — Update AndroidManifest.xml

Open `app/manifests/AndroidManifest.xml` and add these lines:

Inside the `<application>` tag, add the MQTT service:
```xml
<service android:name="org.eclipse.paho.android.service.MqttService" />
```

Before the `</manifest>` closing tag, add permissions:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

---

### Step 4.5 — Design the Screen Layout

Open `res/layout/activity_main.xml`, switch to **Code** view and replace everything with:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center"
    android:padding="24dp"
    android:background="#F5F5F5">

    <TextView
        android:id="@+id/status_icon"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="🏠"
        android:textSize="80sp"
        android:layout_marginBottom="24dp"/>

    <TextView
        android:id="@+id/status_label"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="All Clear"
        android:textSize="28sp"
        android:textStyle="bold"
        android:textColor="#2E7D32"
        android:layout_marginBottom="16dp"/>

    <TextView
        android:id="@+id/last_detection"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="No activity detected yet"
        android:textSize="16sp"
        android:textColor="#555555"
        android:gravity="center"
        android:layout_marginBottom="32dp"/>

    <TextView
        android:id="@+id/connection_status"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="● Connecting..."
        android:textSize="14sp"
        android:textColor="#999999"/>

</LinearLayout>
```

---

### Step 4.6 — Write the Main Activity Code

Open `java/com/myapp/intrusiondetection/MainActivity.java` and replace everything with:

```java
package com.myapp.intrusiondetection; // make sure this matches your package name

import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.NotificationCompat;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.os.Build;
import android.os.Bundle;
import android.widget.TextView;
import android.graphics.Color;

import org.eclipse.paho.client.mqttv3.IMqttDeliveryToken;
import org.eclipse.paho.client.mqttv3.MqttCallback;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.MqttException;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;

import java.text.DateFormat;
import java.util.Date;

public class MainActivity extends AppCompatActivity {

    // ⚠️ Must match EXACTLY what you set in your ESP8266 code
    private static final String MQTT_BROKER   = "tcp://broker.hivemq.com:1883";
    private static final String MQTT_TOPIC    = "myproject/intrusiondetection"; // ← same as ESP8266
    private static final String CLIENT_ID     = "androidClient_" + System.currentTimeMillis();
    private static final String CHANNEL_ID    = "IntrusionAlerts";

    private TextView statusIcon, statusLabel, lastDetection, connectionStatus;
    private MqttClient mqttClient;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Get references to UI elements
        statusIcon       = findViewById(R.id.status_icon);
        statusLabel      = findViewById(R.id.status_label);
        lastDetection    = findViewById(R.id.last_detection);
        connectionStatus = findViewById(R.id.connection_status);

        // Create notification channel (required for Android 8+)
        createNotificationChannel();

        // Connect to MQTT in background thread
        new Thread(this::connectToMQTT).start();
    }

    private void connectToMQTT() {
        try {
            MqttConnectOptions options = new MqttConnectOptions();
            options.setCleanSession(true);
            options.setAutomaticReconnect(true);

            mqttClient = new MqttClient(MQTT_BROKER, CLIENT_ID, new MemoryPersistence());
            mqttClient.setCallback(new MqttCallback() {
                @Override
                public void connectionLost(Throwable cause) {
                    updateConnectionStatus("● Disconnected", "#F44336");
                }

                @Override
                public void messageArrived(String topic, MqttMessage message) {
                    // Motion detected! Update UI and show notification
                    String time = DateFormat.getDateTimeInstance().format(new Date());
                    String alertText = "Intrusion Detected @ " + time;
                    updateUI(alertText);
                    showNotification("🚨 Intrusion Alert!", alertText);
                }

                @Override
                public void deliveryComplete(IMqttDeliveryToken token) { }
            });

            mqttClient.connect(options);
            mqttClient.subscribe(MQTT_TOPIC, 0);
            updateConnectionStatus("● Connected to MQTT", "#4CAF50");

        } catch (MqttException e) {
            e.printStackTrace();
            updateConnectionStatus("● Connection Failed", "#F44336");
        }
    }

    private void updateUI(String message) {
        runOnUiThread(() -> {
            statusIcon.setText("🚨");
            statusLabel.setText("INTRUSION DETECTED");
            statusLabel.setTextColor(Color.parseColor("#C62828"));
            lastDetection.setText(message);
        });
    }

    private void updateConnectionStatus(String text, String colorHex) {
        runOnUiThread(() -> {
            connectionStatus.setText(text);
            connectionStatus.setTextColor(Color.parseColor(colorHex));
        });
    }

    private void createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(
                CHANNEL_ID, "Intrusion Alerts",
                NotificationManager.IMPORTANCE_HIGH
            );
            channel.setDescription("Alerts when motion is detected");
            NotificationManager manager = getSystemService(NotificationManager.class);
            manager.createNotificationChannel(channel);
        }
    }

    private void showNotification(String title, String message) {
        NotificationCompat.Builder builder = new NotificationCompat.Builder(this, CHANNEL_ID)
            .setSmallIcon(android.R.drawable.ic_dialog_alert)
            .setContentTitle(title)
            .setContentText(message)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setAutoCancel(true);

        NotificationManager manager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
        manager.notify(1, builder.build());
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        try {
            if (mqttClient != null && mqttClient.isConnected()) {
                mqttClient.disconnect();
            }
        } catch (MqttException e) {
            e.printStackTrace();
        }
    }
}
```

---

### Step 4.7 — Deploy to Your Android Phone

1. On your Android phone, go to **Settings → About Phone**
2. Tap **Build Number** 7 times — you'll see "Developer mode enabled"
3. Go back to **Settings → Developer Options**
4. Enable **USB Debugging**
5. Connect your phone to your PC with a USB cable
6. A dialog will appear on your phone — tap **"Allow"**
7. In Android Studio, click the **green Play button ▶** (or press `Shift + F10`)
8. Select your phone from the device list and click OK
9. The app will install and launch on your phone

---

## 🧪 PHASE 5 — Test the Complete System

### Testing Checklist

1. **Power on your NodeMCU** (plug it into USB or a power bank)
2. Open **Serial Monitor** in Arduino IDE to watch the logs
3. Wait 30 seconds for sensor calibration to complete
4. **Open the Android app** on your phone — it should show "● Connected to MQTT"
5. **Wave your hand** in front of the HC-SR501 sensor
6. Within 1-2 seconds you should see:
   - Serial Monitor: `[INFO] 🚨 Motion detected...` and `[INFO] Alert published!`
   - Android app: Screen changes to show 🚨 and "INTRUSION DETECTED"
   - Android notification: A popup notification appears

---

## 🔧 Troubleshooting Guide

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| No COM port showing | Missing driver | Install CP2102 driver (link in Step 1.4) |
| WiFi not connecting | Wrong credentials | Double-check ssid/password in code |
| Sensor never triggers | Bad wiring | Check OUT wire goes to D3, not 3V3 or GND |
| Sensor triggers constantly | Calibration still running | Wait 30 seconds after powering on |
| App says "Connection Failed" | Firewall or network | Make sure phone is on mobile data, not same WiFi |
| No notification on phone | Notifications disabled | Phone Settings → Apps → Your App → Allow Notifications |
| Build.gradle sync fails | Gradle version mismatch | Go to File → Invalidate Caches → Restart |

---

## 🎓 What You've Learned

By completing this project you now understand:

- **IoT pattern: Realtime Clients** — how sensor data flows from hardware to your phone
- **MQTT protocol** — a lightweight publish/subscribe messaging system used in all IoT
- **NodeMCU ESP8266** — a WiFi-capable microcontroller programmable in C++
- **Android development** — creating apps with layouts and background services
- **Circuit building** — connecting sensors to microcontrollers safely

---

## 🚀 Next Steps (When You're Ready)

- **Add an LED** that blinks when motion is detected (hardware feedback)
- **Add a buzzer** for an audible alarm
- **Log detections to a database** using a free service like Firebase
- **Send email alerts** by connecting to an email API
- **Add a camera** (ESP32-CAM board) to take photos when motion triggers

---

*Guide written for: Windows + NodeMCU ESP8266 + HC-SR501 + Android*
*MQTT Broker used: broker.hivemq.com (free, public, no account needed)*