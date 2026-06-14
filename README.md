# MiniSense-T7S3

An ESPHome-based room sensor node built on the LilyGo T7-S3 (ESP32-S3). Combines CO2, 
temperature, and humidity monitoring with a fully local voice assistant pipeline using 
Hey Jarvis wake word detection — all processed on-device with no cloud dependency.

The 2.42" SSD1309 OLED does double duty: displaying live sensor readings at idle, then 
taking over with full-screen state transitions as the voice pipeline moves through 
listening, thinking, and responding phases — giving you clear visual feedback at a 
glance without needing to check Home Assistant. If CO2 levels exceed the configured 
threshold the display flashes a warning regardless of voice state.

Designed as a self-contained room node that slots into an existing Home Assistant and 
Ollama/Open WebUI local AI stack.

## Hardware

| Component | Description |
|-----------|-------------|
| LilyGo T7-S3 | ESP32-S3 main board |
| SSD1309 2.42" OLED | 128x64 I2C display |
| SCD40 | CO2, temperature, humidity sensor |
| INMP441 | I2S MEMS microphone |
| MAX98357A | I2S Class D amplifier |

## Features

- Hey Jarvis wake word (on-device, no cloud)
- Voice assistant via Home Assistant
- Live OLED display — shows listening / thinking / responding states during voice interaction
- Random phrases for each voice state — 20 options per state, picked at transition and held for duration
- Idle display shows time, room temperature, humidity, CO2, and WiFi signal bars
- CO2 warning — display flashes when threshold is exceeded (default 1000ppm)
- SCD40 sensors reported to Home Assistant
- Board LED controllable from HA

## Pin Assignments

| T7S3 Pin | Device | Device Pin |
|----------|--------|------------|
| 3V3 | OLED, SCD40, INMP441, MAX98357 SD | VDD / SD |
| GND | OLED, SCD40, INMP441 L/R, MAX98357 | GND / L/R |
| 5V | MAX98357 | VIN |
| 14 | OLED, SCD40 | SCL |
| 18 | OLED, SCD40 | SDA |
| 15 | INMP441 | WS |
| 16 | INMP441 | SCK |
| 8 | INMP441 | SD |
| 5 | MAX98357 | LRC |
| 10 | MAX98357 | BCLK |
| 7 | MAX98357 | DIN |
| 17 | Board LED | — |

> **Note:** INMP441 and MAX98357 use separate I2S buses — mic on 15/16, speaker on 5/10. The hardware supports simultaneous audio in/out but ESPHome's voice assistant pipeline does not currently support barge-in; wake word detection resumes after the response finishes.

> **Note:** INMP441 L/R must be tied to GND for left channel selection. MAX98357 SD must be tied to 3V3 to enable the amp.

> **Note:** SCL is on pin 14 rather than the more common pin 17, which is reserved for the board LED.

## First Time Flash

### Requirements

- Python 3.x
- ESPHome CLI

```bash
pip install esphome
```

### 1. Generate an API key

```bash
python -c "import base64, os; print(base64.b64encode(os.urandom(32)).decode())"
```

### 2. Edit the bootstrap config

Copy `t7s3-bootstrap.yml` and fill in:

- `YOUR_WIFI_SSID` / `YOUR_WIFI_PASSWORD`
- `YOUR_BASE64_API_KEY` — from the command above
- `YOUR_OTA_PASSWORD` — any password you choose

### 3. Flash via USB

Plug the T7S3 into your PC and check Device Manager for the COM port, then:

```bash
esphome run t7s3-bootstrap.yml --device COM11
```

### 4. Verify on network

```bash
ping t7s3-room-sensor.local
```

### 5. Add to Home Assistant

Settings → Devices & Services → Add Integration → ESPHome

- Host: IP shown by ping above
- Port: 6053
- Key: your base64 API key

## Full Config (OTA after first flash)

Edit `t7s3-room-sensor.yml` with your credentials and flash:

```bash
esphome run t7s3-room-sensor.yml
```

No `--device` needed after first flash — updates over WiFi.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Not on network | Check wifi credentials, ping t7s3-room-sensor.local |
| Not in HA | Add manually via IP, port 6053 |
| Wake word not detecting | Check INMP441 wiring, L/R must be grounded |
| No speaker output | Check MAX98357 SD pin is on 3V3 |
| OLED blank | Confirm I2C address 0x3C, check pins 14/18 |
| SCD40 no readings | Allow 3 minutes after boot for first measurement |
| Compile error shadow warning | Ensure `-Wno-shadow` build flag is present |
| Words clipped in response | Increase `buffer_duration` on speaker, lower `noise_suppression_level` |
| CO2 warning not flashing | Confirm `uptime` sensor has `id: uptime_sensor` set |

## Voice Assistant States

The OLED display takes over with large centered text during voice interaction. Each state picks randomly from 20 phrases at transition time and holds it for the duration of the state — keeping interactions fresh without flickering.

- **LISTENING** — wake word detected, mic active (e.g. "YEAH?", "GO AHEAD", "I'M HERE")
- **THINKING** — speech received, waiting on response from Home Assistant (e.g. "ON IT", "HMMM...", "BRAIN HURTS")
- **RESPONDING** — TTS playing, held on screen for 4 seconds (e.g. "BOOM", "HERE YA GO", "HOPE SO")
- **CO2 HIGH** — flashes when CO2 exceeds threshold, overrides idle display
- **Idle** — displays time, room temperature, humidity and CO2 in small text, WiFi signal bars in top right corner

## CO2 Threshold

The warning threshold is set via the `co2_warning_threshold` global at the top of the config. Default is 1000ppm — the ASHRAE action threshold above which air quality begins to affect cognitive function. Adjust to suit your environment.

| Level | Meaning |
|-------|---------|
| <1000ppm | Normal |
| 1000ppm | Action threshold — open a window |
| 1500ppm | Stuffy, ventilate now |
| 2000ppm+ | Unhealthy |

## To Do

### mmWave Presence Detection

Planned addition: HLK-LD2410B mmWave sensor for true presence detection (not just motion).
This will allow automations to detect occupancy even when the room occupant is stationary.

**Additional pin assignments:**

| T7S3 Pin | Device | Device Pin |
|----------|--------|------------|
| 3V3 | LD2410B | VCC |
| GND | LD2410B | GND |
| 13 | LD2410B | RX |
| 12 | LD2410B | TX |

**Config changes required:**
- Add `uart:` block with `tx_pin: 13`, `rx_pin: 12`, `baud_rate: 256000`
- Add `ld2410` sensor platform block
- Add binary sensors for presence and motion detection
- Add sensor for detection distance
- Update repo topics to include `ld2410` and `presence-detection`
