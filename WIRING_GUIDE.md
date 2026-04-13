## mtKeebs SplitChevron Wiring Guide & Flashing Instructions

### Overview

![Generative AI 3D image of the mtKeebs SplitChevron keyboard](https://imgur.com/a/3d-generative-ai-image-of-splitchevron-keyboard-osvKigf "mtKeebs SplitChevron")

The SplitChevron is my custom-built, hand-wired, unibody ergonomic split keyboard. It has a single rotary encoder and a 1.3" OLED display. I initially considered both QMK and ZMK, but I really wanted a wireless option via bluetooth (ruling out QMK) and I wanted to the display to show things that I was unable to figure out how to do with ZMK (e.g., current weather via OpenWeather's API), so I ended up programming the keyboard using CircuitPython. The keyboard is essentially a 5 x 15 key matrix, although there aren't physical keys at every matrix location. The build uses the Adafruit HUZZAH32 ESP32 Feather, perhaps not the best option, but it has integrated WiFi (integrated 802.11b/g/n) and dual-mode bluetooth (classic and BLE), a 2-pin JST-PH connector for connecting a 3.7v LiPo battery, a click switch button to turn the battery on/off when not not plugged in and not being used, and a momentary reset button to reset the board if the keyboard key combo isn't working for some reason so I don't have to open the case to reset the board.




---

## 1. Bill of Materials

| Qty | Item                                                               |
| --- | ------------------------------------------------------------------ |
| 1   | [Adafruit HUZZAH32 - ESP32 Feather Board][1]                       |
| 67  | MX-compatible mechanical key switches                              |
| 67  | 1N4148 signal diodes (one per switch)                              |
| 1   | EC11 rotary encoder with push button                               |
| 1   | MakerFocus 1.3" SH1106 I²C OLED                                    |
| 1   | MCP23017 I²C GPIO expander (strongly recommended — see note below) |
| —   | 28 AWG stranded wire (matrix)                                      |
| —   | 22 AWG solid wire (power rails)                                    |
| —   | FR4 or acrylic plate for switch mounting                           |
| 2   | 4.7 kΩ resistors (I²C pull-ups, if your OLED module lacks them)    |

> **GPIO note:** The ESP32 Feather exposes \~18 usable digital GPIOs.
> A 5-row + 14-column matrix needs 5 + 14 = 19 pins, plus 3 for the
> encoder and 2 for I²C — a total of 24.  Use an **MCP23017** to
> provide the extra column inputs (C8–C14) over I²C.  Only C1–C7
> and the 5 row pins need direct GPIO connections.

---

## 2. Key Matrix Layout

```
         C0  C1  C2  C3  C4  C5  C6  C7  C8  C9  C10 C11 C12 C13 C14
R0        ■   ■   ■   ■   ■   ■   —   -   —   ■   ■   ■   ■   ■   ■
R1        ■   ■   ■   ■   ■   ■   —   —   —   ■   ■   ■   ■   ■   ■
R2        ■   ■   ■   ■   ■   ■   —   ■   —   ■   ■   ■   ■   ■   ■
R3        ■   ■   ■   ■   ■   ■   ■   ■   ■   ■   ■   ■   ■   ■   ■
R4        —   —   ■   ■   ■   ■   ■   —   ■   ■   ■   ■   ■   —   —

■ = physical switch   

Rotary encoder button is wired directly, not in keyboard matrix

```

### Physical key count per row

| Row       | Columns with keys | Count                                                   |
| --------- | ----------------- | ------------------------------------------------------- |
| R0        | C0–C5, C9–C14     | 12                                                      |
| R1        | C0–C5, C9–C14     | 12                                                      |
| R2        | C0–C5, C7, C9–C14 | 13                                                      |
| R3        | C0–C14            | 15                                                      |
| R4        | C2–C6, C8–C12     | 10                                                      |
| **Total** |                   | **62 switches + 1 encoder button = 63 physical inputs** |

---

## 3. Diode Orientation

COL2ROW orientation

Install **1N4148 diodes** to each switch, **cathode (band) toward the row wire** (anode toward the switch). This prevents ghosting during multi-key presses.

```
 COL wire ────|>|  [switch] ──── |>| diode ──── |>| ROW wire
                            cathode (band) faces ROW
```

---

## 4. GPIO Pin Assignments

### Row pins (direct GPIO, output)

| Row | HUZZAH32 Pin | GPIO \# |
| --- | ------------ | ------- |
| R0  | D13          | 13      |
| R1  | D12          | 12      |
| R2  | D27          | 27      |
| R3  | D33          | 33      |
| R4  | D15          | 15      |

### Column pins C0–C7 (direct GPIO, input with pull-up)

| Column | HUZZAH32 Pin | GPIO \# | Notes                                       |
| ------ | ------------ | ------- | ------------------------------------------- |
| C0     | D32          | 32      |                                             |
| C1     | D26          | 26      |                                             |
| C2     | D25          | 25      |                                             |
| C3     | A5           | 32      |                                             |
| C4     | A4           | 36      | input-only                                  |
| C5     | A3           | 39      | input-only                                  |
| C6     | A2           | 34      | input-only                                  |
| C7     | —            | —       | encoder button only — wired to ENC\_BTN pin |

### Column pins C8–C14 via MCP23017

Wire the MCP23017 to the I²C bus (SDA/SCL).  Set its address pins
A0=A1=A2=GND → I²C address **0x20**.

| Column | MCP23017 Pin | Port          |
| ------ | ------------ | ------------- |
| C8     | GPA0         | Port A, bit 0 |
| C9     | GPA1         | Port A, bit 1 |
| C10    | GPA2         | Port A, bit 2 |
| C11    | GPA3         | Port A, bit 3 |
| C12    | GPA4         | Port A, bit 4 |
| C13    | GPA5         | Port A, bit 5 |
| C14    | GPA6         | Port A, bit 6 |

Connect INTA (interrupt pin) to a spare GPIO if you want interrupt-driven
scanning instead of polling.  For a polling build, leave INTA unconnected.

> **If you skip the MCP23017:** You must re-layout your board to use
> only 14 total columns and remap C8–C14 to free GPIOs, or accept that
> those columns won't function until the expander is added.

### Rotary encoder (direct GPIO)

| Signal   | HUZZAH32 Pin | GPIO \# |
| -------- | ------------ | ------- |
| ENC\_A   | SCK          | 5       |
| ENC\_B   | MOSI         | 18      |
| ENC\_BTN | MISO         | 19      |

Connect encoder GND to board GND.  Add a 100 nF cap across each signal
line to GND to debounce.

### I²C bus (shared: OLED + MCP23017)

| Signal | HUZZAH32 Pin | GPIO \# |
| ------ | ------------ | ------- |
| SDA    | SDA          | 23      |
| SCL    | SCL          | 22      |

Add **4.7 kΩ pull-up resistors** from SDA and SCL to 3.3V if your OLED
module does not include them.

### OLED SH1106 power

| OLED pin | Connect to    |
| -------- | ------------- |
| VCC      | 3.3V          |
| GND      | GND           |
| SDA      | SDA (GPIO 23) |
| SCL      | SCL (GPIO 22) |

Default I²C address: **0x3C**.  Some modules ship as 0x3D — check with
a short I²C scanner script if the display doesn't initialise.

---

## 5. Wiring Checklist

- [ ] All row wires connected to correct GPIO output pins
- [ ] All column wires connected (C1–C6 direct, C8–C14 via MCP23017)
- [ ] Diodes installed cathode-toward-column on all 58 switches
- [ ] Encoder A/B/GND/BTN connected
- [ ] OLED VCC, GND, SDA, SCL connected
- [ ] MCP23017 VCC (3.3V), GND, SDA, SCL, A0/A1/A2 to GND connected
- [ ] I²C pull-up resistors installed (if not on module PCBs)
- [ ] No solder bridges between adjacent pads

---

## 6. Required CircuitPython Libraries

Download the **CircuitPython 9.x library bundle** from
https://circuitpython.org/libraries and copy these folders/files into
the `lib/` directory on your CIRCUITPY drive:

```
lib/
├── adafruit_hid/               (keyboard, consumer control)
├── adafruit_displayio_sh1106.mpy
├── adafruit_display_text/      (label)
├── adafruit_requests.mpy
├── adafruit_bus_device/        (for MCP23017)
├── adafruit_mcp230xx/          (MCP23017 driver)
└── rotaryio.mpy                (built into CP9, no copy needed)
```

`rotaryio` and `busio` / `digitalio` are built into CircuitPython and
**do not need to be copied**.

---

## 7. File Layout on CIRCUITPY Drive

```
CIRCUITPY/
├── boot.py          ← enable USB HID (runs before code.py)
├── code.py          ← main firmware
├── keymap.py        ← layer keymaps
├── display.py       ← OLED driver
├── encoder.py       ← rotary encoder handler
├── wpm.py           ← WPM counter
├── weather.py       ← OpenWeatherMap fetch
├── settings.toml    ← WiFi credentials + OWM API key
└── lib/
    └── (libraries listed above)
```

---

## 8. Flashing CircuitPython to the HUZZAH32

### Step 1 — Download CircuitPython UF2

Go to https://circuitpython.org/board/adafruit\_feather\_esp32\_v2 (or the
correct HUZZAH32 variant you have) and download the latest **stable**
`.bin` file.

