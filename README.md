# AI-Assisted Early Osteoarthritis Monitoring — Build Guide

A wearable prototype that uses two IMUs, a piezo sensor, and an FSR on an
ESP32 to estimate knee range of motion, detect crepitus (joint
clicking/grinding), and track loading pattern during movement — then shows a
live dashboard with a heuristic risk score.

> **Disclaimer:** This is a DIY engineering prototype for learning, research,
> or a hackathon demo. It is **not** a certified medical device, and the risk
> score uses simple fixed heuristics, not a clinically validated algorithm.
> Don't use it for real diagnosis or treatment decisions — see Part 6 for how
> to evolve it into something more rigorous.

---

## Part 1 — Parts list

| Qty | Part |
|---|---|
| 1 | ESP32 DevKit (any ESP32-WROOM board) |
| 2 | MPU6050 breakout boards (accelerometer + gyroscope) |
| 1 | Piezo disc sensor (the passive kind, like a small buzzer element) |
| 1 | FSR (Force Sensitive Resistor), e.g. FSR402 |
| 1 | 10 kΩ resistor (FSR voltage divider) |
| 1 | 100 kΩ resistor (piezo series/protection resistor) |
| 1 | 1 MΩ resistor (piezo bleed resistor) |
| 2 | 1N4148 signal diodes (piezo ADC clamp/protection) |
| — | Breadboard, jumper wires, elastic strap/velcro to mount sensors on the leg |

---

## Part 2 — Circuit wiring

### 2a. Both MPU6050 boards share one I2C bus

| MPU6050 pin | Connect to |
|---|---|
| VCC | ESP32 3.3V |
| GND | ESP32 GND |
| SCL | ESP32 GPIO 22 (both boards, same wire) |
| SDA | ESP32 GPIO 21 (both boards, same wire) |
| AD0 (Board 1 — "thigh") | ESP32 GND → I2C address **0x68** |
| AD0 (Board 2 — "shank") | ESP32 3.3V → I2C address **0x69** |

Two devices can share an I2C bus as long as they have different addresses —
that's exactly what the AD0 pin trick above gives you.

### 2b. FSR (foot loading)

Wire it as a voltage divider so the ADC reads a voltage that changes with
pressure:

```
3.3V ---[ FSR ]---+---[ 10kΩ ]--- GND
                   |
                 ESP32 GPIO 34 (ADC)
```

No pressure → high resistance → the ADC reads close to 0.
Pressure → resistance drops → the ADC reading rises.

### 2c. Piezo disc (crepitus / vibration pickup)

Piezo elements can generate voltage spikes outside 0–3.3V when knocked, which
can damage the ESP32's ADC pin, so add simple protection:

```
Piezo (+) ----+----[100kΩ]---- ESP32 GPIO 35 (ADC)
              |                        |
           [1MΩ]                  [1N4148]----3.3V   (clamps high spikes)
              |                        |
Piezo (–) ----+----GND            [1N4148]----GND     (clamps negative spikes)
```

- The 1 MΩ resistor across the piezo bleeds off static charge so readings
  settle instead of drifting.
- The two diodes clamp any spike above 3.3V or below 0V, protecting the ESP32.
- Tape the piezo disc flat against the skin near the knee joint line (patella
  border) with medical tape — that's typically where clicking/crepitus
  vibration is most detectable.

### 2d. Sensor placement on the body

- **MPU6050 #1 ("thigh")**: strapped to the front of the thigh, above the knee.
- **MPU6050 #2 ("shank")**: strapped to the front of the shin, below the knee.
- **Piezo**: taped near the knee joint line.
- **FSR**: placed under the heel or forefoot inside a shoe/insole to capture
  loading during standing or walking.

---

## Part 3 — Software setup

