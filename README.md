# humidity-temp-esphome

Small ESPHome node that measures temperature and relative humidity, shows both
on a 4-digit 7-segment display and publishes everything to MQTT.

```
   [2][3] :  [4][1]
 temperature  humidity %RH
```

* **Temperature** on digits 1-2, **humidity** on digits 3-4 (both rounded to integers).
* The **two colon dots blink** every 800 ms. Without WiFi they blink **fast**
  (150 ms) as an offline indicator.
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
| Protocols | Native ESPHome API (encrypted, Home Assistant) and MQTT. No web server. WiFi SSID/password, MQTT credentials and the API key come from `secrets.yaml`. |

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
| Colon | Both dots sit on one decimal-point line, so they switch together: `colon_mode` `0` = permanently on, `1` (default) = blink every `colon_blink_period_ms` (800 ms) |
| No WiFi | The colon blinks fast, period `wifi_blink_period_ms` (150 ms), and overrides `colon_mode`. Back to normal as soon as WiFi associates again. |
| Humidity < min or > max | Digits 3-4 go dark and back on every `humidity_blink_period_ms` (default 800 ms). Digits 1-2 keep showing the temperature. The colon keeps its pattern. |
| Temperature -1 .. -9 °C | `-5:41` |
| Temperature <= -9.5 °C | `LO:41` |
| Temperature >= 99.5 °C | `HI:41` |
| Humidity >= 99.5 %RH | `23:HI` |
| Sensor read fails (NaN) | `--` in the affected digit pair |

The colon dots and the digit segments are separate LEDs on the module, so
blanking the humidity digits never blanks the colon dot.

Auto-dimming, where `lux` is the BH1750 reading:

```
intensity = dim_min_intensity
          + (7 - dim_min_intensity) * clamp((lux - dim_lux_low) / (dim_lux_high - dim_lux_low), 0, 1) ^ dim_gamma
```

`intensity` is the TM1637 brightness index, `0` to `7`. It selects the LED pulse
width and is **not** a linear scale:

| `intensity` | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Pulse width | 1/16 | 2/16 | 4/16 | 10/16 | 11/16 | 12/16 | 13/16 | 14/16 |

**Intensity `0` is a valid level** - the dimmest pulse width, still illuminated,
not "off". Switching the display off is a separate control bit that this
configuration does not use. Because the hardware steps are non-linear (it jumps
from 4/16 to 10/16 between index 2 and 3), `dim_gamma` shapes an index, not the
perceived brightness.

**Dim Min Intensity** sets the dark-room floor: `1` (the default) keeps the
display softly visible, `0` drops to the dimmest duty. Setting **Dim Lux High**
<= **Dim Lux Low** pins it to full brightness.

The value actually applied is published as **Display Brightness**, `0` to `7`,
whenever it changes.

---

## Temperature calibration

The AHT20 reading is off by a constant in most builds - self-heating, the
enclosure, cable routing. **Temperature Offset** shifts it: `-10` to `+10` °C in
`0.1` steps, default `0`.

```bash
mosquitto_pub -h <broker> -u <user> -P <pass> \
  -t 'humidity-temp/number/temperature_offset/command' -m '-1.5'
```

The offset is a sensor filter on the AHT20 temperature, not a display trick, so
the panel, the published MQTT state and Home Assistant all show the same
corrected value - one number to trust, no second source of truth. A new offset
takes effect on the next sensor update, so allow up to `10 s`.

Humidity is not calibrated.

### The AHT20's own calibration

The AHT20 does self-calibrate, but **only at boot**, and ESPHome gives you no way
to trigger or repeat it. On every start-up the driver
(`esphome/components/aht10/aht10.cpp`) does:

1. Soft reset (`0xBA`), then waits 30 ms.
2. Sends the calibration/initialisation command: `0xBE 0x08 0x00` for the AHT20
   variant, `0xE1 0x08 0x00` for AHT10.