### Step 2 — Install `esptool`

```bash
pip install esptool
```

### Step 3 — Enter download mode

Hold the **BOOT** button on the HUZZAH32, then press and release **RESET**
while still holding BOOT.  Release BOOT.  The board enters ROM bootloader
mode and appears as a serial device.

### Step 4 — Erase flash

```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 erase_flash
# On macOS: --port /dev/cu.usbserial-XXXX
# On Windows: --port COM3
```

### Step 5 — Flash CircuitPython

```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 \
  write_flash -z 0x0000 adafruit-circuitpython-feather_esp32_v2-en_US-9.x.x.bin
```

### Step 6 — Verify

Press RESET.  A `CIRCUITPY` USB drive should appear within a few seconds.

---

## 9. Deploying the Firmware

1. Open the `CIRCUITPY` drive in your file manager.
2. Copy **all project files** to the drive root and `lib/` as shown in §7.
3. Edit `settings.toml` with your WiFi credentials and OWM API key.
4. **Safely eject** the drive.
5. Press RESET — `boot.py` runs first, then `code.py`.

> **Safe mode:** Hold the BOOT button during power-on to skip `code.py`
> and regain filesystem access for editing.

---

## 10. Troubleshooting

| Symptom                        | Likely cause               | Fix                                    |
| ------------------------------ | -------------------------- | -------------------------------------- |
| CIRCUITPY doesn't appear       | Flash not successful       | Re-flash; check baud rate              |
| Keys not registering           | Diodes backward            | Verify cathode→column orientation      |
| Ghosting on multi-key press    | Missing diodes             | Add missing diodes                     |
| OLED blank                     | Wrong I²C address          | Try 0x3D in `display.py`               |
| Encoder not responding         | Wrong pins                 | Double-check ENC\_A/B pin mapping      |
| Weather not updating           | Invalid API key or no WiFi | Check settings.toml; verify SSID       |
| `ImportError` on boot          | Missing library            | Copy missing lib from bundle           |
| Keyboard not recognised as HID | boot.py not run            | Power-cycle board after saving boot.py |

