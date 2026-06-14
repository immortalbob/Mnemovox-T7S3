# MiniSense-T7S3


An ESPHome-based room sensor node built on the LilyGo T7-S3 (ESP32-S3). Combines CO2, temperature, and humidity monitoring with a local voice assistant pipeline using Hey Jarvis wake word detection.

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
- Live OLED display — shows listening / processing / responding states during voice interaction, time / temperature / humidity when idle
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

> **Note:** OLED and SCD40 share the I2C bus (pins 14/18). INMP441 and MAX98357 use separate I2S buses — mic on 15/16, speaker on 5/10 — so simultaneous listen and speak works correctly.

> **Note:** INMP441 L/R must be tied to GND for left channel selection. MAX98357 SD must be tied to 3V3 to enable the amp.

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

## Voice Assistant States

The OLED display takes over with large centered text during voice interaction:

- **LISTENING** — wake word detected, mic active
- **PROCESSING** — speech received, waiting on response
- **RESPONDING** — TTS playing (held on screen for 4 seconds)
- Idle — returns to time, temperature, humidity readout
