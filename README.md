# chasing_your_tail — ESP32 Wi-Fi & BLE Device Tracker

A compact, efficient ESP32 proof-of-concept for detecting and tracking nearby Wi-Fi and Bluetooth Low Energy devices. Displays results on an LCD with visual and audio alerts when suspicious devices are detected.

---

## Features

### 📡 **Core Scanning**
- **Wi-Fi Promiscuous Mode**: Captures 802.11 management frames (probe requests)
- **BLE Scanner**: Periodic Bluetooth Low Energy advertisement scanning
- **Unified Tracker**: Single hash table combines both Wi-Fi and BLE MAC addresses

### 🎛 **Device Tracking**
- **Fixed-size Hash Table** (128 entries): O(1) lookups, eviction on collision
- **Eviction Policy**: Removes lowest-seen entries (ties broken by oldest timestamp)
- **Suspicious Threshold**: Devices seen ≥10 times marked as suspicious
- **Type Tracking**: Records device source (Wi-Fi, BLE, or both)

### 📺 **User Interface**
- **TFT LCD Display** (multiple screens via rotary encoder):
  - **Summary**: Total devices, suspicious count, current Wi-Fi channel
  - **Device List**: All MACs with counts, color-coded (green=normal, red=suspicious)
  - **Alerts**: Red-highlighted suspicious devices only
  - **Settings**: Clear logs or cycle Wi-Fi channels 1–13

### ⚠️ **Alerts**
- **NeoPixel Ring**: Green (normal) or Red (suspicious detected)
- **Speaker Beep**: 2 kHz tone when threshold reached

### 🔧 **Control**
- **Rotary Encoder**: Scroll menu items (A/B + push button)
- **Settings Menu**: Clear device logs, toggle Wi-Fi channel

---

## Hardware Wiring

| Component | ESP32 Pin | Notes |
|-----------|-----------|-------|
| **NeoPixel Ring (Data)** | GPIO 13 | WS2812B RGB ring, 8 LEDs |
| **Speaker** | GPIO 25 | Via LEDC PWM (tone output) |
| **Rotary Encoder A** | GPIO 32 | Phase A |
| **Rotary Encoder B** | GPIO 33 | Phase B |
| **Encoder Button** | GPIO 27 | Pull-up internally enabled |
| **TFT Display (SPI)** | CS, CLK, MOSI, DC, RST | See TFT_eSPI User_Setup.h |

### Adjusting Pins
Edit the `#define` block at the top of `chasing_your_tail.ino`:
```cpp
#define PIN_NEOPIXEL 13
#define PIN_SPEAKER 25
#define PIN_ENCODER_A 32
#define PIN_ENCODER_B 33
#define PIN_ENCODER_BTN 27
```

---

## Compilation & Upload

### Prerequisites
1. **Arduino IDE** or **PlatformIO** (ESP32 Arduino core)
2. **ESP32 Board Package** (via Arduino Board Manager)
3. **Libraries**:
   - `TFT_eSPI` (Bodmer) — Configure `User_Setup.h` for your display
   - `Adafruit_NeoPixel`
   - `BLEDevice`, `BLEUtils`, `BLEScan` (included in ESP32 core)

### Arduino IDE
1. **Install ESP32 Board**:
   - File → Preferences → Additional Board Manager URLs
   - Add: `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
   - Tools → Board Manager → Search "ESP32" → Install

2. **Install Libraries**:
   - Sketch → Include Library → Manage Libraries
   - Search and install: `TFT_eSPI`, `Adafruit NeoPixel`

3. **Configure TFT_eSPI**:
   - Locate `Arduino/libraries/TFT_eSPI/User_Setup.h`
   - Uncomment/edit pins to match your display wiring (SPI clock, MOSI, DC, CS, RST)

4. **Select Board & Port**:
   - Tools → Board → "ESP32 Dev Module" (or your variant)
   - Tools → Port → Select your COM port
   - Baud Rate: 115200

5. **Upload**:
   - Sketch → Upload (or Ctrl+U)

### PlatformIO
```ini
[env:esp32]
platform = espressif32
board = esp32doit-devkit-v1
framework = arduino
lib_deps = 
    TFT_eSPI
    Adafruit NeoPixel
