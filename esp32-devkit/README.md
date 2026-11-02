# ESP32 DevKit build (30-pin, USB-C) with DHT22

This build replaces the project's recommended FireBeetle 2 ESP32-E with a
generic 30-pin ESP32 DevKit (ESP-WROOM-32 module, USB-C) and the BME280 with a
DHT22. It is currently powered over USB only; battery operation is described
under [Battery (not connected yet)](#battery-not-connected-yet).

## Hardware

| Part | Notes |
|---|---|
| ESP32 DevKit, 30 pins, USB-C | ESP-WROOM-32 module, pins labelled `Dxx` = GPIOxx |
| DESPI-C02 e-paper adapter | RESE switch set to **0.47** |
| 7.5" black/white e-paper panel, 800×480 | `DISP_BW_V2` in `config.h` |
| DHT22 temperature/humidity sensor | 3-pin module or bare 4-pin sensor |

## Wiring

![Connections](wiring_connections.png)

![Used pins on the DevKit, USB-C at the top](wiring_pinout.png)

| From | To ESP32 pin |
|---|---|
| DESPI BUSY | D14 |
| DESPI RES | D21 |
| DESPI D/C | D22 |
| DESPI CS | D13 |
| DESPI SCK | D18 |
| DESPI SDI | D23 |
| DESPI GND | GND |
| DESPI 3.3V | 3V3 |
| DHT22 VCC | D4 |
| DHT22 DATA | D17 |
| DHT22 GND | GND |

- The DevKit has a GND pin on each side (2nd from the USB-C end); the DHT22
  and the DESPI can each use one.
- The firmware powers the DHT22 from D4 only while reading it.
- A bare 4-pin DHT22 (grille facing you, pins down: VCC, DATA, unused, GND)
  needs a 10 kΩ pull-up between DATA and the sensor's VCC (D4). Connecting the
  pull-up to 3V3 instead leaks current through the unpowered sensor during
  deep sleep. 3-pin DHT22 modules include the pull-up.
- VIN stays unconnected.

The diagrams are rendered from `wiring_connections.svg` and `wiring_pinout.svg`:

```sh
rsvg-convert -z 2 -o wiring_connections.png wiring_connections.svg
rsvg-convert -z 2 -o wiring_pinout.png wiring_pinout.svg
```

## Firmware changes

### `platformio/platformio.ini`
- New `[env:esp32dev]` environment (`board = esp32dev`, 80 MHz CPU,
  `huge_app.csv` partitions), set as `default_envs`.
- The `esp32dev` board variant does not define `LED_BUILTIN`, which
  `disableBuiltinLED()` uses. The environment adds `-DLED_BUILTIN=2`, the
  DevKit's blue LED.
- New library dependency `adafruit/DHT sensor library @ 1.4.7`.

### `platformio/include/config.h`
- New sensor option `SENSOR_DHT22`, selected. `SENSOR_BME280` is commented out.
- The sensor check accepts exactly one of `SENSOR_BME280`, `SENSOR_BME680`,
  `SENSOR_DHT22`.
- `BATTERY_MONITORING` is `0` while no battery is connected. With it enabled
  and nothing on the ADC pin, the firmware reads ~0 V, treats the battery as
  critically low and hibernates without updating.
- Metric units: Celsius, km/h, hPa, kilometres, and millimetres for daily
  precipitation. The hourly graph shows probability of precipitation.
- `LOCALE de_DE`: German display text, day/month names, and `DD.MM.YY` dates
  in the refresh-time line. Weather descriptions come from OpenWeatherMap in
  German.

### `platformio/src/config.cpp`
- `PIN_BAT_ADC = 35` (D35). The FireBeetle 2 ESP32-E uses `A2`.
- New `PIN_DHT_DATA = 17` (D17). `PIN_BME_PWR` (D4) powers whichever sensor is
  selected.
- Wi-Fi credentials and location are set locally.
- `OWM_APIKEY` comes from `OWM_APIKEY_SECRET` in `platformio/include/secrets.h`,
  which is gitignored. To set it up on a fresh checkout, copy
  `platformio/include/secrets.h.example` to `platformio/include/secrets.h` and
  enter the key. Without the file the build stops with an error naming it.
- `TIMEZONE = "CET-1CEST,M3.5.0,M10.5.0/3"` (Europe/Berlin, including
  daylight saving time).
- `DATE_FORMAT = "%a, %e. %B"`: the top-right date reads e.g. "Mo, 2. November".

### `platformio/src/main.cpp`
- With `SENSOR_DHT22`, the sensor is powered from `PIN_BME_PWR`, read after a
  2 s power-up delay via the Adafruit DHT library, and a failed read shows
  `DHT22` plus the read-failed text in the status bar. The BME280/BME680 code
  path is unchanged and only compiled when one of those sensors is selected.
- The 2 s delay adds about 2 s of awake time per update.

## Building and flashing

PlatformIO Core is installed with the official installer, which places it in
`~/.platformio/penv`:

```sh
curl -fsSL -o get-platformio.py https://raw.githubusercontent.com/platformio/platformio-core-installer/master/get-platformio.py
python3 get-platformio.py
ln -sf ~/.platformio/penv/bin/pio ~/.local/bin/pio
ln -sf ~/.platformio/penv/bin/platformio ~/.local/bin/platformio
```

On Arch-based systems the USB serial port (`/dev/ttyUSB0`) belongs to the
`uucp` group:

```sh
sudo usermod -aG uucp $USER   # then log out and back in
```

From `platformio/`:

```sh
pio run -e esp32dev -t upload
pio device monitor -b 115200
```

If the upload stalls at `Connecting....`, hold the board's BOOT button until it
continues. After each update the board deep-sleeps; the EN button wakes it for
an immediate refresh.

## OpenWeatherMap

`OWM_ONECALL_VERSION` is `"3.0"`. One Call 3.0 requires the separate
"One Call by Call" subscription (1,000 free calls/day). Without it, the API
returns `401 Unauthorized` and the display shows the "One Call 3.0 API /
401: Unauthorized" error screen, while the Air Pollution API works with the
same key. Setting "Calls per day (no more than)" to 1,000 at
<https://home.openweathermap.org/subscriptions> keeps usage within the free
tier.

## Battery (not connected yet)

The DevKit has no LiPo charger or battery-voltage divider. Planned wiring:

- LiPo → TP4056 module with protection (B+ / B−). Charging goes through the
  TP4056's USB port.
- TP4056 OUT+ → MCP1700-3302 low-dropout regulator VIN, regulator VOUT → 3V3.
  1 µF ceramic capacitors from VIN and VOUT to GND. TP4056 OUT− is ground for
  everything; B− is not, because the protection circuit sits between them.
- Battery sense: OUT+ → 1 MΩ → D35 → 1 MΩ → GND, with 100 nF from D35 to GND.
  `readBatteryVoltage()` multiplies the ADC reading by 2, which matches two
  equal resistors.
- The battery must not be connected to VIN (the board's AMS1117 regulator needs
  ~1 V of headroom) or directly to 3V3 (a full LiPo is 4.2 V; the ESP32 maximum
  is 3.6 V).
- Disconnect OUT+ from the regulator before plugging the DevKit's own USB-C in
  for flashing.

Once the battery and divider are connected, set `BATTERY_MONITORING` back to
`1`.

The DevKit's always-on red power LED and its USB-serial chip keep drawing
current in deep sleep. Removing the power LED is the largest single
battery-life improvement on this board.