3. Polls the status byte while the busy bit (`0x80`) is set - at most 10 times
   with 5 ms between reads, so roughly 50 ms.
4. Requires `(status & 0x68) == 0x08`: mode bits `[6:5]` = `00` (normal) **and**
   bit 3, the calibrated flag. Anything else logs `Initialization failed` and
   marks the component failed, leaving it dead until the next reboot.

Consequences:

* There is **no runtime auto-calibration** - no entity, no option, no periodic
  re-calibration. For a constant error the only lever is **Temperature Offset**
  above, or a sensor filter.
* Calibration is **re-run on every boot and every reflash**, which is the only
  way to repeat it.
* If the chip never raises the calibrated flag, or stays busy past ~50 ms, the
  temperature and humidity entities go unavailable and the display shows `--`.
* Independent drivers treat that `0xBE` write as optional. Adafruit's AHTX0
  library sends it with the comment *"may not 'succeed' on newer AHT20s"* and
  ignores the return value, while ESPHome fails the whole component if the write
  is not acknowledged. A module that works on Arduino can therefore still fail
  here - `variant: AHT10` is worth trying in that case.
* AHT20 humidity can never legitimately read exactly `0 %RH`. ESPHome publishes
  `NaN` and logs `Invalid humidity reading (0%)` when it sees that.

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
| Display brightness (0-7) | `humidity-temp/sensor/display_brightness/state` |
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
| Temperature Offset | `humidity-temp/number/temperature_offset/command` | 0 | -10..10 | 0.1 | Calibration added to the AHT20 temperature (°C) |
| Humidity Min | `humidity-temp/number/humidity_min/command` | 40 | 0-100 | 1 | Lower edge of the comfort band (%RH) |
| Humidity Max | `humidity-temp/number/humidity_max/command` | 60 | 0-100 | 1 | Upper edge of the comfort band (%RH) |
| Dim Lux Low | `humidity-temp/number/dim_lux_low/command` | 10 | 1-100 | 1 | Lux at which the display is at its dimmest |
| Dim Lux High | `humidity-temp/number/dim_lux_high/command` | 300 | 10-2000 | 10 | Lux at which the display is at its brightest |
| Dim Gamma | `humidity-temp/number/dim_gamma/command` | 1.0 | 0.2-3.0 | 0.1 | Dimming curve coefficient; 1.0 = linear, > 1 dims earlier |
| Dim Min Intensity | `humidity-temp/number/dim_min_intensity/command` | 1 | 0-7 | 1 | Brightness in a fully dark room; `0` = dimmest pulse width (1/16), not off |

All seven values are stored in flash and survive a reboot and an OTA update. A
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

## Colon dots

On the reference module both colon dots are wired to the **same**
decimal-point line, so they always light up and go dark together. Steering them
separately is not possible - that is a wiring property of the module, not a
software setting.

`colon_dot_digit` says which digit's decimal-point bit drives the colon, and
`colon_mode` picks the pattern:

| `colon_mode` | Behaviour |
| --- | --- |
| `0` | Both dots permanently on |
| `1` (default) | Both dots blink together, period `colon_blink_period_ms` |

Independently of `colon_mode`, the colon blinks fast (period
`wifi_blink_period_ms`, default 150 ms) while `wifi::global_wifi_component`
reports **not connected**. That is the only status the node can signal without a
network, so it takes precedence - including over `colon_mode: 0`. Connection is
decided by WiFi association, not by MQTT or the API being reachable.

Measured on the reference module: decimal-point index **1** (the second digit
from the left) lights both dots. Module vendors wire this differently, so
`colon_dot_digit` is a substitution rather than a hard-coded constant.

### Calibrating the colon

If you swap the display, or the dots stay dark, re-measure with the built-in
diagnostic. It ignores the sensors and, for 2 s per step, shows a digit that
**names** the decimal-point index currently being lit:

```bash
uvx esphome -s colon_diagnostic true run humidity-temp.yaml
```

Watch which LED lights up while each digit is on screen:

