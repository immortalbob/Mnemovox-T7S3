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
- ESP32-S3 internal die temperature reported to Home Assistant
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

## Configuration

All tunable options are documented inline in `t7s3-room-sensor.yml`. Key settings:

**CO2 Warning Threshold**
Set via `co2_warning_threshold` global at the top of the config. Default 1000ppm — the ASHRAE action threshold above which air quality begins to affect cognitive function.

| Level | Meaning |
|-------|---------|
| <1000ppm | Normal |
| 1000ppm | Action threshold — open a window |
| 1500ppm | Stuffy, ventilate now |
| 2000ppm+ | Unhealthy |

**SCD40 Sensor**

| Option | Default | Notes |
|--------|---------|-------|
| `temperature_offset` | 0 | Adjust if readings drift from a reference. Tested at 0 for this enclosure — self-heating negligible |
| `altitude_compensation` | 0m | Set to your elevation in meters for more accurate CO2 readings. Find yours at whatismyelevation.com |
| `automatic_self_calibration` | false | Disabled — assumes sensor is always indoors and never sees fresh outdoor air |
| `measurement_mode` | periodic | Switch to `low_power` for battery operation (30s intervals vs 5s) |

**Display**

| Option | Default | Notes |
|--------|---------|-------|
| `brightness` | 100% | Reduce if display is too bright in a dark room |
| `update_interval` | 500ms | How often the display redraws |
| `invert` | false | White background, black text — useful in high ambient light |

**Audio**

| Option | Default | Notes |
|--------|---------|-------|
| `auto_gain` | 20dBFS | Reduced from 25dBFS — INMP441 is a hot mic, higher values risk saturating the wake word model |
| `noise_suppression_level` | 1 | 0-4, increase if background noise causes false triggers |
| `buffer_duration` | 1500ms | Increase if words are clipped at the start of responses |
| `timeout` | 500ms | How long after playback ends before speaker task suspends |
| `use_apll` | true | ESP32 Audio PLL for more accurate clock timing on both mic and speaker |

## Voice Assistant States

The OLED display takes over with large centered text during voice interaction. Each state picks randomly from 20 phrases at transition time and holds it for the duration of the state — keeping interactions fresh without flickering.

- **LISTENING** — wake word detected, mic active (e.g. "YEAH?", "GO AHEAD", "I'M HERE")
- **THINKING** — speech received, waiting on response from Home Assistant (e.g. "ON IT", "HMMM...", "CRUNCHING")
- **RESPONDING** — TTS playing, held on screen for 4 seconds (e.g. "HERE YA GO", "NO PROBLEM", "ON THE AIR")
- **CO2 HIGH** — flashes when CO2 exceeds threshold, overrides idle display
- **Idle** — displays time, room temperature, humidity and CO2 in small text, WiFi signal bars in top right corner

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Not on network | Check wifi credentials, ping t7s3-room-sensor.local |
| Not in HA | Add manually via IP, port 6053 |
| Wake word not detecting | Check INMP441 wiring, L/R must be grounded |
| No speaker output | Check MAX98357 SD pin is on 3V3 |
| OLED blank | Confirm I2C address 0x3C, check pins 14/18 |
| SCD40 no readings | Allow 3 minutes after boot for first measurement |
| Temperature reads wrong | Adjust `temperature_offset` in scd4x block. Compare against a known reference |
| CO2 reads wrong | Set `altitude_compensation` to your elevation in meters |
| Compile error shadow warning | Ensure `-Wno-shadow` build flag is present |
| Words clipped in response | Increase `buffer_duration`, lower `noise_suppression_level` |
| Audio crackling | Try setting `use_apll: false` on mic and speaker |

## To Do

### mmWave Presence Detection

Planned addition: DFRobot Fermion C4002 mmWave Human Presence Sensor for true presence detection (not just motion). Detects static presence up to 10m and motion up to 11m using 24GHz FMCW radar. Includes an integrated ambient light sensor (0-50 lux) enabling condition-based automations without a separate light sensor. Built-in adaptive noise filtering ignores ceiling fans, curtains, and plants.

Uses a DFRobot external ESPHome component — not a native ESPHome platform. Source: https://github.com/cdjq/Home_Assistant_C4002

