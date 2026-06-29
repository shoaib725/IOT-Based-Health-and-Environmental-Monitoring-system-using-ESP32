# 🩺 IoT-Based Integrated Health and Environmental Monitoring System (ESP32)

**An Offline-First Health & Environmental Monitoring Node — Firmware v2.8**

A portable, low-cost (< INR 3,500) ESP32 device that monitors heart rate, SpO₂, single-lead ECG, body temperature, ambient temperature/humidity, gas levels, ambient light, and GPS location — and keeps working even when the internet doesn't. Local alerts (buzzer + OLED) fire in under 100 ms regardless of network status; when Wi-Fi is available, everything also streams to a Blynk 2.0 mobile dashboard.

> B.Tech Capstone Project — Dept. of Electronics & Communication Engineering, Seshadri Rao Gudlavalleru Engineering College (JNTUK), 2025–2026.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Hardware Components & Pin Mapping](#hardware-components--pin-mapping)
- [System Architecture](#system-architecture)
- [Software & Libraries](#software--libraries)
- [ECG Signal Processing (250 Hz Bandpass Filter)](#ecg-signal-processing-250-hz-bandpass-filter)
- [Alert & Safety Logic](#alert--safety-logic)
- [Blynk Virtual Pin Mapping](#blynk-virtual-pin-mapping)
- [OLED Display Pages](#oled-display-pages)
- [Operational Modes](#operational-modes)
- [Getting Started](#getting-started)
- [Results Summary](#results-summary)
- [Limitations](#limitations)
- [Future Work](#future-work)
- [Bill of Materials](#bill-of-materials)
- [Authors](#authors)
- [References](#references)

---

## Overview

Existing low-cost patient monitors either depend entirely on the cloud for alerting (introducing 1.5–5 s latency that's unacceptable for a gas leak or cardiac event) or skip environmental sensing altogether. This project addresses that gap with an **offline-first** architecture: every safety-critical function — sensor acquisition, OLED display, and alert generation — runs entirely on the ESP32 with **no network dependency**, while cloud sync to Blynk happens opportunistically whenever Wi-Fi is available.

Nine sensors are fused into one unit, spanning both **biomedical** (heart rate, SpO₂, ECG, body temperature) and **environmental** (ambient temp/humidity, gas, light, GPS) domains, so a caregiver can correlate a patient's vitals with what's happening in the room around them.

---

## Key Features

- **Nine-sensor integration** on a single ESP32-WROOM-32: MAX30105 (HR/SpO₂), AD8232 (ECG), DS18B20 (body temp), DHT11 (ambient temp/humidity), MQ-2 (gas), LDR (light), NEO-6M (GPS), TTP223 (panic button), SSD1306 (OLED).
- **Offline-first firmware** — local alerts fire within ~90 ms regardless of Wi-Fi/cloud status.
- **Medical-grade ECG** — 250 Hz sampling with a 3-stage digital bandpass filter resolving the P-QRS-T complex.
- **Blynk 2.0 cloud integration** over MQTT (TLS, port 443) across 15 virtual pin datastreams.
- **Smart LED** with dual control: automatic LDR-based ambient lighting or manual Blynk override.
- **Automatic Wi-Fi/Blynk reconnection** every 30 s, with zero interruption to local operation in between.
- **Three-page auto-rotating OLED**: Vital Signs (30 s) → Environment (3 s) → GPS/System (3 s).
- **Dual-trigger alerting**: gas threshold crossing (5 beeps) or manual panic button press (3 beeps), with a 30 s cooldown to avoid alarm fatigue.
- **Total BOM cost under INR 3,500** (~USD 42) — roughly 70–99% cheaper than comparable commercial or hospital-grade monitors.

---

## Hardware Components & Pin Mapping

| Component | Interface | GPIO / Pin | Specification | Function |
|---|---|---|---|---|
| ESP32-WROOM-32 | Native | All | 240 MHz, 12-bit ADC | Main MCU |
| MAX30105 | I2C (`0x57`) | SDA 21, SCL 22 | 18-bit ADC | Heart rate & SpO₂ |
| AD8232 | Analogue | ECG: 34, Ctrl: 15 | 0.5–40 Hz bandpass | ECG front-end |
| DS18B20 | 1-Wire | GPIO 25 | ±0.5 °C, 12-bit | Body temperature |
| DHT11 | Digital | GPIO 27 | ±5% RH, ±2 °C | Ambient temp/humidity |
| MQ-2 | ADC | GPIO 35 | 200–5000 ppm | Gas detection |
| LDR (10 kΩ) | ADC | GPIO 33 | 0–4095 ADC range | Ambient light |
| NEO-6M GPS | UART2 | RX 16, TX 17 | 9600 baud, NMEA | Location |
| TTP223 | Digital IN | GPIO 4 | Active HIGH | Panic button |
| SSD1306 OLED | I2C (`0x3C`) | SDA 21, SCL 22 | 128×64 px | Local display |
| Active Buzzer | Digital OUT | GPIO 26 | 85 dB @ 1 m | Audible alert |
| Smart LED | Digital OUT | GPIO 32 | 3 mm red LED | Visual alert/status |

> Note: MAX30105 and SSD1306 share the I2C bus (SDA 21 / SCL 22 @ 400 kHz fast mode).

---

## System Architecture

The firmware is organized into **three concurrency layers**, running cooperatively (no RTOS):

| Layer | Frequency | Responsibility |
|---|---|---|
| **Fast Layer** | 250 Hz | ECG sampling + 3-stage bandpass filtering |
| **Medium Layer** | ~0.67 Hz (1.5 s loop) | All other sensor reads, safety evaluation, OLED update, Blynk transmission |
| **Slow Layer** | 0.033 Hz (every 30 s) | Wi-Fi / Blynk reconnection via `checkConnectivity()` |

`checkSafetyTriggers()` is deliberately placed **after** `updateOLED()` and **before** `sendToBlynk()`, so local alert activation is never delayed by MQTT transmission.

**Data flow** is dual-path:
- **Local path** (always active): acquire → process → OLED → safety check, fully self-contained.
- **Cloud path** (when connected): same data is also pushed to Blynk via `Blynk.virtualWrite()` on 15 virtual pins.
- **ECG path** (high-speed, special case): when GPIO 15 goes LOW (electrodes attached), a dedicated 250 Hz loop preempts the main loop, filters the signal, and streams it to Blynk V12 at 10 Hz while also printing to Serial for the Arduino IDE Serial Plotter.

**Communication protocols**: I2C @ 400 kHz (MAX30105, SSD1306), UART2 @ 9600 baud 8-N-1 (GPS), 1-Wire @ 5 kHz (DS18B20), and MQTT-over-TLS on port 443 to `blynk.cloud` for cloud sync.

---

## Software & Libraries

Developed in **Arduino IDE 2.3.2** with the **Arduino-ESP32 core 2.0.14** (board: "ESP32 Dev Module", 240 MHz, no PSRAM, default partition scheme).

| Library | Purpose |
|---|---|
| `Wire.h` | I2C master for MAX30105 and SSD1306 |
| `WiFi.h` | ESP32 Wi-Fi station mode |
| `BlynkSimpleEsp32.h` | Blynk 2.0 MQTT client, `virtualWrite()` / `BLYNK_WRITE()` |
| `TinyGPSPlus.h` | NMEA sentence parser for NEO-6M |
| `Adafruit_GFX.h` / `Adafruit_SSD1306.h` | Graphics + OLED driver |
| `MAX30105.h` / `heartRate.h` / `spo2_algorithm.h` | SparkFun driver + Maxim SpO₂/HR algorithm |
| `DHT.h` | DHT11 bit-bang driver |
| `OneWire.h` / `DallasTemperature.h` | 1-Wire bus + DS18B20 driver |

**Firmware execution sequence:**
1. `setup()` — initializes all hardware, attempts Wi-Fi/Blynk connection (non-blocking).
2. `loop()` (every 1.5 s) — `checkConnectivity()` → GPS decode → MAX30105 sampling → temperature reads → environmental reads → `updateLED()` → ECG check → `updateOLED()` → `checkSafetyTriggers()` → `sendToBlynk()`.
3. If GPIO 15 is LOW, a dedicated 250 Hz ECG loop preempts the main loop until the electrode is disconnected.

---

## ECG Signal Processing (250 Hz Bandpass Filter)

Implemented in `readECGFiltered()`, applied to every 12-bit ADC sample at 250 Hz:

**Stage 1 — Moving Average (FIR low-pass, ~12.5 Hz cutoff)**
```cpp
int sum = 0;
for (int i = 0; i < FILTER_SIZE; i++) sum += ecgBuffer[i];
float filtered = sum / (float)FILTER_SIZE;
```

**Stage 2 — IIR High-Pass (removes baseline wander, fc ≈ 0.32 Hz, `HP_ALPHA = 0.996`)**
```cpp
filterHP = HP_ALPHA * filterHP + HP_ALPHA * (filtered - ecgBaseline);
ecgBaseline = filtered;
```

**Stage 3 — IIR Low-Pass (removes muscle noise, fc ≈ 4.4 Hz, `LP_ALPHA = 0.1`)**
```cpp
filterLP = LP_ALPHA * filterHP + (1.0 - LP_ALPHA) * filterLP;
```

The cascaded HP–LP pair forms a ~0.32–4.4 Hz bandpass, covering the fundamental frequency range of ECG waveforms for resting heart rates of 60–100 BPM, and is sufficient to resolve the P-QRS-T complex on an unshielded breadboard.

---

## Alert & Safety Logic

Two independent triggers, evaluated every main-loop iteration in `checkSafetyTriggers()`:

| Trigger | Condition | Response | Cooldown |
|---|---|---|---|
| **Gas hazard** | `gasValue > GAS_MAX_LIMIT (200)` (~950 ppm LPG equiv.) | 5-beep buzzer sequence (150 ms ON / 100 ms OFF) + LED flash + `vital_alert` push | 30 s |
| **Panic button** | TTP223 touch on GPIO 4 → HIGH | 3-beep buzzer sequence (750 ms total) + `vital_alert` push | 30 s |

```cpp
void checkSafetyTriggers() {
  if (millis() - lastAlertTime < ALERT_COOLDOWN) return; // 30 s
  String alertMsg = ""; int beepCount = 0;

  if (gasValue > GAS_MAX_LIMIT) {
    alertMsg = "CRITICAL: High Gas (" + String(gasValue) + ")";
    beepCount = 5;
  } else if (touchValue == HIGH) {
    alertMsg = "ALERT: Emergency Button Pressed!";
    beepCount = 3;
  }

  if (beepCount > 0) {
    for (int i = 0; i < beepCount; i++) {
      digitalWrite(BUZZER_PIN, HIGH); delay(200);
      digitalWrite(BUZZER_PIN, LOW);  delay(200);
    }
    if (blynkConnected) Blynk.logEvent("health_alert", alertMsg);
    lastAlertTime = millis();
  }
}
```

Measured local alert latency: **88–94 ms**, including in offline mode (Wi-Fi disabled) — confirming local alerting is independent of network status. The TTP223 capacitive button replaced an MPU6050 accelerometer used in firmware v2.7 for fall detection, which produced 47 false alarms in 72 hours of testing from normal movement; the capacitive button eliminated false alarms entirely.

---

## Blynk Virtual Pin Mapping

| V-Pin | Data Source | Unit | Type | Update Rate |
|---|---|---|---|---|
| V0 | Heart Rate – MAX30105 | BPM | Sensor | 1.5 s |
| V1 | SpO₂ – MAX30105 | % | Sensor | 1.5 s |
| V2 | Body Temperature – DS18B20 | °C | Sensor | 1.5 s |
| V3 | Ambient Temperature – DHT11 | °C | Sensor | 1.5 s |
| V4 | Humidity – DHT11 | % | Sensor | 1.5 s |
| V5 | Gas Level – MQ-2 | ADC | Sensor | 1.5 s |
| V6 | Light Level – LDR | ADC | Sensor | 1.5 s |
| V8 | Touch Sensor – TTP223 | 0/1 | Sensor | 1.5 s |
| V9 | GPS Latitude – NEO-6M | Degrees | Sensor | 1.5 s |
| V10 | Smart LED State/Control | 0/1 | Control | On event |
| V11 | Remote Buzzer Test | — | Control | On press |
| V12 | ECG Waveform – AD8232 | ADC | Sensor | 10 Hz (ECG mode) |
| V13 | System Uptime | s | System | 1.5 s |
| V14 | Wi-Fi RSSI | dBm | System | 1.5 s |
| V15 | GPS Longitude – NEO-6M | Degrees | Sensor | 1.5 s |

`BLYNK_WRITE(V10)` toggles the LED override (manual ON forces the LED, clearing it returns control to automatic LDR mode). `BLYNK_WRITE(V11)` fires the buzzer for 100 ms as a remote test.

---

## OLED Display Pages

| Page | Dwell Time | Content |
|---|---|---|
| **Page 0 – Vital Signs** | 30 s | Heart rate, SpO₂, body temp (DS18B20), ambient temp (DHT11), humidity. Shows "Place Finger..." if no finger detected on MAX30105. |
| **Page 1 – Environment** | 3 s | Gas concentration, light level, LED mode (AUTO/BLYNK), touch state. Shows a large "!! GAS DANGER !!" warning above threshold. |
| **Page 2 – GPS/System** | 3 s | GPS latitude/longitude, satellite count, system uptime, Wi-Fi RSSI. |

Every page carries a persistent status bar (`W:OK/X`, `B:OK/X`, `P:n`) showing Wi-Fi/Blynk connectivity and current page — this updates within 30 s of a connectivity change but never blocks sensor display.

---

## Operational Modes

**Online mode** — persistent MQTT connection to `blynk.cloud`. All 15 virtual pins update every 1.5 s; remote dashboard, push notifications, GPS map widget, and remote LED/buzzer control are all active.

**Offline mode** (no network dependency) — the firmware proceeds to local operation immediately with no blocking timeout. Continues: full 3-page OLED display, sub-100 ms local alerts, full ECG recording + Serial Plotter output, and all cooldown/threshold logic. Only cloud transmission, push notifications, and the GPS map widget are unavailable.

**Automatic reconnection** — `checkConnectivity()` runs every main-loop iteration and retries Wi-Fi/Blynk every 30 s if disconnected; a failed attempt completes in ~10 s without blocking local operation. During a 72-hour uptime test, two Blynk disconnections (caused by router DHCP lease renewal) were self-healed within 30 s with no manual intervention.

---

## Getting Started

1. Install **Arduino IDE 2.3.2+** and add the **ESP32 board package** (Arduino-ESP32 core 2.0.14) via Boards Manager.
2. Install the libraries listed in [Software & Libraries](#software--libraries) via the Library Manager.
3. Wire all sensors per the [pin mapping table](#hardware-components--pin-mapping).
4. Create a free **Blynk 2.0** account, set up a new device/template, and note your auth token and the 15 virtual pin datastreams (table above).
5. In the sketch, set your Wi-Fi SSID/password and Blynk auth token.
6. Select board **ESP32 Dev Module**, 240 MHz CPU frequency, default partition scheme, then upload.
7. Open the Serial Monitor (9600 baud) or Serial Plotter to view debug output and ECG waveform.

---

## Results Summary

| Parameter | Reference | Measured |
|---|---|---|
| Heart Rate (MAX30105) | Hospital pulse oximeter — 72 BPM | 73 BPM (±1 BPM) |
| SpO₂ (MAX30105) | Hospital pulse oximeter — 98% | 97% (−1%) |
| Body Temperature (DS18B20) | IR thermometer — 36.8 °C | 36.6 °C (−0.2 °C) |
| Ambient Temp (DHT11) | Calibrated thermometer — 25.4 °C | 26.1 °C (+0.7 °C) |
| Humidity (DHT11) | Calibrated hygrometer — 62% RH | 65% RH (+3%) |
| Gas threshold | Gas analyser ≥1000 ppm LPG | ADC > 200 ≈ 950 ppm |
| Gas alert latency | — | 88–94 ms (91 ms offline) |
| Wi-Fi reconnection | 30 s interval | Self-heals within 30 s |
| Uptime | 72 h continuous | 0 crashes |
| Power (online / offline / ECG) | 5 V supply | 2.1 W / 1.55 W / 2.2 W |
| Battery life (3,000 mAh, 85% eff.) | — | 6.1 h / 8.3 h / 5.8 h |
| Flash / RAM usage | 2 MB flash / 328 KB RAM | 55.9% / 16.6% |

---

## Limitations

- Single-lead ECG (Lead I) supports heart-rate extraction but **cannot support arrhythmia classification**, which requires multi-lead clinical interpretation — this is a monitoring aid, not a diagnostic instrument.
- The MQ-2 detects combustible gases generically and **cannot distinguish gas type** (e.g., cooking near the sensor can trigger an alert indistinguishable from a genuine leak).
- DHT11 and LDR readings were displayed continuously but **not independently bench-validated** against separate reference instruments.
- Accuracy figures were gathered from a small cohort of healthy young adults, not the target elderly/comorbid population — wider real-world error margins should be expected.
- Continuous current draw (~310–445 mA) limits practical battery-portable runtime until Wi-Fi duty-cycling is implemented.

---

## Future Work

- **Hardware**: custom 4-layer PCB, 3,000 mAh LiPo + TP4056 charging circuit, wearable enclosure; pulse-transit-time (PTT) cuffless blood pressure estimation; upgrade to a 2.4" ILI9341 colour TFT for on-device ECG waveform display.
- **Software**: FreeRTOS task separation (ECG on Core 1 real-time, sensors/display on Core 0) to reduce jitter; on-device ML anomaly detection via TensorFlow Lite for Microcontrollers / Edge Impulse.
- **Connectivity**: GSM/LTE (SIM800L/SIM7600) for coverage without fixed Wi-Fi; Bluetooth LE credential provisioning to remove the need to reflash firmware for Wi-Fi changes; integration with India's ABDM (Ayushman Bharat Digital Mission) health data exchange.
- **Validation**: a pilot study with elderly participants is identified as the critical missing step before any real-world performance claims can be made.

---

## Bill of Materials

**Total cost: under INR 3,500 (≈ USD 42)** — a 70–80% reduction versus commercial low-cost IoT health monitors (INR 5,000–15,000) and >99% versus hospital-grade multi-parameter monitors (INR 5–10 lakh).

---

## Authors

| Name | Roll No. |
|---|---|
| N. Dharmesh | 23485A0422 |
| P.M. Shoaib Khan | 22481A04I8 |
| N. Sri Venkata Satish Babu | 22481A04H5 |
| M. Syam Ganga Raju | 22481A04E6 |

**Guide:** Dr. P. Sai Vinay Kumar, Assistant Professor, Dept. of ECE
**Institution:** Seshadri Rao Gudlavalleru Engineering College (Autonomous, affiliated to JNTUK), Gudlavalleru, Andhra Pradesh — 2025–2026

This work was also published as: *"IoT-Based Integrated Health and Environmental Monitoring System using ESP32 Microcontroller,"* International Journal of Engineering Research & Technology (IJERT), Vol. 15, Issue 03, March 2026, ISSN: 2278-0181. Licensed under CC BY 4.0.

---

## References

Key datasheets and references used in this project: ESP32 Series Datasheet (Espressif), MAX30105 Datasheet (Maxim Integrated), AD8232 Datasheet (Analog Devices), DS18B20 Datasheet (Maxim Integrated), DHT11 Datasheet (AOSONG), MQ-2 Datasheet (Hanwei Electronics), NEO-6 Datasheet (u-blox), SSD1306 Datasheet (Solomon Systech), and the Blynk 2.0 documentation (docs.blynk.io). Full academic citation list is available in the project report.