```

Upload with: `platformio run -t upload`

---

## Operation

### Startup
1. Initializes TFT, NeoPixel, speaker, encoder inputs, and Wi-Fi/BLE scanning
2. Shows **Summary** screen with 0 devices
3. Opens serial monitor (115200 baud) to see debug logs

### Normal Use
1. **Rotate encoder** to navigate menu (Summary → List → Alerts → Settings)
2. **Press encoder button** to select screen
3. **New devices** appear in Device List with green text
4. **Suspicious devices** (count ≥ 10) turn red + beep + NeoPixel turns red
5. On **Alerts** screen, only red devices shown
6. On **Settings** screen, press button to **clear logs** or **cycle Wi-Fi channel**

### Serial Output
Open Tools → Serial Monitor (115200 baud) to see:
- **New devices**: `New device AA:BB:CC:DD:EE:FF type=WiFi`
- **Evictions**: `Evicting AA:BB:CC:DD:EE:FF count=2 lastSeen=... slot=45`
- **Suspicious alerts**: Logged when threshold reached (check code for details)

---

## Configuration & Tweaks

### Memory Footprint
- **Table size**: 128 entries (adjust `TABLE_SIZE` in code)
- **Each entry**: ~30 bytes → ~3.8 KB total tracker
- **To reduce**: Lower `TABLE_SIZE` (power of 2 recommended: 64, 128, 256)

### Suspicious Threshold
```cpp
const int SUSPICIOUS_THRESHOLD = 10;  // Change to 5, 15, etc.
```

### BLE Scan Interval
```cpp
const unsigned long BLE_SCAN_INTERVAL_MS = 10000;  // 10 seconds
```

### Wi-Fi Channels
Default cycles 1–13 (US/EU). Adjust in `setWifiChannel()` if needed.

### Speaker Tone
```cpp
ledcWriteTone(0, 2000);  // Hz (adjust frequency)
delay(120);              // Duration (ms)
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Compilation error: TFT_eSPI not found** | Install via Library Manager; check User_Setup.h exists |
| **NeoPixel not lighting** | Verify pin (GPIO 13), power supply, WS2812B data line |
| **Encoder not responding** | Check GPIO 32/33/27 connectivity; confirm INPUT_PULLUP mode |
| **No Wi-Fi captures** | Verify esp_wifi functions available (ESP32 core issue); check promiscuous mode enabled |
| **BLE scan hangs** | Increase `BLE_SCAN_INTERVAL_MS` or reduce `pBLEScan->start(3, ...)` timeout |
| **Table full, devices disappearing** | Increase `TABLE_SIZE` or adjust eviction policy |
| **Serial output garbage** | Verify baud rate is 115200 |

---

## Code Structure

- **Tracker**: Fixed-size open-addressing hash table with 6-byte MAC keys
- **UI**: Four draw functions (Summary, List, Alerts, Settings) rendering to TFT
- **Input**: Rotary encoder polling in `pollEncoder()`
- **Alerts**: `updateAlertsVisuals()` checks for suspicious devices every loop
- **Callbacks**: `wifiSnifferCallback()` and `MyAdvertisedDeviceCallbacks` feed MAC addresses into tracker

---

## Future Enhancements

- [ ] TF card logging (save detections to SD)
- [ ] MQTT push (send alerts to remote broker)
- [ ] Adjustable threshold via menu
- [ ] Geofencing or signal strength filtering
- [ ] Data persistence across reboots
- [ ] Multi-protocol fusion (combining Wi-Fi + BLE confidence scores)

---

## License & Notes

This is a proof-of-concept for learning. Adapt as needed for your use case.

**WARNING**: Use responsibly. Monitoring devices without consent may violate local laws. Intended for:
- Security research in controlled environments
- Network debugging on devices you own
- Educational projects

---

## Support

For issues or questions:
1. Check **Troubleshooting** section above
2. Review serial debug output (115200 baud)
3. Verify hardware wiring and pin configurations
4. Inspect sketch variables and thresholds

Enjoy tracking! 🎯
