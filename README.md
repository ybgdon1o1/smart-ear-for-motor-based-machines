

## 📋 Table of Contents

- [Hardware Requirements](#-hardware-requirements)
- [Wiring Diagram](#-wiring-diagram)
- [Project Structure](#-project-structure)
- [ESP32 Firmware Setup](#-esp32-firmware-setup)
- [Python Application Setup](#-python-application-setup)
- [Usage Guide](#-usage-guide)
- [Serial Communication Protocol](#-serial-communication-protocol)
- [Dataset Output Format](#-dataset-output-format)
- [Edge Impulse Upload](#-edge-impulse-upload)
- [Troubleshooting](#-troubleshooting)

---

## 🔧 Hardware Requirements

| Component | Model | Purpose |
|-----------|-------|---------|
| Microcontroller | ESP32 DevKit V1 | Main processing unit |
| Microphone | INMP441 I2S MEMS | Acoustic signal capture |
| Display | SH1106 1.3" OLED (I2C) | Status display |
| Motor | DC Motor | Fault simulation target |
| Safety | Relay Module | Motor safety control |
| Cable | USB Micro-B | Serial data + power |

---

## 🔌 Wiring Diagram

### INMP441 Microphone → ESP32

```
INMP441          ESP32
────────         ──────
VDD  ──────────→ 3.3V
GND  ──────────→ GND
L/R  ──────────→ GND        (selects Left channel)
SCK  ──────────→ GPIO 14    (I2S Bit Clock)
WS   ──────────→ GPIO 15    (I2S Word Select / LRCLK)
SD   ──────────→ GPIO 32    (I2S Serial Data)
```

### SH1106 OLED Display → ESP32

```
SH1106           ESP32
──────           ──────
VCC  ──────────→ 3.3V
GND  ──────────→ GND
SDA  ──────────→ GPIO 21    (I2C Data)
SCL  ──────────→ GPIO 22    (I2C Clock)
```

### Connection Notes

- **INMP441 L/R pin to GND**: This selects the Left channel. The firmware uses `I2S_CHANNEL_FMT_ONLY_LEFT`.
- **3.3V only**: Both INMP441 and SH1106 operate at 3.3V. Do NOT connect to 5V.
- **GPIO 14, 15, 32**: These are confirmed working pins for I2S on ESP32 DevKit V1.
- **OLED I2C address**: `0x3C` (default for most SH1106 modules).

---

## 📁 Project Structure

```
tinyml-predictive-maintenance/
│
├── esp32_data_collector/
│   └── esp32_data_collector.ino    ← ESP32 firmware (Arduino sketch)
│
├── python_tools/
│   ├── data_collector_gui.py       ← Main GUI application (PyQt5)
│   ├── wav_recorder.py             ← CLI recorder (lightweight alternative)
│   ├── verify_dataset.py           ← Dataset verification tool
│   └── requirements.txt            ← Python dependencies
│
├── dataset/                        ← Generated WAV files (auto-created)
│   ├── normal/
│   └── loose_bearing/
│
├── esp32_inference/                ← TinyML inference firmware
│   └── esp32_inference.ino
│
├── dashboard/                      ← Web monitoring dashboard
│   ├── index.html
│   ├── style.css
│   └── app.js
│
└── README.md                       ← This file
```

---

## 🔥 ESP32 Firmware Setup

### Prerequisites

1. **Arduino IDE** (v2.x recommended) or **PlatformIO**
2. **ESP32 Board Package**: 
   - In Arduino IDE → File → Preferences → Additional Board Manager URLs:
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
   - Tools → Board Manager → Search "esp32" → Install "esp32 by Espressif Systems"

3. **Required Libraries** (install via Library Manager):
   - `Adafruit SH110X` (for SH1106 OLED)
   - `Adafruit GFX Library` (dependency)

### Upload Steps

1. Connect ESP32 via USB cable
2. Open `esp32_data_collector/esp32_data_collector.ino` in Arduino IDE
3. Select Board: **Tools → Board → ESP32 Dev Module**
4. Select Port: **Tools → Port → COMx** (your ESP32 port)
5. Board Settings:
   - Upload Speed: `921600`
   - Flash Frequency: `80MHz`
   - CPU Frequency: `240MHz`
   - Flash Mode: `QIO`
   - Partition Scheme: `Default 4MB with spiffs`
6. Click **Upload** (→ button)
7. Open Serial Monitor at **921600 baud** to verify boot messages

### Verification

After upload, the Serial Monitor should show:
```
=================================
TinyML Predictive Maintenance
Data Collection Mode
Baud: 921600
=================================
I2S initialized successfully.
OLED initialized successfully.
READY
```

Type `STATUS` in Serial Monitor to test → should respond with `STATUS:IDLE,0,5`

---

## 🐍 Python Application Setup

### Prerequisites

- **Python 3.8+** (Python 3.10+ recommended)
- **pip** package manager

### Installation

```bash
# Navigate to the python_tools directory
cd tinyml-predictive-maintenance/python_tools

# Install dependencies
pip install -r requirements.txt
```

### Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| pyserial | ≥3.5 | Serial communication with ESP32 |
| PyQt5 | ≥5.15 | GUI framework |
| pyqtgraph | ≥0.13 | Real-time waveform & spectrogram plots |
| numpy | ≥1.21 | Audio data processing |
| sounddevice | ≥0.4 | Audio playback |
| scipy | ≥1.7 | FFT / spectrogram computation |

### Launch the GUI

```bash
python data_collector_gui.py
```

### Alternative: CLI Recorder

For quick recording without the GUI:
```bash
python wav_recorder.py --port COM3 --label normal --count 50
python wav_recorder.py --port COM3 --label loose_bearing --count 50 --delay 1.5
```

---

## 🎯 Usage Guide

### Step 1: Connect Hardware

1. Wire the INMP441 and OLED as shown in the wiring diagram
2. Connect ESP32 to PC via USB cable
3. Upload the firmware (if not already done)

### Step 2: Launch the Application

```bash
cd python_tools
python data_collector_gui.py
```

### Step 3: Connect to ESP32

1. The application auto-detects available COM ports
2. Select the correct port from the dropdown (or click Refresh)
3. Click **Connect** — status indicator turns green

### Step 4: Configure Recording Session

| Setting | Recommended Value | Description |
|---------|------------------|-------------|
| Class Label | `normal` | Fault class being recorded |
| Number of Samples | 50 | Files per session |
| Recording Duration | 5 seconds | Audio length per sample |
| Delay Between Samples | 1.0 seconds | Pause between recordings |

### Step 5: Start Batch Recording

1. Click **START BATCH** (green button)
2. Monitor progress in the GUI:
   - Live waveform shows captured audio
   - Spectrogram shows frequency content
   - Level meter shows audio amplitude
   - Progress bar and counter track completion
3. System automatically:
   - Sends commands to ESP32
   - Captures audio for specified duration
   - Saves WAV file to `dataset/<label>/`
   - Validates each recording (duration, RMS, integrity)
   - Retries failed recordings (up to 3 attempts)
   - Waits the specified delay before next sample
4. Click **STOP** at any time to halt the batch

### Step 6: Review & Manage Dataset

- Browse recorded files in the Dataset Browser panel
- Click **Play** to listen to any sample
- Click **Delete** to remove bad samples
- Click **Validate Dataset** for a full quality check
- Click **Export Statistics** for a CSV report

### Step 7: Repeat for Each Fault Class

Recommended minimum per class for Edge Impulse:
- **normal**: 50+ samples
- **loose_bearing**: 50+ samples

---

## 📡 Serial Communication Protocol

### Overview

| Parameter | Value |
|-----------|-------|
| Baud Rate | 921600 |
| Data Bits | 8 |
| Parity | None |
| Stop Bits | 1 |
| Line Ending | `\n` (LF) |

### Commands (PC → ESP32)

| Command | Description |
|---------|-------------|
| `START_RECORDING\n` | Begin audio capture to RAM buffer |
| `STOP_RECORDING\n` | Abort current recording |
| `STATUS\n` | Request current state |
| `SET_DURATION:N\n` | Set recording duration (N = 1–30 seconds) |

### Responses (ESP32 → PC)

| Response | Description |
|----------|-------------|
| `READY\n` | System initialized, ready for commands |
| `RECORDING_START\n` | Audio capture has begun |
| `DATA_START:NBYTES\n` | Binary data follows (NBYTES = byte count) |
| *(binary PCM data)* | Exactly NBYTES of 16-bit LE PCM samples |
| `DATA_END\n` | Binary transfer complete |
| `RECORDING_END\n` | Full cycle complete |
| `STATUS:state,count,duration\n` | Current state report |
| `ERROR:message\n` | Error description |
| `INFO:message\n` | Diagnostic message |

### Recording Sequence Diagram

```
  PC (Python App)              ESP32
       │                         │
       │   START_RECORDING\n     │
       │────────────────────────→│
       │                         │
       │   RECORDING_START\n     │
       │←────────────────────────│
       │                         │  ← Capturing audio to RAM
       │    (wait N seconds)     │     (no serial output)
       │                         │
       │   DATA_START:160000\n   │
       │←────────────────────────│
       │                         │
       │   <160000 bytes PCM>    │  ← Binary data in 1024-byte chunks
       │←════════════════════════│
       │                         │
       │   DATA_END\n            │
       │←────────────────────────│
       │                         │
       │   RECORDING_END\n       │
       │←────────────────────────│
       │                         │
       │   (save & validate WAV) │
       │                         │
```

### Data Format

- **Audio Format**: 16-bit signed PCM, little-endian
- **Sample Rate**: 16,000 Hz
- **Channels**: 1 (Mono)
- **Byte calculation**: `duration_seconds × 16000 × 2 = total_bytes`
  - 1 second = 32,000 bytes
  - 5 seconds = 160,000 bytes
  - 10 seconds = 320,000 bytes

---

## 📂 Dataset Output Format

### Directory Structure

```
dataset/
├── normal/
│   ├── normal_001.wav
│   ├── normal_002.wav
│   ├── normal_003.wav
│   └── ...
├── loose_bearing/
│   ├── loose_bearing_001.wav
│   ├── loose_bearing_002.wav
│   └── ...
└── loose_bearing/
    ├── loose_bearing_001.wav
    └── ...
```

### WAV File Specifications

| Property | Value |
|----------|-------|
| Format | RIFF WAV (PCM) |
| Sample Rate | 16,000 Hz |
| Bit Depth | 16-bit signed |
| Channels | 1 (Mono) |
| Byte Order | Little-endian |
| Duration | As configured (default 5 seconds) |
| File Size | ~160 KB for 5 seconds |

These files are **directly uploadable** to Edge Impulse without any post-processing.

---

## 🚀 Edge Impulse Upload

### Method 1: Studio Web UI

1. Go to [studio.edgeimpulse.com](https://studio.edgeimpulse.com)
2. Create a new project → Select "Audio Classification"
3. Go to **Data Acquisition** tab
4. Click **Upload Data**
5. Select all WAV files from a class folder
6. Set the label to match the folder name
7. Choose split (e.g., 80% training, 20% testing)
8. Repeat for each class

### Method 2: Edge Impulse CLI

```bash
# Install the CLI
npm install -g edge-impulse-cli

# Upload with auto-labeling (folder name = label)
edge-impulse-uploader --label normal dataset/normal/*.wav
edge-impulse-uploader --label loose_bearing dataset/loose_bearing/*.wav
```

### Recommended Impulse Design

1. **Input Block**: Audio (Window size: 5000ms, Window increase: 5000ms, Frequency: 16000 Hz)
2. **Processing Block**: MFE (Mel-Filterbank Energy) — Frame length: 0.04, Frame stride: 0.02, Filters: 40, FFT length: 512, Low freq: 100 Hz, High freq: 4000 Hz, Noise floor: -40 dB
3. **Learning Block**: Classification (Keras) — 16/32 conv filters, 0.30 dropout, 150 training cycles, data augmentation ON
4. **Output Features**: normal, loose_bearing

---

## 🔍 Troubleshooting

### ESP32 Not Detected

| Issue | Solution |
|-------|----------|
| No COM port visible | Install CP2102 or CH340 USB driver |
| Port busy error | Close Arduino Serial Monitor before running Python app |
| Multiple COM ports | Unplug other USB devices, or check Device Manager |

### No Audio / Silent Recordings

| Issue | Solution |
|-------|----------|
| All-zero samples | Check INMP441 wiring (especially SD → GPIO 32) |
| Very low RMS (< 100) | Verify INMP441 VDD is connected to 3.3V |
| L/R pin floating | Connect INMP441 L/R pin to GND |
| Noise but no signal | Check SCK and WS connections |

### Recording Timeouts

| Issue | Solution |
|-------|----------|
| START marker timeout | Reset ESP32, check serial connection |
| DATA timeout | Reduce recording duration, check USB cable quality |
| Corrupted data | Use a shorter, higher-quality USB cable |

### Python Application Issues

| Issue | Solution |
|-------|----------|
| `ModuleNotFoundError` | Run `pip install -r requirements.txt` |
| `Permission denied` on COM port | Run as Administrator, or close other serial apps |
| GUI freezes | Check that ESP32 is responding (try `STATUS` command) |
| PyQt5 not installing | Use `pip install PyQt5==5.15.9` (specific version) |
| pyqtgraph import error | `pip install pyqtgraph numpy` |

### Audio Quality Issues

| Issue | Solution |
|-------|----------|
| Clipping (amplitude hitting ±32767) | Move microphone further from sound source |
| DC offset | Expected with `>>16` shift; Edge Impulse handles this |
| Background noise | Record in a controlled environment |
| Inconsistent volumes | Keep microphone placement consistent |

---

## 📄 License

This project is part of a Final Year Project for academic purposes.

---

## 🙏 Acknowledgments

- [Edge Impulse](https://edgeimpulse.com) — TinyML model training platform
- [Espressif](https://www.espressif.com) — ESP32 development tools
- [InvenSense (TDK)](https://invensense.tdk.com) — INMP441 MEMS microphone
- [Adafruit](https://adafruit.com) — SH110X display libraries
