# Inkplate 2 Weather Display

A tiny always-on e-paper display that shows an outdoor temperature from
**Home Assistant**. Built with [ESPHome](https://esphome.io) on a
[Soldered Inkplate 2](https://soldered.com/product/soldered-inkplate-2/),
3D-printable case and stand included.

No cloud, no app, no polling — Home Assistant pushes the value over the
native ESPHome API, and the panel only redraws when the temperature
actually moves by a full degree.

![Photo placeholder — drop a picture of your build in docs/](docs/photo.jpg)

---

## Why this exists

The Inkplate 2 is a lovely little board: an ESP32, a 2.13" three-colour
e-paper panel and a USB-C port, all on 65 × 35 mm. It's a natural fit for
an ambient display you glance at rather than interact with.

Getting it working in ESPHome turned out to be less obvious than expected,
so this repo contains a configuration that actually works, plus a
[troubleshooting section](#troubleshooting) documenting every wall I ran
into. If you're building something similar, that section will probably
save you an afternoon.

## Features

- **Large, readable temperature** in whole degrees — 64 pt, visible across a room
- **Automatic font shrink** for sub-zero values, so `-12` still fits the 104 px width
- **Humidity and timestamp** on the same screen
- **Redraws only on change** — a full refresh flashes the panel for ~20 seconds,
  so it happens only when the temperature moved by at least 1 K
- **Portrait layout** that adapts to whichever way round you mount it
- **Manual refresh button** exposed to Home Assistant

## Hardware

| Item | Notes |
|---|---|
| Soldered Inkplate 2 | 212 × 104 px, black / white / red e-paper |
| USB-C cable + power supply | Any phone charger will do |
| 3D printed case | `hardware/case-top.stl` + `hardware/case-bottom.stl` |
| 3D printed stand | `hardware/stand.stl` |

No soldering and no wiring — everything is on the board.

### Printing the case

Standard PLA settings work fine. 0.2 mm layer height, 15 % infill,
no supports needed if you print the shells face-down. The stand is a
separate part; print it flat on the bed.

The case parts come from Soldered's own design, the stand is a
community remix.

## Installation

### 1. Add your secrets

Copy `secrets.yaml.example` to your ESPHome `secrets.yaml` and fill in
the values, or add the missing keys to an existing one:

```yaml
wifi_ssid: "YourNetwork"
wifi_password: "..."
ap_password: "..."
api_encryption_key: "..."   # generate at https://esphome.io/components/api
ota_password: "..."
```

### 2. Point it at your sensors

Open `esphome/inkplate2-weather.yaml` and edit the two entity IDs at
the top:

```yaml
substitutions:
  temp_entity: sensor.your_outdoor_temperature
  hum_entity: sensor.your_outdoor_humidity
```

If you don't have a humidity sensor, delete the `h_out` sensor block and
the humidity section of the lambda.

### 3. Flash

First flash has to happen over USB, after that OTA works:

```bash
esphome run esphome/inkplate2-weather.yaml
```

The first build takes a while — ESPHome pulls the Soldered display
component from GitHub and compiles it.

> **Leave the board powered for at least a minute after flashing.**
> ESPHome's safe mode only marks the new firmware as valid after a
> while. Unplug it too early and the bootloader rolls back to the
> previous version, and you'll spend a confused twenty minutes wondering
> why your changes did nothing.

### 4. Adopt it in Home Assistant

Go to **Settings → Devices & Services**. The device appears under
discovered integrations and has to be added once. Until you do, the
panel shows placeholders — without an API connection there is neither a
temperature nor a clock.

### 5. Orientation

If the display comes out upside down, change `rotation: 180` to
`rotation: 0` in the display block. The layout is built from
`get_width()` / `get_height()`, so it adapts by itself.

## Customising

| What | Where |
|---|---|
| Redraw threshold | `delta: 1.0` in the temperature sensor's filters |
| Decimal places | `"%.0f"` → `"%.1f"` in the lambda (drop the font size to ~38) |
| Header text | `"AUSSEN"` in the lambda |
| Font sizes | `f_temp` (64 pt) and `f_temp_neg` (42 pt) |
| Frost highlight | The panel is three-colour — `Color(255, 0, 0)` gives you red |

### A note on red

The panel can show black, white and red. Using red costs nothing extra:
the 15-second refresh time is a fixed waveform in the panel controller,
and it runs the full sequence whether or not your image contains red
pixels. The variable `RED` is already defined in the lambda if you want
to highlight sub-zero temperatures.

## Troubleshooting

Everything below actually happened while building this.

### The panel never updates, and the log says `Display already in state POWER_OFF`

The built-in `epaper_spi` platform lists `inkplate2` as a supported
model, and it compiles, and the lambda runs without errors — but nothing
ever reaches the panel. Manual `component.update` calls fail with
`Display already in state POWER_OFF`.

**Fix:** use Soldered's own component instead. That's what the
`external_components` block in the config is for:

```yaml
external_components:
  - source: github://SolderedElectronics/Soldered-Inkplate-ESPHome
    components: [inkplate_spi]

display:
  - platform: inkplate_spi     # not epaper_spi
    model: inkplate2
```

With the manufacturer's driver you get a proper debug trace:
`update #1 — full refresh`, states 0 through 9, `DTM1 BW`, `DTM2 RED`,
`panel deep sleep`. A complete refresh takes about 19 seconds.

### Characters are missing — `Codepoint 0x0000002d not found in font`

`0x2D` is the minus sign. Writing glyphs as a plain string can be parsed
as a *single multi-character glyph* rather than a set of individual
characters. Use the list form:

```yaml
glyphs: ["-", ".", "0", "1", "2"]     # not: glyphs: "-.012"
```

The degree sign needs the same treatment — it isn't reliably part of the
default character set.

### The display shows `--` and `01.01. 01:00`

That's Unix epoch zero. The time source is `platform: homeassistant`, so
if the clock is wrong the API connection isn't up, which means there's no
sensor value either. Both symptoms have one cause.

Check the log for this pair:

```
[W][component:290]: api set Warning flag: waiting for client connection
[W][component:313]: api cleared Warning flag        <- this one must appear
```

If the second line never comes, the device hasn't been adopted in Home
Assistant yet.

### First values appear only after a few minutes

The panel draws once at boot, which happens a second or two before Home
Assistant connects. A redraw scheduled too soon afterwards hits the
still-running refresh and gets skipped:

```
[W][inkplate_spi:038]: update() skipped — display busy (state 7)
```

Hence the 20-second delay in the `refresh` script — that's longer than a
full panel refresh, so jobs never overlap.

### Log says the firmware is older than what you just flashed

```
[W][safe_mode:094]: OTA rollback detected! Rolled back from partition 'app1'
[W][safe_mode:094]:  The device reset before the boot was marked successful
```

You power-cycled or reset the board too soon after flashing. Flash again
and leave it alone for a minute. Always sanity-check the `compiled on`
timestamp in the log before debugging anything else.

### `The configured log level for X (DEBUG) must not be less severe than the global log level (INFO)`

Per-component log levels can't be more verbose than the global one. Set
`logger: level: DEBUG` globally.

## Battery operation

This configuration is **USB powered on purpose**. Running it on a
battery is possible but needs deep sleep, and the numbers aren't
flattering.

The problem is the panel: the Inkplate 2 is a three-colour display with a
**15-second refresh and no partial update support**. Every single change
is a full refresh, because the red pigment particles are heavier and need
long waveform pulses. That's roughly 0.2 mAh per redraw on top of 0.2 mAh
for the WiFi connect.

| Wake interval | Battery for ~3 months |
|---|---|
| 10 min | ~6000 mAh |
| 15 min | ~4000 mAh |
| 30 min | ~2000 mAh |

A monochrome panel refreshes in about 2 seconds instead of 15, which
roughly halves the budget. If battery life matters more than the red
pigment, that's the better starting point.

Deep sleep also inverts the logic: a sleeping device can't react to a
temperature change, so it has to wake on a fixed schedule, compare
against a value stored in a `global` with `restore_value: true`, and only
then decide whether to redraw. And you need an OTA window — a
template switch calling `deep_sleep.prevent` with
`restore_mode: RESTORE_DEFAULT_OFF` — or you'll lock yourself out of the
device.

## Repository layout

```
esphome/
  inkplate2-weather.yaml    ESPHome configuration
hardware/
  case-top.stl              Case, front shell
  case-bottom.stl           Case, back shell
  stand.stl                 Desk stand
docs/                       Put your build photos here
secrets.yaml.example        Template for ESPHome secrets
```

## Credits

- [Soldered Electronics](https://soldered.com) for the Inkplate 2 and
  their [ESPHome component](https://github.com/SolderedElectronics/Soldered-Inkplate-ESPHome)
- The [ESPHome](https://esphome.io) and
  [Home Assistant](https://www.home-assistant.io) projects

Case STLs are Soldered's own design for the Inkplate 2; the stand is a
community remix. Check their original licensing before redistributing.

## License

MIT — see [LICENSE](LICENSE). Applies to the configuration and
documentation in this repository, not to the third-party STL files.