1. Install the **Arduino IDE** (2.x).
2. In **File → Preferences**, add this Additional Board Manager URL:
   `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
3. In **Tools → Board → Boards Manager**, search "esp32" and install the
   Espressif package.
4. In **Tools → Manage Libraries**, install:
   - **Adafruit MPU6050**
   - **Adafruit Unified Sensor** (installs automatically as a dependency)
   - **ArduinoJson** (version 6.x)
5. Open `osteo_prototype.ino`.
6. Edit these two lines near the top with your Wi-Fi details:
   ```cpp
   const char* WIFI_SSID     = "YOUR_WIFI_NAME";
   const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";
   ```
7. Select your board under **Tools → Board → ESP32 Arduino → (your board)**
   and the correct COM/serial port.
8. Click **Upload**.

---

## Part 4 — First run

1. Open **Tools → Serial Monitor** at baud rate `115200`.
2. Wait for it to connect to Wi-Fi; it will print something like:
   ```
   Connected! Open this in your browser: http://192.168.1.42
   ```
3. Open that IP address in a browser on any device on the same Wi-Fi network
   (phone or laptop). You'll see the live dashboard — knee angle chart, foot
   load chart, crepitus counter, and the risk report, updating twice a second.
4. Click **Reset Session** any time you want to start a fresh
   ROM/crepitus/loading measurement window.

---

## Part 5 — Calibration (do this before trusting the numbers)

The raw sensor values vary a lot by hardware batch, mounting, and body/shoe,
so a couple of quick calibration passes will make results much more sensible:

1. **Piezo threshold** — with the piezo taped on and the joint still, watch
   the raw ADC values (add a quick `Serial.println(raw)` in `samplePiezo()`
   temporarily). Note the typical "noise floor" range, then set
   `PIEZO_EVENT_THRESHOLD` comfortably above that noise floor so only real
   taps/clicks register, not ambient vibration.
2. **IMU axis check** — slowly bend the knee and watch `kneeAngle` in the
   Serial Monitor. If the number goes the wrong direction or barely changes,
   swap which accelerometer axis is used in `accAngleThigh`/`accAngleShank`
   (the code currently uses Y/Z — try X/Z or X/Y depending on how your boards
   are oriented on the strap).
3. **FSR baseline** — stand normally and check the "Foot Load" reading is
   mid-range, not pegged at 0 or 4095. Adjust the 10 kΩ divider resistor value
   up or down if it's saturating.
4. **ROM reference** — compare the displayed range-of-motion number against a
   physical goniometer (or even a phone angle app) for a few known bend
   angles, and adjust the complementary filter's `ALPHA` constant or the
   axis assumptions if the numbers don't line up.

---

## Part 6 — How the "risk score" works (and how to make it more rigorous)

Right now `computeRiskReport()` combines three simple 0–100 sub-scores with
fixed weights:

- **ROM risk** — how far session range-of-motion falls short of an assumed
  130° healthy benchmark.
- **Crepitus risk** — detected events per minute, scaled by a fixed factor.
- **Loading risk** — how variable/uneven the FSR signal is over time (a rough
  proxy for guarded/antalgic gait).

This is intentionally simple so the whole pipeline runs standalone on the
ESP32 with no internet dependency. To turn this into something closer to
genuine "AI-assisted" scoring:

1. **Log labeled sessions.** Have the device stream its raw + filtered
   features (ROM, crepitus rate, load variability, cadence, etc.) to a CSV,
   alongside a known label (e.g., a clinician's assessment or a self-reported
   symptom score) for many sessions/people.
2. **Train a model offline** (e.g., logistic regression, random forest, or a
   small neural net in scikit-learn/PyTorch) on those labeled feature
   vectors instead of hand-picked weights.
3. **Deploy the model** either:
   - server-side: have the ESP32 POST its feature snapshot to a small
     Flask/FastAPI server that runs the trained model and returns the score, or
   - on-device: convert the trained model to **TensorFlow Lite for
     Microcontrollers** and run inference directly on the ESP32.
4. Validate against a real clinical scale (e.g., WOMAC) before trusting the
   output for anything beyond a demo.

---

## Part 7 — Files in this project

- `osteo_prototype.ino` — the full ESP32 firmware. It reads all sensors,
  applies the filters, computes the risk report, and **also hosts the web
  dashboard itself** (no separate web server needed — just open the ESP32's
  IP address in a browser).