---

## 11. Customising Keymaps

Open `keymap.py` and edit the `BASE`, `SHIFTR`, `SHIFTL`, or `FNKEYS`
dictionaries.  Each row is built with `_row({col_index: Keycode, ...})`.
All valid keycodes are in `adafruit_hid.keycode.Keycode`.

To change which physical key activates a layer, update `RSHIFT_POS`,
`LSHIFT_POS`, or `FN_POS` in `code.py`.

---

## 12. OpenWeatherMap Setup

1. Create a free account at https://openweathermap.org
2. Go to **API Keys** in your profile and generate a key.
3. Paste the key into `settings.toml` as `OWM_API_KEY`.
4. Update `OWM_LAT` / `OWM_LON` for your location.

The free tier supports 60 calls/minute and 1,000,000 calls/month —
more than sufficient for polling every 10 minutes.

[1]:	https://www.adafruit.com/product/3405
■ = physical switch   

Rotary encoder button is wired directly, not in keyboard matrix

```

### Physical key count per row
| Row | Columns with keys | Count |
|-----|-------------------|-------|
| R0  | C0–C5, C9–C14     |  12   |
| R1  | C0–C5, C9–C14     |  12   |
| R2  | C0–C5, C7, C9–C14 |  13   |
| R3  | C0–C14            |  15   |
| R4  | C2–C6, C8–C12     | 10    |
| **Total** | | **62 switches + 1 encoder button = 63 physical inputs** |

---

## 3. Diode Orientation

COL2ROW orientation

Install **1N4148 diodes** to each switch, **cathode (band) toward the row wire** (anode toward the switch). This prevents ghosting during multi-key presses.

```
 COL wire ───> [switch] ───> diode ───> ROW wire
                            cathode (band) faces ROW