| Digit shown | Index lit | What you should see |
| --- | --- | --- |
| `1` | 0 | decimal point of digit 1 |
| `2` | 1 | the colon dots, or nothing |
| `3` | 2 | the colon dots, or nothing |
| `4` | 3 | decimal point of digit 4 |

Note the **digit** that was on screen while the colon dots lit up - the index is
that digit minus one - and put it into `colon_dot_digit`. Digits whose step
lights nothing just mean the module has no LED on that line; that is normal for
clock modules, which often omit the per-digit decimal points.

Leave `colon_diagnostic: "false"` in the YAML, so a later build cannot silently
ship the test pattern. The `-s` override is not stored anywhere and simply
disappears on the next build.

---

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `esptool` fails with `Cannot configure port ... PermissionError(13, ... 31)` | CH340 driver issue on Windows, not a permission problem. Install CH341SER **3.5.2019.1** and stop Windows Update from re-upgrading the driver. |
| Nothing on the display, no `display.tm1637` log line | Check CLK/DIO are not swapped, and that the module is powered. |
| Display shows garbage or flickers | Power the module from 3.3 V and shorten the wires. |
| BH1750 missing in the boot scan (with `i2c: scan: true`; addresses are logged) | ADDR pin must be tied to GND for `0x23`. Without it the address is `0x5C`. |
| AHT20 present but humidity is constantly NaN | Try `variant: AHT10`. Some chips labelled AHT10 need the AHT20 driver and vice versa. |
| `Initialization failed` at boot | The AHT20 did not report the calibrated flag (status bit 3) or is not in normal mode (`[6:5]` = `00`). Reset the board; if it persists, check wiring and power, and try `variant: AHT10`. |
| `Initialization timed out` at boot | The sensor stayed busy longer than the ~50 ms the driver allows. Same checks as above. |
| `Invalid humidity reading (0%)` in the log | The AHT20 reported exactly `0 %RH`, which it cannot really measure, so the reading is discarded. |
| I2C read errors after long cable runs | Lower `i2c: frequency:` to `10kHz`. |
| Humidity digits blink although the air feels fine | Humidity is outside the comfort band - check the `Humidity Min`/`Humidity Max` values. |
| Display is very dim | The room is dark and the auto-dimming is working. Set `Dim Min Intensity` to `0` for the dimmest level, lower `Dim Lux Low`/`Dim Lux High`, or set `Dim Lux High` <= `Dim Lux Low` for full brightness. |
| `Display Brightness` stays `unknown` | The BH1750 never returns a value, so the auto-dim never runs. The log says so at WARNING level. See the BH1750 row above. |
| `Display Brightness` is stuck at `7` and no setting helps | Read the `INFO` log line - it prints every input. Setting `Dim Lux High` <= `Dim Lux Low` deliberately forces full brightness; otherwise raise `Dim Lux High` or lower `Dim Lux Low`. |
| Nothing in Home Assistant | MQTT discovery is enabled; make sure the broker credentials in `secrets.yaml` are correct and check the retained `homeassistant/#` topics. |

Whenever the brightness changes, the device logs the result **and every input
that produced it**, at `INFO` level, so the configured `logger: level: INFO` is
enough:

```
[I][humidity_temp:...] Display intensity 3 (120.0 lx, lux_low 10, lux_high 300, gamma 1.00, min 1)
```

Watch it live with:

```bash
uvx esphome logs humidity-temp.yaml
```

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
  `illuminance` (bh1750, 10 s); `display_brightness` (template sensor,
  `update_interval: never`, published from the display lambda);
  `humidity_in_comfort_range` (template binary sensor, publish-only);
  `humidity_min` 40, `humidity_max` 60, `temperature_offset` 0,
  `dim_lux_low` 10, `dim_lux_high` 300, `dim_gamma` 1.0,
  `dim_min_intensity` 1 (all template numbers, `optimistic` + `restore_value` +
  `mode: BOX`).
