# ESPHome — M5Stack PaperColor

An ESPHome configuration for the [M5Stack PaperColor](https://docs.m5stack.com/en/core/PaperColor) (SKU: C151), a 4" E Ink Spectra 6 (7-color) display device based on the ESP32-S3.

Integrates with Home Assistant to display a dashboard with outdoor temperature, room temperature/humidity, battery level, and WiFi signal strength. Designed for battery-powered use with deep sleep.

![M5Stack PaperColor running ESPHome](PaperColorESPHome3.jpg)

---

## Hardware

| Component | Details |
|---|---|
| SoC | ESP32-S3R8 (dual-core, 240MHz) |
| Display | E Ink Spectra 6 (7-color), 4", 400×600px |
| PSRAM | 8MB OPI |
| Flash | 16MB |
| Temp/Humidity | SHT40 (I2C 0x44) |
| PMIC | M5PM1 (I2C 0x6E) |
| RTC | RX8130CE (I2C 0x32) |
| Battery | 1250mAh LiPo |
| WiFi | 2.4GHz 802.11 b/g/n |

---

## Features

- **7-color e-paper display** — outdoor temp as large hero value (from HA), room temp °F / humidity (local SHT40), MDI battery icon, WiFi signal arc, last update time
- **Color-coded header** — header background changes color based on outdoor temperature: blue (≤64°F), green (≤75.5°F), orange (≤85.5°F), red (>85.5°F); toggleable via HA switch (on by default)
- **Deep sleep** — configurable sleep cycle (default 20 min on battery), ~2–3 day estimated battery life
- **USB-aware** — skips deep sleep when USB connected; the screen refreshes every 10 minutes by default (configurable via the "On USB Refresh Interval" slider in HA)
- **Configurable intervals** — "On Battery Sleep Duration" and "On USB Refresh Interval" sliders (5–120 min, step 5) available in the HA device config; device must be awake for changes to be delivered
- **Battery monitoring** — voltage and percentage from M5PM1 PMIC via I2C; MDI icon varies by charge level and charging state. The piecewise calibration curve is a work in progress — bottom-end accuracy (below ~20%) will improve after a full discharge cycle.
- **Home Assistant integration** — API encrypted, OTA updates, full sensor telemetry
- **Smart boot refresh** — waits for valid sensor values before first display update; fallback refresh if HA is slow to respond
- **3 physical buttons** — A (manual refresh), B (wake from sleep / OTA), C (spare)
- **2× RGB LEDs** — brief blue flash on boot confirmation; blinking green when woken by Button B on battery power (indicates device is awake and ready for wireless OTA flash); LEDs turn off automatically after display update or when USB is connected

---

## Key Implementation Notes

### M5PM1 PMIC Initialization
The EPD power rail is **off by default** and must be enabled at boot before the display can operate. This config uses ESPHome's I2C bus API (not Arduino `Wire`) to write the required M5PM1 registers at `on_boot` priority 800 — before the display component initializes.

The sequence also enables the boost converter (`PWR_CFG` register 0x06, bits 0 and 3), which generates the high-voltage rails (~15V+) required for e-paper pixel switching.

Register sequence derived from [M5GFX source](https://github.com/m5stack/M5GFX) and [M5Stack factory firmware HAL](https://github.com/m5stack/M5PaperColor-UserDemo).

### Display BUSY Pin Polarity
The Spectra-E6 panel uses **inverted BUSY pin polarity** (HIGH = ready, LOW = busy) — opposite of the ESPHome driver default. The `busy_pin` must be configured with `inverted: true` or the display initialization hangs indefinitely. Confirmed from the Seeed reTerminal E1002 reference design (also Spectra-E6).

### I2C Pin Mapping
`SDA=GPIO3`, `SCL=GPIO2` — confirmed from M5GFX source and factory firmware `hal.h`. GPIO3 is an ESP32-S3 strapping pin; the ESPHome warning about this is expected and safe to ignore on this hardware.

### Deep Sleep & OTA
The device sleeps for 20 minutes between refreshes by default (configurable via the "On Battery Sleep Duration" slider in HA). To perform OTA updates:
1. Press **Button B** (GPIO9) to wake the device
2. The **RGB LEDs will blink green** — confirming the device is awake and ready to accept a wireless flash (blue flash = normal timer wake; blinking green = Button B wake)
3. The device stays awake for **60 seconds** — initiate OTA flash within this window
4. LEDs turn off automatically once the display has updated (~30s); the device remains flashable for the rest of the 60s window
5. The display auto-refreshes on every wake (no need to press Button A separately)

---

## ESPHome Requirements

- ESPHome **2026.5.3** or later (the `epaper_spi` component with `Spectra-E6` model was added in ESPHome 2026.x)
- Home Assistant with ESPHome integration

---

## Configuration

Copy `papercolor.yaml` to your ESPHome config directory (`/config/esphome/`).

Add the following entries to your `secrets.yaml`:

```yaml
papercolor_api_key: "your-32-byte-base64-key"   # generate: openssl rand -base64 32
papercolor_ota_password: "your-ota-password"
papercolor_ap_password: "your-fallback-ap-password"
wifi_ssid: "your-wifi-ssid"
wifi_password: "your-wifi-password"
```

Update `use_address` in the wifi block to match your device's IP (set a DHCP reservation in your router using your device's MAC address (visible in your router's DHCP table or in the ESPHome logs under `Local MAC`)).

Update the `ha_outdoor_temp` entity ID to match your outdoor temperature sensor in Home Assistant:
```yaml
  - platform: homeassistant
    id: ha_outdoor_temp
    entity_id: sensor.your_outdoor_temperature_entity
```

---

## First Flash

The device ships with factory firmware. First flash must be done via USB:

1. In ESPHome dashboard, add a new device and select **ESP32-S3**
2. Replace the generated stub config with `papercolor.yaml`
3. Click **Install → Plug into this computer** (USB-C cable required)
4. Select **Factory format** in the ESPHome Web installer

After first flash, all subsequent updates can be done OTA.

---

## Display Layout

```
┌─────────────────────────────────────────────────────────┐
│  )))        HOME ASSISTANT               🔋 CHG/91%     │
├─────────────────────────────────────────────────────────┤
│                    OUTDOORS                              │
│                                                          │
│                      75°F          ← large hero value   │
│                                                          │
│                                                          │
│                                                          │
│ ROOM                                                     │
│ 79.3°F  44% RH                                          │
│ ─────────────────────────────────────────────────────── │
│                  Updated 11:06 AM                        │
└─────────────────────────────────────────────────────────┘
```

---

## Pin Reference

| Function | GPIO |
|---|---|
| I2C SDA | GPIO3 |
| I2C SCL | GPIO2 |
| SPI CLK | GPIO15 |
| SPI MOSI | GPIO13 |
| EPD CS | GPIO44 |
| EPD DC | GPIO43 |
| EPD BUSY | GPIO11 |
| EPD RST | GPIO12 |
| Button A | GPIO10 |
| Button B (wake) | GPIO9 |
| Button C | GPIO1 |
| RGB LEDs | GPIO21 |
| IR TX | GPIO48 |

---

## Credits

- **[Anthropic's Claude Code](https://claude.ai/claude-code)** — this project was developed interactively with Claude Code on real hardware. The PMIC reverse-engineering, display driver research, boot sequencing, and ESPHome integration were all developed and debugged through an AI-assisted session.

- **[PFalko/m5stack-papercolor-esphome](https://github.com/PFalko/m5stack-papercolor-esphome)** — independent ESPHome implementation for the same device. Several M5PM1 PMIC improvements in this config were informed by their reverse-engineering work, specifically: the `HOLD_CFG` power-hold register sequence, enabling the 3.3V LDO rail, disabling hardware button reset functions, and the 5-reading median filter for battery voltage.

- **[M5Stack factory firmware](https://github.com/m5stack/M5PaperColor-UserDemo)** — original HAL source used to derive the M5PM1 EPD power rail init sequence.

- **[M5GFX](https://github.com/m5stack/M5GFX)** — additional PMIC register details and I2C pin mapping confirmation.

---

## Disclaimer

This project is not affiliated with or endorsed by M5Stack. Register sequences for the M5PM1 PMIC were reverse-engineered from M5Stack's open-source [M5GFX](https://github.com/m5stack/M5GFX) and [factory firmware](https://github.com/m5stack/M5PaperColor-UserDemo) — use at your own risk. This config drives a PMIC over I²C; incorrect register writes could potentially damage hardware.

This project was developed with AI assistance (Anthropic's Claude Code) and verified on a real device. It works on my unit — treat it as a starting point and review the code before flashing your own.

## License

MIT
