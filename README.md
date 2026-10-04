# Gesture Controlled Application — Python + Arduino

A real-time **hand gesture controlled hardware interface** using Python, OpenCV, CVZone, and Arduino.

The system uses a webcam to detect hand gestures and converts them into **1 PWM channel + 4 digital control channels**. The commands are transmitted to an Arduino through a reliable serial communication protocol with heartbeat and watchdog protection.

---

## 🚀 Features

* Real-time hand gesture detection
* **1 PWM channel** with smooth control
* **4 digital ON/OFF channels**
* Fist gesture for system ON/OFF
* Pinch gesture for PWM control
* Finger gestures for digital channel control
* Gesture hold detection to prevent accidental switching
* Median + EMA smoothing for stable PWM output
* Serial communication at **115200 baud**
* Automatic serial reconnection
* Heartbeat monitoring
* Arduino watchdog safety system
* Real-time OpenCV HUD
* FPS monitoring
* Serial connection status display
* Demo mode without Arduino

---

## 🧠 Gesture Mapping

| Gesture             | Function                   |
| ------------------- | -------------------------- |
| ✊ Fist              | Hold to turn system ON/OFF |
| ☝️ 1 Finger + Pinch | Control CH1 PWM            |
| ✌️ 2 Fingers        | Toggle CH2                 |
| 3 Fingers           | Toggle CH3                 |
| 4 Fingers           | Toggle CH4                 |
| 5 Fingers           | Toggle CH5                 |

### System ON/OFF

A closed fist is detected and must be held continuously for the configured activation time.

Default:

```text
5 seconds
```

This prevents accidental system switching.

---

## 🎛️ Channel Architecture

```text
                 Webcam
                    │
                    ▼
          ┌───────────────────┐
          │  OpenCV + CVZone  │
          │ Hand Detection    │
          └─────────┬─────────┘
                    │
                    ▼
          ┌───────────────────┐
          │  Gesture Engine   │
          └─────────┬─────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
   CH1 PWM                 CH2–CH5 Digital
        │                       │
        └───────────┬───────────┘
                    ▼
             Serial Protocol
                    │
                    ▼
               Arduino
                    │
          ┌─────────┴─────────┐
          │                   │
        PWM              Digital Outputs
```

---

# 🔧 Hardware

### Main Components

* Arduino
* USB cable
* Webcam
* Relay modules / LED / motor controller / other loads
* Computer running Python

### Arduino Pin Configuration

| Channel    | Arduino Pin | Type    |
| ---------- | ----------: | ------- |
| CH1        |          D3 | PWM     |
| CH2        |          D4 | Digital |
| CH3        |          D5 | Digital |
| CH4        |          D6 | Digital |
| CH5        |          D7 | Digital |
| CH4 Mirror |          D9 | Digital |
| CH5 Mirror |         D10 | Digital |

> CH4 and CH5 mirror outputs are configured as **active-low** outputs.

---

# 💻 Software Requirements

* Python 3.10+
* OpenCV
* CVZone
* MediaPipe
* NumPy
* PySerial

Install the required packages:

```bash
pip install opencv-python cvzone mediapipe numpy pyserial
```

---

# 📁 Project Structure

```text
Gesture_Controlled_Application/
│
├── main.py
├── config.py
├── gesture_engine.py
├── serial_handler.py
├── ui_renderer.py
├── arduino/
│   └── gesture_controller.ino
│
└── README.md
```

---

# ⚙️ Configuration

All major settings are available in `config.py`.

### Camera

```python
CAMERA_ID = 0
FRAME_WIDTH = 1280
FRAME_HEIGHT = 720
FPS_TARGET = 30
```

### Serial

```python
SERIAL_PORT = "COM3"
BAUD_RATE = 115200
SERIAL_TIMEOUT = 0.5
```

For Linux:

```python
SERIAL_PORT = "/dev/ttyUSB0"
```

You can also specify the port from the command line:

```bash
python main.py --port COM5
```

---

# ✋ Gesture Detection

The application uses CVZone's hand tracking system.

```python
detector = HandDetector(
    detectionCon=0.75,
    maxHands=1
)
```

The detected hand landmarks are used for both:

* Finger counting
* Pinch-distance measurement