* Display buffer: 4 raw TM1637 bytes, `bit0=A ... bit6=G, bit7=decimal point`.
  Digits 1-2 = temperature, digits 3-4 = humidity. Glyphs used: `0`-`9`
  (`0x3F, 0x06, 0x5B, 0x4F, 0x66, 0x6D, 0x7D, 0x07, 0x7F, 0x6F`), blank `0x00`,
  `-` `0x40`, `H` `0x76`, `I` `0x06`, `L` `0x38`, `O` `0x3F`.
* Formatting: `NaN` -> `--`; temp `>99` -> `HI`, `<-9` -> `LO`, `-9..-1` ->
  `-` + digit, `0..99` -> two digits; humidity clamped to `>= 0`, `>99` -> `HI`.
* Colon: `buf[colon_dot_digit] |= 0x80` every refresh when `colon_mode == 0`,
  and only on the even phase of `(millis()/800) % 2` when `colon_mode == 1`.
  Both colon dots hang on that one decimal-point line - they cannot be lit
  separately. While `wifi::global_wifi_component->is_connected()` is false the
  phase uses `wifi_blink_period_ms` (150 ms) and `colon_mode` is ignored. `colon_diagnostic: true` overrides everything and shows a digit
  naming the index being lit (`probe = (millis()/2000) % 4`, digit `probe + 1`,
  with `buf[probe] |= 0x80`).
* Temperature calibration: `sensor::OffsetFilter` on the AHT20 `temperature`
  (`filters: - offset: !lambda return id(temperature_offset).state;`), so the
  display, MQTT state and HA all report the corrected value. Range -10..10 °C in
  0.1 steps, applied on the next 10 s sensor update, not instantly.
* AHT20 calibration is **boot-only and not runtime-accessible**: ESPHome sends
  soft reset `0xBA`, then `0xBE 0x08 0x00` (`0xE1 0x08 0x00` for AHT10), polls
  busy (`0x80`) up to 10x5 ms, then requires `(status & 0x68) == 0x08` (mode
  `00`, calibrated bit 3) or it calls `mark_failed()`. No option, entity or
  MQTT command can re-trigger it; only a reboot/reflash. Adafruit's AHTX0 driver
  sends the same command but ignores a non-ACK ("may not 'succeed' on newer
  AHT20s"), ESPHome does not - so try `variant: AHT10` if init fails.
  Humidity of exactly `0 %RH` is treated as invalid and published as `NaN`.
* Humidity blink: blank `buf[2]` and `buf[3]` when humidity is outside
  `[humidity_min, humidity_max]` and `(millis()/800) % 2 == 1`.
* Auto-dim: the four tuning inputs are sanitised first - a `NaN` or nonsense
  parameter is replaced by its default instead of silently falling through to
  full brightness. Then `floor = clamp(dim_min_intensity, 0, 7)`,
  `intensity = clamp(round(floor + (7-floor) * clamp((lux-low)/(high-low),0,1)^gamma), 0, 7)`;
  skipped while `lux` is `NaN` (logged once at WARNING). `high <= low`
  deliberately forces `7`. Intensity `0` = 1/16 pulse width (dimmest, still lit),
  `7` = 14/16; the hardware duty table is non-linear:
  `1/16, 2/16, 4/16, 10/16, 11/16, 12/16, 13/16, 14/16` for indices `0..7`.
  Published to `display_brightness` and logged at `INFO` only when it changes,
  never on every refresh. Off is a separate control bit (`set_on(false)`) and is
  not used.
* Constants: `display_update_ms 250`, `colon_blink_period_ms 800`,
  `wifi_blink_period_ms 150`, `humidity_blink_period_ms 800`, `colon_mode 1`,
  `colon_diagnostic false`, `colon_dot_digit 1` (measured on the reference
  module).
* `esp8266: restore_from_flash: true` - the default (`false`) keeps restored
  values in RTC memory, which loses them on a real power cut (reboot is fine).
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