```

---

## 4. GPIO Pin Assignments

### Row pins (direct GPIO, output)

| Row | HUZZAH32 Pin | GPIO # |
|-----|-------------|--------|
| R0  |    D13      | 13 |
| R1  |    D12      | 12 |
| R2  |    D27      | 27 |
| R3  |    D33      | 33 |
| R4  |    D15      | 15 |

### Column pins C0–C7 (direct GPIO, input with pull-up)

| Column | HUZZAH32 Pin | GPIO # | Notes |
|--------|-------------|--------|-------|
| C0 | D32 | 32 | |
| C1 | D26 | 26 | |
| C2 | D25 | 25 | |
| C3 | A5 | 32 | |
| C4 | A4 | 36 | input-only |
| C5 | A3 | 39 | input-only |
| C6 | A2 | 34 | input-only |
| C7 | — | — | encoder button only — wired to ENC_BTN pin |

### Column pins C8–C14 via MCP23017

Wire the MCP23017 to the I²C bus (SDA/SCL).  Set its address pins
A0=A1=A2=GND → I²C address **0x20**.

| Column | MCP23017 Pin | Port |
|--------|-------------|------|
| C8 | GPA0 | Port A, bit 0 |
| C9 | GPA1 | Port A, bit 1 |
| C10 | GPA2 | Port A, bit 2 |
| C11 | GPA3 | Port A, bit 3 |
| C12 | GPA4 | Port A, bit 4 |
| C13 | GPA5 | Port A, bit 5 |
| C14 | GPA6 | Port A, bit 6 |

Connect INTA (interrupt pin) to a spare GPIO if you want interrupt-driven
scanning instead of polling.  For a polling build, leave INTA unconnected.

> **If you skip the MCP23017:** You must re-layout your board to use
> only 14 total columns and remap C8–C14 to free GPIOs, or accept that
> those columns won't function until the expander is added.

### Rotary encoder (direct GPIO)

| Signal | HUZZAH32 Pin | GPIO # |
|--------|-------------|--------|
| ENC_A | SCK | 5 |
| ENC_B | MOSI | 18 |
| ENC_BTN | MISO | 19 |

Connect encoder GND to board GND.  Add a 100 nF cap across each signal
line to GND to debounce.

### I²C bus (shared: OLED + MCP23017)

| Signal | HUZZAH32 Pin | GPIO # |
|--------|-------------|--------|
| SDA | SDA | 23 |
| SCL | SCL | 22 |

Add **4.7 kΩ pull-up resistors** from SDA and SCL to 3.3V if your OLED
module does not include them.

### OLED SH1106 power

| OLED pin | Connect to |
|----------|-----------|
| VCC | 3.3V |
| GND | GND |
| SDA | SDA (GPIO 23) |
| SCL | SCL (GPIO 22) |

Default I²C address: **0x3C**.  Some modules ship as 0x3D — check with
a short I²C scanner script if the display doesn't initialise.

---

## 5. Wiring Checklist

- [ ] All row wires connected to correct GPIO output pins
- [ ] All column wires connected (C1–C6 direct, C8–C14 via MCP23017)
- [ ] Diodes installed cathode-toward-column on all 58 switches
- [ ] Encoder A/B/GND/BTN connected
- [ ] OLED VCC, GND, SDA, SCL connected
- [ ] MCP23017 VCC (3.3V), GND, SDA, SCL, A0/A1/A2 to GND connected
- [ ] I²C pull-up resistors installed (if not on module PCBs)
- [ ] No solder bridges between adjacent pads

---

## 6. Required CircuitPython Libraries

Download the **CircuitPython 9.x library bundle** from
https://circuitpython.org/libraries and copy these folders/files into
the `lib/` directory on your CIRCUITPY drive:

```
lib/
├── adafruit_hid/               (keyboard, consumer control)
├── adafruit_displayio_sh1106.mpy
├── adafruit_display_text/      (label)
├── adafruit_requests.mpy
├── adafruit_bus_device/        (for MCP23017)
├── adafruit_mcp230xx/          (MCP23017 driver)
└── rotaryio.mpy                (built into CP9, no copy needed)
```

`rotaryio` and `busio` / `digitalio` are built into CircuitPython and
**do not need to be copied**.

---

## 7. File Layout on CIRCUITPY Drive

```
CIRCUITPY/
├── boot.py          ← enable USB HID (runs before code.py)
├── code.py          ← main firmware
├── keymap.py        ← layer keymaps
├── display.py       ← OLED driver
├── encoder.py       ← rotary encoder handler
├── wpm.py           ← WPM counter
├── weather.py       ← OpenWeatherMap fetch
├── settings.toml    ← WiFi credentials + OWM API key
└── lib/
    └── (libraries listed above)