---

# 🎚️ CH1 PWM Control

CH1 uses the distance between the **thumb tip and index finger tip**.

```text
Thumb ●────────● Index
          ↓
     Pinch Distance
```

The distance is normalized and converted to a PWM value between:

```text
0 ─────────────── 255
```

The current implementation uses:

```python
min_d = 20
max_d = 200
```

The PWM signal is then smoothed using:

1. Median filtering
2. Exponential Moving Average
3. Deadband filtering

This significantly reduces jitter caused by hand-tracking noise.

---

# 🔘 Digital Channel Control

The digital channels use finger-count gestures.

```text
2 fingers → CH2
3 fingers → CH3
4 fingers → CH4
5 fingers → CH5
```

The gesture must be held for the configured duration.

Default:

```python
DIGITAL_TOGGLE_SECS = 1.0
```

After a successful toggle, the same gesture cannot immediately toggle the channel again until the hand gesture is released.

This prevents repeated ON/OFF switching.

---

# 🔌 Serial Communication

The Python application communicates with Arduino using a simple text-based protocol.

### System ON

```text
$SYS:ON
```

### System OFF

```text
$SYS:OFF
```

### State Update

```text
$STATE:pwm,d2,d3,d4,d5
```

Example:

```text
$STATE:180,1,0,1,0
```

This means:

```text
PWM = 180
CH2 = ON
CH3 = OFF
CH4 = ON
CH5 = OFF
```

### Heartbeat

```text
$HB
```

The Python application periodically sends a heartbeat to ensure the Arduino knows that the controller is still active.

---

# 🛡️ Safety Watchdog

The Arduino contains a **3-second watchdog timeout**.

```cpp
static const uint32_t WATCHDOG_MS = 3000UL;
```

If no valid communication is received for 3 seconds:

```text
System → OFF
PWM → 0
Digital outputs → LOW
```

Arduino sends:

```text
WDOG_TIMEOUT
```

This provides a basic fail-safe mechanism in case:

* Python crashes
* Camera application closes unexpectedly
* USB connection fails
* Computer freezes
* Serial communication stops

---

# 🔄 Automatic Serial Reconnection

The Python serial handler continuously monitors the connection.

If the Arduino becomes unavailable, the application automatically attempts to reconnect.

```text
Python
  │
  ├── Connected ──► Send commands
  │
  └── Disconnected
          │
          ▼
     Retry connection
          │
          ▼
       Connected
```

The default reconnect interval is:

```python
RECONNECT_DELAY = 3.0
```

---

# 🖥️ Real-Time HUD

The application provides an OpenCV-based dashboard showing:

* System status
* Serial status
* CH1 PWM level
* CH2–CH5 states
* Current finger count
* Active channel
* Gesture hold progress
* FPS
* PWM control zones

Example interface concept:

```text
┌──────────────────────────────────────────────┐
│                 CAMERA VIEW                  │
│                                              │
│        ✋                                    │
│                                              │
│                         ┌─────────────────┐  │
│                         │ GESTURE PWM/DIG │  │
│                         │                 │  │
│                         │ SYSTEM ON       │  │
│                         │ SER OK          │  │
│                         │                 │  │
│                         │ CH1 PWM ██████  │  │
│                         │ CH2 DIG    ON  │  │
│                         │ CH3 DIG   OFF  │  │
│                         │ CH4 DIG    ON  │  │
│                         │ CH5 DIG   OFF  │  │
│                         │                 │  │
│                         │ Fingers: 1      │  │
│                         └─────────────────┘  │
└──────────────────────────────────────────────┘
```

---

# ▶️ Running the Application

Connect the Arduino and identify its serial port.

For Windows:

```bash
python main.py --port COM3
```

For example:

```bash
python main.py --port COM5
```

You can also select a different camera:

```bash
python main.py --cam 1
```

Or run without Arduino:

```bash
python main.py --no-serial
```

The demo mode allows the gesture-processing and UI system to be tested without hardware.

---

# ⌨️ Keyboard Controls

| Key | Function             |
| --- | -------------------- |
| `Q` | Quit application     |
| `R` | Reset gesture engine |

---

# 🧩 System Logic