> **Status:** Hardware confirmed working (OUT LED active on presence detection). ESPHome component integration causes boot instability on this hardware combination — likely a memory conflict between the C4002 component and the micro_wake_word pipeline. Removed pending component maturity or a workaround. Pin assignments and config blocks documented below for future integration.

**Important:** After first boot or reset, leave the room within 10 seconds and wait 40 seconds for ambient noise calibration.

**Additional pin assignments:**

| T7S3 Pin | Device | Device Pin |
|----------|--------|------------|
| 3V3 | C4002 | VCC |
| GND | C4002 | GND |
| 13 | C4002 | RX |
| 12 | C4002 | TX |

**Config blocks required:**

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/immortalbob/esphome-dfrobot-c4002.git
      ref: dev
    components:
      - dfrobot_c4002
    refresh: 0s

uart:
  id: uart_bus
  tx_pin: GPIO13
  rx_pin: GPIO12
  baud_rate: 115200

dfrobot_c4002:
  id: my_c4002
  uart_id: uart_bus

sensor:
  - platform: dfrobot_c4002
    c4002_id: my_c4002
    movement_distance:
      name: "Motion Distance"
      unit_of_measurement: "m"
      accuracy_decimals: 2
    existing_distance:
      name: "Presence Distance"
      unit_of_measurement: "m"
      accuracy_decimals: 2
    movement_speed:
      name: "Motion Speed"
    movement_direction:
      name: "Motion Direction"
      internal: true
    target_status:
      name: "Target Status"
      internal: true

text_sensor:
  - platform: template
    name: "Movement Direction"
    lambda: |-
      int d = id(movement_direction_sensor).state;
      if (d == 0) return {"Away"};
      else if (d == 1) return {"No Direction"};
      else if (d == 2) return {"Approaching"};
      else return {"Unknown"};
    update_interval: 1s
  - platform: template
    name: "Target Status"
    lambda: |-
      int d = id(target_status_sensor).state;
      if (d == 0) return {"No Target"};
      else if (d == 1) return {"Static Presence"};
      else if (d == 2) return {"Motion"};
      else return {"Unknown"};
    update_interval: 0.5s

switch:
  - platform: dfrobot_c4002
    switch_factory_reset:
      name: "Factory Reset"
    switch_environmental_calibration:
      name: "Sensor Calibration"

select:
  - platform: dfrobot_c4002
    c4002_id: my_c4002
    operating_mode:
      name: "OUT Mode"
      options:
        - "Mode_1"
        - "Mode_2"
        - "Mode_3"

# Uncomment to configure detection zones, range, and light threshold
# number:
#   - platform: dfrobot_c4002
#     max_range:
#       name: "Max detection distance"
#     min_range:
#       name: "Min detection distance"
#     light_threshold:
#       name: "Light Threshold"
#     area1_min:
#       name: "Area 1 Excluded Range Min"
#     area1_max:
#       name: "Area 1 Excluded Range Max"
#     target_disappeard_delay_time:
#       name: "Target Disappear Delay Time"
```

**Available sensors:**
- `movement_distance` — distance to moving target in meters
- `existing_distance` — distance to static target in meters
- `movement_speed` — speed of moving target
- `movement_direction` — Away / No Direction / Approaching
- `target_status` — No Target / Static Presence / Motion
- Light threshold configurable via number platform (lux sensor)

**Additional config changes once resolved:**
- Add presence indicator to OLED display (top left corner)
- Add binary sensor for occupied/vacant derived from `target_status`
- Auto dim OLED display when ambient light drops below lux threshold
- Update repo topics to include `c4002`, `mmwave`, and `presence-detection`
**Additional config changes once added:**
- Add presence indicator to OLED display (top left corner)
- Add binary sensor for occupied/vacant derived from `target_status`
- Auto dim OLED display when ambient light drops below lux threshold — use `light_threshold` number entity to set the trigger level and adjust `brightness` on the display component accordingly
- Update repo topics to include `c4002`, `mmwave`, and `presence-detection`

  
## Part of the Mnemo-Net stack

- [Mnemolis](https://github.com/immortalbob/Mnemolis) — unified local knowledge search API
- [Mnemolis Intents](https://github.com/immortalbob/Mnemolis_Intents) — native Home Assistant LLM integration for MiniSearch