```

---

## 8. Flashing CircuitPython to the HUZZAH32

### Step 1 — Download CircuitPython UF2

Go to https://circuitpython.org/board/adafruit_feather_esp32_v2 (or the
correct HUZZAH32 variant you have) and download the latest **stable**
`.bin` file.

### Step 2 — Install `esptool`

```bash
pip install esptool
```

### Step 3 — Enter download mode

Hold the **BOOT** button on the HUZZAH32, then press and release **RESET**
while still holding BOOT.  Release BOOT.  The board enters ROM bootloader
mode and appears as a serial device.

### Step 4 — Erase flash

```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 erase_flash
# On macOS: --port /dev/cu.usbserial-XXXX
# On Windows: --port COM3
```

### Step 5 — Flash CircuitPython

```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 \
  write_flash -z 0x0000 adafruit-circuitpython-feather_esp32_v2-en_US-9.x.x.bin
```

### Step 6 — Verify

Press RESET.  A `CIRCUITPY` USB drive should appear within a few seconds.

---

## 9. Deploying the Firmware

1. Open the `CIRCUITPY` drive in your file manager.
2. Copy **all project files** to the drive root and `lib/` as shown in §7.
3. Edit `settings.toml` with your WiFi credentials and OWM API key.
4. **Safely eject** the drive.
5. Press RESET — `boot.py` runs first, then `code.py`.

> **Safe mode:** Hold the BOOT button during power-on to skip `code.py`
> and regain filesystem access for editing.

---

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| CIRCUITPY doesn't appear | Flash not successful | Re-flash; check baud rate |
| Keys not registering | Diodes backward | Verify cathode→column orientation |
| Ghosting on multi-key press | Missing diodes | Add missing diodes |
| OLED blank | Wrong I²C address | Try 0x3D in `display.py` |
| Encoder not responding | Wrong pins | Double-check ENC_A/B pin mapping |
| Weather not updating | Invalid API key or no WiFi | Check settings.toml; verify SSID |
| `ImportError` on boot | Missing library | Copy missing lib from bundle |
| Keyboard not recognised as HID | boot.py not run | Power-cycle board after saving boot.py |

---

## 11. Customising Keymaps

Open `keymap.py` and edit the `BASE`, `SHIFTR`, `SHIFTL`, or `FNKEYS`
dictionaries.  Each row is built with `_row({col_index: Keycode, ...})`.
All valid keycodes are in `adafruit_hid.keycode.Keycode`.

To change which physical key activates a layer, update `RSHIFT_POS`,
`LSHIFT_POS`, or `FN_POS` in `code.py`.

---

## 12. OpenWeatherMap Setup

1. Create a free account at https://openweathermap.org
2. Go to **API Keys** in your profile and generate a key.
3. Paste the key into `settings.toml` as `OWM_API_KEY`.
4. Update `OWM_LAT` / `OWM_LON` for your location.

The free tier supports 60 calls/minute and 1,000,000 calls/month —
more than sufficient for polling every 10 minutes.