```text
                 START
                   │
                   ▼
             Open Webcam
                   │
                   ▼
           Detect Hand
                   │
                   ▼
          Count Fingers
                   │
                   ▼
        Gesture Debouncing
                   │
          ┌────────┴────────┐
          │                 │
       Fist?              Other
          │                 │
          ▼                 ▼
    Hold 5 seconds     System ON?
          │                 │
          ▼            ┌────┴────┐
      Toggle           │         │
      System           NO        YES
                        │         │
                        ▼         ▼
                      Wait    Process Gesture
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                  1 finger      2–5 fingers   None
                    │             │
                    ▼             ▼
                Pinch PWM      Digital Toggle
                    │             │
                    └──────┬──────┘
                           ▼
                    Serial Update
                           │
                           ▼
                        Arduino
                           │
                           ▼
                      Hardware
```

---

# 🧪 Signal Processing

### PWM Processing Pipeline

```text
Pinch Distance
      ↓
Normalization
      ↓
Median Filter
      ↓
PWM Mapping
      ↓
EMA Smoothing
      ↓
Deadband
      ↓
PWM Output
```

### Why filtering is used

Hand tracking naturally contains small movements and measurement noise.

Without filtering:

```text
120 → 127 → 118 → 131 → 122
```

With smoothing:

```text
120 → 121 → 121 → 123 → 123
```

This produces a much more stable hardware output.

---

# 📡 Communication Reliability

The communication system uses:

* Background serial worker thread
* Thread-safe queue
* Write retry mechanism
* Heartbeat
* Automatic reconnection
* Command prioritization
* Rate limiting
* Arduino watchdog

The state command is rate-limited using:

```python
SERIAL_SEND_HZ = 50
```

This prevents unnecessary serial traffic while maintaining responsive control.

---

# 🔐 Safety Considerations

This project is intended for experimentation, robotics, automation, and educational applications.

When controlling motors, actuators, relays, or other potentially dangerous hardware:

* Use appropriate electrical isolation.
* Do not connect high-power loads directly to Arduino GPIO pins.
* Use proper relay/MOSFET/motor-driver circuits.
* Provide an independent emergency stop.
* Test with low-power loads first.
* Keep the watchdog enabled.
* Ensure the controlled hardware defaults to a safe state.

The software watchdog should **not** be considered a replacement for a physical emergency-stop system.

---

# 🛠️ Customization

You can modify the gesture timing in `config.py`.

For example:

```python
ACTIVATION_HOLD_SECS = 5.0
DIGITAL_TOGGLE_SECS = 1.0
```

PWM smoothing:

```python
PWM_EMA_ALPHA = 0.15
PWM_MEDIAN_WIN = 7
PWM_DEADBAND = 4
```

Pinch sensitivity:

```python
min_d = 20
max_d = 200
```

These values can be tuned according to:

* Camera position
* Hand distance
* Lighting conditions
* User hand size
* Required PWM sensitivity

---

# 🔮 Possible Future Improvements

* Multiple-hand control
* Custom gesture training
* Machine-learning gesture classification
* Bluetooth/Wi-Fi control
* ESP32 support
* MQTT integration
* Voice + gesture hybrid control
* Gesture recording and playback
* Mobile application
* ROS 2 integration
* Robotic-arm control
* Motor speed and direction control
* User-defined gesture-to-channel mapping
* Authentication/authorized gesture profiles

---

# 📌 Applications

This architecture can be adapted for:

* Robotics
* Home automation
* Industrial automation
* Smart appliances
* Motor controllers
* Robotic arms
* Assistive technology
* Educational robotics
* Human-machine interfaces
* Contactless control systems

---

# 📜 License

This project can be used for educational and research purposes. Add or replace this section with your preferred license before publishing the repository.

---

## 👨‍💻 Author

**Mohd Arhan**

Mechanical & Robotics Engineering
Interested in Robotics, Automation, AI, Embedded Systems and Human-Machine Interaction.

---

## ⭐ Project Summary

> **Gesture Controlled Application** converts human hand gestures into real-time hardware control commands using computer vision, Python, and Arduino.

**1 PWM + 4 Digital Channels • Computer Vision • Serial Communication • Watchdog Safety • Real-Time HUD**
