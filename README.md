# humidity-temp-esphome

Small ESPHome node that measures temperature and relative humidity, shows both
on a 4-digit 7-segment display and publishes everything to MQTT.

```
   [2][3] :  [4][1]
 temperature  humidity %RH
```

* **Temperature** on digits 1-2, **humidity** on digits 3-4 (both rounded to integers).
* The **two colon dots blink alternately** - exactly one dot is lit at a time.
* If the humidity leaves the **comfort band** (default 40-60 %RH, adjustable over
  MQTT) the **humidity digits blink** slowly.
* A BH1750 ambient light sensor **auto-dims the display** so it is readable but
  not glaring at night.
* **OTA updates** and MQTT auto-discovery in Home Assistant.

---

## Features / design notes

| Topic | Choice |
| --- | --- |
| MCU | ESP8266, Wemos D1 Mini (30-pin clone, 4 MB flash). ESP32 is a two-line change - see the header of `humidity-temp.yaml`. |
| Configuration | Everything sits in one file, `humidity-temp.yaml`. No `packages:`, no custom C++. |
| Display rendering | Raw segment bytes via `TM1637Display::set_buffer()`, which gives per-digit control including the decimal-point bits. Requires **ESPHome >= 2026.4**. |
| Runtime configuration | ESPHome `number` entities (`optimistic`, `restore_value`), so the values survive a reboot and can be changed over MQTT at any time. |
| Protocols | MQTT only - no native API, no web server. WiFi SSID/password and MQTT credentials come from `secrets.yaml`. |

---

## Hardware

| Part | Notes |
| --- | --- |
| Wemos D1 Mini | ESP8266, 4 MB flash, CH340 USB-serial |
| AHT20 | Temperature + humidity, I2C address `0x38` |
| BH1750 | Ambient light, I2C address `0x23` (ADDR pin low; `0x5C` when ADDR is high) |
| TM1637 module | 4-digit 7-segment, ~0.36", with a 2-dot colon between the digit pairs |

### Wiring

| Signal | D1 Mini pin | GPIO | Connects to |
| --- | --- | --- | --- |
| I2C SDA | D2 | GPIO4 | AHT20 `SDA`, BH1750 `SDA` |
| I2C SCL | D1 | GPIO5 | AHT20 `SCL`, BH1750 `SCL` |
| TM1637 CLK | D6 | GPIO12 | TM1637 `CLK` |
| TM1637 DIO | D5 | GPIO14 | TM1637 `DIO` |
| 3V3 | 3V3 | - | AHT20 `VCC`, BH1750 `VCC`, TM1637 `VCC` |
| GND | G | - | all `GND`, plus BH1750 `ADDR` (selects `0x23`) |

Notes:

* **Power the TM1637 module from 3.3 V, not 5 V.** The module works on both, but
  at 5 V its logic high threshold is above what an ESP8266 output reliably
  drives. At 3.3 V it is slightly dimmer and completely reliable.
* Keep the I2C wires short, or lower `i2c: frequency:` to `10kHz`.
* The AHT20 and the BH1750 share the same bus - two different addresses, no
  conflict.

---

## Display behaviour

| Situation | What you see |
| --- | --- |
| Normal | `23:41` - temperature 23 °C, humidity 41 %RH |
| Colon | One dot lit, the two dots swap every `colon_blink_period_ms` (default 800 ms) |
| Humidity < min or > max | Digits 3-4 go dark and back on every `humidity_blink_period_ms` (default 800 ms). Digits 1-2 keep showing the temperature. The colon keeps alternating. |
| Temperature -1 .. -9 °C | `-5:41` |
| Temperature <= -9.5 °C | `LO:41` |
| Temperature >= 99.5 °C | `HI:41` |
| Humidity >= 99.5 %RH | `23:HI` |
| Sensor read fails (NaN) | `--` in the affected digit pair |

The colon dots and the digit segments are separate LEDs on the module, so
blanking the humidity digits never blanks the colon dot.

Auto-dimming, where `lux` is the BH1750 reading:

```
intensity = 1 + 6 * clamp((lux - dim_lux_low) / (dim_lux_high - dim_lux_low), 0, 1) ^ dim_gamma
```

`intensity` is the TM1637 brightness, `1` = dimmest and `7` = brightest, so the
display is never fully dark. Setting **Dim Lux High** <= **Dim Lux Low** pins it
to full brightness.

---

## MQTT interface

`mqtt.topic_prefix` is `humidity-temp`, so every entity is reachable at
`humidity-temp/<domain>/<object_id>/{state,command}`. Entity object IDs are
derived from the entity names; the authoritative list is what MQTT discovery
publishes - check it with:

```bash
mosquitto_sub -h <broker> -u <user> -P <pass> -t 'homeassistant/#' -v
```

### Published state

| Entity | State topic |
| --- | --- |
| Temperature (°C) | `humidity-temp/sensor/temperature/state` |
| Humidity (%RH) | `humidity-temp/sensor/humidity/state` |
| Illuminance (lx) | `humidity-temp/sensor/illuminance/state` |
| Humidity in comfort range (on/off) | `humidity-temp/binary_sensor/humidity_in_comfort_range/state` |
| Online / offline (LWT) | `humidity-temp/status` |

### Runtime configuration

Publish to `<topic>/command` (in ESPHome 2026.x the old `/set` topics are
ignored), for example:

```bash
mosquitto_pub -h <broker> -u <user> -P <pass> \
  -t 'humidity-temp/number/humidity_min/command' -m '45'
```

| Entity | Command topic | Default | Range | Step | Effect |
| --- | --- | --- | --- | --- | --- |
| Humidity Min | `humidity-temp/number/humidity_min/command` | 40 | 0-100 | 1 | Lower edge of the comfort band (%RH) |
| Humidity Max | `humidity-temp/number/humidity_max/command` | 60 | 0-100 | 1 | Upper edge of the comfort band (%RH) |
| Dim Lux Low | `humidity-temp/number/dim_lux_low/command` | 10 | 1-100 | 1 | Lux at which the display is at its dimmest |
| Dim Lux High | `humidity-temp/number/dim_lux_high/command` | 300 | 10-2000 | 10 | Lux at which the display is at its brightest |
| Dim Gamma | `humidity-temp/number/dim_gamma/command` | 1.0 | 0.2-3.0 | 0.1 | Dimming curve coefficient; 1.0 = linear, > 1 dims earlier |

All five values are stored in flash and survive a reboot and an OTA update. A
stored value always wins over the `initial_value` in the YAML.

---

## Build, flash, update

No local venv is needed - `uvx` fetches ESPHome on demand.

```bash
# 1. Check the configuration (fast, no compiler)
uvx esphome config humidity-temp.yaml

# 2. Compile only
uvx esphome compile humidity-temp.yaml

# 3. First flash over USB - adjust the port
uvx esphome run humidity-temp.yaml --device COM5

# 4. Every later update over the air
uvx esphome run humidity-temp.yaml
```

`secrets.yaml` (gitignored) must provide:

```yaml
wifi_ssid: "..."
wifi_password: "..."
mqtt_broker: "192.168.1.10"
mqtt_username: "..."
mqtt_password: "..."
ota_password: "..."
```

`ota_password` doubles as the fallback access point password, so it must be at
least 8 characters.

---

## Calibrating the colon dots

The two colon dots are ordinary decimal-point LEDs of two digits, but *which*
digit's decimal-point bit drives which dot depends on the module. The YAML
exposes that mapping as two substitutions:

```yaml
colon_dot_upper_digit: "1"   # digit index 0..3, 0 = leftmost digit
colon_dot_lower_digit: "2"
```

The colon of a 4-digit module sits between digit 2 and digit 3, so the two
candidates are index `1` and index `2`. To find out which is which:

1. Set **both** substitutions to the same value, e.g. `"1"`, and flash.
   With both equal the dot no longer alternates, it stays lit - easy to spot.
2. Note which physical LED lights up: the upper colon dot, the lower colon dot,
   or the decimal point of a digit.
3. Repeat for `"0"`, `"2"` and `"3"`.
4. Put the index that lights the **upper** dot into `colon_dot_upper_digit` and
   the one that lights the **lower** dot into `colon_dot_lower_digit`, then
   flash the final configuration.

If a dot ends up on a digit's own decimal point instead of the colon, the module
simply does not expose that dot separately - pick the two indices that do.

---

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `esptool` fails with `Cannot configure port ... PermissionError(13, ... 31)` | CH340 driver issue on Windows, not a permission problem. Install CH341SER **3.5.2019.1** and stop Windows Update from re-upgrading the driver. |
| Nothing on the display, no `display.tm1637` log line | Check CLK/DIO are not swapped, and that the module is powered. |
| Display shows garbage or flickers | Power the module from 3.3 V and shorten the wires. |
| BH1750 missing in the boot scan (with `i2c: scan: true`; addresses are logged) | ADDR pin must be tied to GND for `0x23`. Without it the address is `0x5C`. |
| AHT20 present but humidity is constantly NaN | Try `variant: AHT10`. Some chips labelled AHT10 need the AHT20 driver and vice versa. |
| I2C read errors after long cable runs | Lower `i2c: frequency:` to `10kHz`. |
| Humidity digits blink although the air feels fine | Humidity is outside the comfort band - check the `Humidity Min`/`Humidity Max` values. |
| Display is very dim | The room is dark and the auto-dimming is working. Lower `Dim Lux Low`/`Dim Lux High`, or set `Dim Lux High` <= `Dim Lux Low` for full brightness. |
| Nothing in Home Assistant | MQTT discovery is enabled; make sure the broker credentials in `secrets.yaml` are correct and check the retained `homeassistant/#` topics. |

To watch the auto-dimming decisions, set `logger: level: DEBUG`; the display logs
its intensity whenever it changes.

---

## Repository layout

| File | Purpose |
| --- | --- |
| `humidity-temp.yaml` | The complete device configuration. |
| `secrets.yaml` | Credentials. Gitignored - never commit it. |
| `README.md` | This document. |
| `LICENSE` | MIT. |

---

## AI metadata (for agents)

Machine-readable summary. Keep in sync with `humidity-temp.yaml`.

* Device: `humidity-temp`, friendly name `Humidity Temp Monitor`, board `d1_mini`.
* Platform: `esp8266`; for `esp32` use `board: esp32dev` and re-pick I2C pins.
* MQTT: `topic_prefix: humidity-temp`, `discovery: true`,
  `discovery_prefix: homeassistant`, LWT `humidity-temp/status`.
  Command topics are `<prefix>/<domain>/<object_id>/command` - `/set` is dead.
* I2C: SDA `GPIO4` (D2), SCL `GPIO5` (D1), 100 kHz.
* TM1637: CLK `GPIO12` (D6), DIO `GPIO14` (D5), `update_interval: 250ms`.
* Entities and IDs: `temperature`, `humidity` (aht10, `variant: AHT20`, 10 s);
  `illuminance` (bh1750, 10 s); `humidity_in_comfort_range` (template
  binary sensor, publish-only); `humidity_min` 40, `humidity_max` 60,
  `dim_lux_low` 10, `dim_lux_high` 300, `dim_gamma` 1.0 (all template numbers,
  `optimistic` + `restore_value` + `mode: BOX`).
* Display buffer: 4 raw TM1637 bytes, `bit0=A ... bit6=G, bit7=decimal point`.
  Digits 1-2 = temperature, digits 3-4 = humidity. Glyphs used: `0`-`9`
  (`0x3F, 0x06, 0x5B, 0x4F, 0x66, 0x6D, 0x7D, 0x07, 0x7F, 0x6F`), blank `0x00`,
  `-` `0x40`, `H` `0x76`, `I` `0x06`, `L` `0x38`, `O` `0x3F`.
* Formatting: `NaN` -> `--`; temp `>99` -> `HI`, `<-9` -> `LO`, `-9..-1` ->
  `-` + digit, `0..99` -> two digits; humidity clamped to `>= 0`, `>99` -> `HI`.
* Colon: `buf[colon_dot_upper_digit] |= 0x80` when `(millis()/800) % 2 == 0`,
  otherwise `buf[colon_dot_lower_digit] |= 0x80`.
* Humidity blink: blank `buf[2]` and `buf[3]` when humidity is outside
  `[humidity_min, humidity_max]` and `(millis()/800) % 2 == 1`.
* Auto-dim: `intensity = clamp(round(1 + 6 * clamp((lux-low)/(high-low),0,1)^gamma), 0, 7)`;
  skipped while `lux` is `NaN`; `high <= low` forces `7`.
* Constants: `display_update_ms 250`, `colon_blink_period_ms 800`,
  `humidity_blink_period_ms 800`, `colon_dot_upper_digit 1`,
  `colon_dot_lower_digit 2`.
* Requires ESPHome >= 2026.4 for `TM1637Display::set_buffer()`; developed
  against 2026.8.2.

---

## References

* [ESPHome TM1637 display](https://esphome.io/components/display/tm1637.html)
* [ESPHome AHT10 / AHT20 sensor](https://esphome.io/components/sensor/aht10.html)
* [ESPHome BH1750 sensor](https://esphome.io/components/sensor/bh1750.html)
* [ESPHome MQTT client](https://esphome.io/components/mqtt.html)
* [ESPHome OTA updates](https://esphome.io/components/ota.html)

A sibling project, [bathvent-esphome](https://github.com/dimitri-rebrikov/bathvent-esphome),
uses the same conventions (MQTT-only, template numbers for runtime
configuration, `uvx esphome` builds).

## License

MIT - see [LICENSE](LICENSE).
