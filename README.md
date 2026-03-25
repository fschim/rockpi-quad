# ROCK Pi Quad SATA

Top Board control program for ROCK Pi Quad SATA

[Quad SATA HAT wiki](<https://wiki.radxa.com/Dual_Quad_SATA_HAT>)

# How to use

Compile the code into deb package
```
$ cd rockpi-quad
$ chmod 0755 rockpi-quad/DEBIAN/postinst
$ chmod 0755 rockpi-quad/DEBIAN/prerm
$ dpkg-deb --build . rockpi-quad.custom.deb
```
Install the deb package
```
$ sudo dpkg -i rockpi-quad.custom.deb
```

# Developer Guide

## What It Does

This repo is the **control software for the Radxa Quad SATA HAT** — a hardware add-on board that sits on top of a Raspberry Pi 4 or ROCK Pi 4C+ single-board computer. The HAT provides:

- **4 SATA ports** for connecting hard drives/SSDs
- **A cooling fan** (PWM-controlled)
- **A small OLED display** (128×32 SSD1306 via I2C)
- **A physical button** for user interaction

The software runs as a **systemd service** (`rockpi-quad.service`) and handles:
1. Powering on the SATA drives via GPIO
2. Controlling the fan speed based on CPU temperature
3. Driving the OLED to show system stats (uptime, IP, CPU, memory, disk usage)
4. Responding to physical button presses (single-click, double-click, long-press)

It's packaged as a **.deb package** and installed directly onto the target SBC.

## Repository Structure

```
rockpi-quad/                          ← Debian package root (becomes /)
├── DEBIAN/
│   ├── control                       ← Package metadata & dependencies
│   ├── postinst                      ← Post-install script (board detection, I2C/PWM setup)
│   └── prerm                         ← Pre-remove script (stop service, cleanup)
├── etc/
│   └── rockpi-quad.conf              ← User config (fan temps, button mappings, OLED settings)
├── lib/systemd/system/
│   └── rockpi-quad.service           ← systemd unit file
└── usr/bin/rockpi-quad/
    ├── main.py                       ← Entry point — launches all threads
    ├── fan.py                        ← Fan control (PWM or GPIO bit-bang)
    ├── misc.py                       ← Config, button reading, disk info, utilities
    ├── oled.py                       ← OLED display rendering
    ├── test_fan.py                   ← Manual hardware test for fan
    ├── test_misc.py                  ← Manual hardware test for GPIO/button
    ├── test_oled.py                  ← Manual hardware test for OLED display
    ├── requirements.txt              ← Python pip dependencies
    ├── env/                          ← Board-specific pin mappings
    │   ├── rpi4.env                  ← Raspberry Pi 4 GPIO/I2C/PWM config
    │   ├── rock_pi_4.env             ← ROCK Pi 4C+ (Radxa OS)
    │   └── rock_pi_4_armbian.env     ← ROCK Pi 4C+ (Armbian/DietPi)
    ├── fonts/                        ← TTF fonts for the OLED display
    └── overlays/
        └── rk3399-pwm1.dtbo          ← Device-tree overlay for ROCK Pi PWM
```

## How It Works Step by Step

### 1. Installation (`postinst`)
When you `dpkg -i` the deb package:
1. **Installs Python dependencies** from `requirements.txt` via `pip3`.
2. **Enables the systemd service** (`rockpi-quad.service`).
3. **Detects the board model** by reading `/proc/device-tree/model`.
4. Based on the board:
   - **Raspberry Pi 4**: Enables I2C via `raspi-config`, adds PWM device-tree overlay to `/boot/firmware/config.txt`, copies `rpi4.env` → `/etc/rockpi-quad.env`.
   - **ROCK Pi 4C+**: Enables I2C7 overlay (Armbian or Radxa OS), copies the appropriate `.env` file.
5. **Requires a reboot** for hardware overlays to take effect.

### 2. Service Startup (`main.py`)
On boot, systemd runs `python3 /usr/bin/rockpi-quad/main.py` with environment from `/etc/rockpi-quad.env`. The service:
1. **Powers on SATA drives** — sets two GPIO lines HIGH to supply power (`misc.disk_turn_on()`).
2. **Detects OLED presence** — if the `oled` module imports successfully, the "top board" (with OLED+button) is present.
3. Launches **4 daemon threads** (or 1 if no top board):

| Thread | Function | Purpose |
|--------|----------|---------|
| `p0` | `receive_key(q)` | Dequeues button events, executes mapped actions |
| `p1` | `watch_key(q)` | Polls GPIO for button presses, classifies as click/twice/press |
| `p2` | `auto_slider(lock)` | Auto-rotates OLED pages on a timer |
| `p3` | `fan.running()` | Continuously adjusts fan speed based on CPU temp |

4. Main thread **joins on `p3`** (the fan thread) and waits for `SIGINT` (KeyboardInterrupt) to shut down gracefully.

### 3. Fan Control (`fan.py`)
- **Two modes** depending on `HARDWARE_PWM` env var:
  - `HARDWARE_PWM=1` (RPi4): Uses Linux sysfs PWM (`/sys/class/pwm/pwmchipX/pwmY/`). Creates `Pwm` objects that write period and duty cycle values.
  - `HARDWARE_PWM=0` (ROCK Pi): Uses GPIO bit-banging via `libgpiod`. A background thread toggles the pin on/off to simulate PWM.
- **Temperature → duty cycle mapping** (in `misc.py`): Iterates `lv2dc` dict from highest to lowest temperature threshold. Higher temp = lower duty cycle value = faster fan (duty cycle is inverted: 0 = full speed, 0.999 = off).
- Re-reads temperature every **60 seconds** (cached in `get_dc()`), checks for changes every **1 second**.

### 4. OLED Display (`oled.py`)
- Drives a **128×32 SSD1306** OLED over I2C using `adafruit_ssd1306`.
- Three "pages" that rotate automatically (or via button press):
  - **Page 0**: Uptime, CPU temp, IP address
  - **Page 1**: CPU load, memory usage
  - **Page 2**: Disk usage (root, RAID, individual SATA drives)
- Rendering uses **Pillow** (`PIL`) to draw text onto an offscreen image, then pushes to the display.
- Supports **180° rotation** via config.

### 5. Button Handling (`misc.py`)
- Polls a GPIO line at **10 Hz** (100ms intervals).
- Builds a string of `0`s and `1`s representing recent button state.
- Matches against regex patterns to classify:
  - **click**: `1+0+1{wait,}` — brief press then release
  - **twice**: `1+0+1+0+1{3,}` — double-click
  - **press**: `1+0{size,}` — long hold
- Each event type maps to a configurable action: `slider` (next OLED page), `switch` (fan on/off), `reboot`, `poweroff`, or `none`.

### 6. Configuration (`rockpi-quad.conf`)
INI-style config file read by `misc.read_conf()`:
- `[fan]` — Temperature thresholds (°C) for 4 fan speed levels
- `[key]` — Button action mappings (click/twice/press → action)
- `[time]` — Timing parameters for button detection
- `[slider]` — OLED auto-rotation settings
- `[oled]` — Display rotation and Fahrenheit toggle

## Key Dependencies

| Dependency | Purpose |
|-----------|---------|
| `python3-libgpiod` (≥2.0.0) | GPIO control (button, SATA power, GPIO-mode fan) |
| `adafruit-circuitpython-ssd1306` | OLED display driver |
| `Adafruit-Blinka` | CircuitPython compatibility layer for Linux SBCs |
| `python3-pillow` (PIL) | Image/text rendering for the OLED |
| `RPi.GPIO` | Raspberry Pi GPIO (pulled in by Blinka) |
| `raspi-config` | RPi-specific I2C/SPI enablement |

### Side Effects
- **Writes to sysfs** (`/sys/class/pwm/`, `/sys/class/thermal/`) — requires root.
- **Holds GPIO lines open** for the lifetime of the service (SATA power, fan, button).
- **Modifies boot config** on install (`/boot/firmware/config.txt` on RPi, `/boot/armbianEnv.txt` on Armbian).
- **Installs pip packages system-wide** (uses `--break-system-packages` on Python ≥3.11).

## Non-obvious Design Patterns

1. **Debian package IS the repo structure**: The `rockpi-quad/` directory is literally the filesystem layout that `dpkg` will extract to `/`. There's no build step for the Python code — it's installed verbatim to `/usr/bin/rockpi-quad/`.

2. **Environment-file-driven hardware abstraction**: Instead of if/else code, board-specific GPIO pin numbers and I2C bus names live in `.env` files. `postinst` copies the right one to `/etc/rockpi-quad.env`, which the systemd service loads via `EnvironmentFile=`. The Python code reads `os.environ` at runtime.

3. **Mutable default arguments as caches**: `get_dc(cache={})` and `change_dc(dc, cache={})` in `fan.py`, and `get_disk_info(cache={})` in `misc.py` use Python's mutable default argument trick to persist state between calls without using globals or classes.

4. **`multiprocessing.Value` for cross-thread state**: `conf['idx']` and `conf['run']` use `mp.Value('d', ...)` for thread-safe shared state (OLED page index, fan on/off toggle), even though the code uses threads not processes.

5. **Inverted duty cycle for fan**: `0` = full speed, `0.999` = off. The `lv2dc` mapping goes from hottest (lv3→0) to coolest (lv0→0.75). Returning `0.999` (not `1.0`) means the fan is "off" — this is because the PWM polarity is set to `inversed` on RPi4.

6. **Graceful degradation**: If the OLED/top-board import fails, the service still runs — just the fan thread. The `try/except` around `import oled` enables this.

7. **Button detection via regex on GPIO state string**: Rather than interrupts, it polls and accumulates a binary string, then regex-matches patterns. Unusual but effective for debouncing and multi-gesture recognition on resource-constrained hardware.

## Gotchas

1. **Must run as root**: The service writes to sysfs PWM files and controls GPIO. Running without root privileges silently fails or crashes.

2. **Reboot required after install**: The `postinst` enables device-tree overlays that only take effect on next boot. The service will fail if started before rebooting.

3. **No automated tests**: The `test_*.py` files are **manual hardware tests** that require physical access to the board. They load the RPi4 env file and directly interact with hardware. There's no CI test suite.

4. **`postinst` uses `mv` not `cp` for RPi4**: `mv /usr/bin/rockpi-quad/env/rpi4.env /etc/rockpi-quad.env` — if you reinstall the package, this file won't exist anymore (it was moved). The ROCK Pi path uses `cp`. This asymmetry can cause issues on re-install.

5. **Hardcoded board detection**: The `case` statement in `postinst` only matches `Raspberry Pi 4` and `Radxa ROCK 4C+`. Any other board (including Pi 5) gets "Not support in your board" and exits with error code 2.

6. **`pip3 install` runs as root at install time**: This installs Python packages system-wide as root, which can conflict with system packages on newer Debian/Ubuntu. The `--break-system-packages` flag is used for Python ≥3.11, but this is a known pain point.

7. **Thread safety**: The OLED rendering uses a `threading.Lock`, but `conf` dict mutations (like `fan_switch()`) use `mp.Value` for atomicity. The `conf` dict itself isn't thread-locked — concurrent reads of config values are assumed safe (dict reads are atomic in CPython but this isn't guaranteed).

8. **Fan `get_dc()` returns `0.999` (off) when `run.value == 0`**: The fan switch sets `run.value` to `0` which means the fan duty cycle goes to `0.999` (nearly off, inverted polarity). If polarity isn't inversed, this logic reverses.

9. **CI only builds the deb, doesn't test it**: The GitHub Actions workflow (`build-deb.yml`) just runs `dpkg-deb --build` to verify the package builds. No installation or runtime testing.

10. **Config file silently falls back to defaults**: If `/etc/rockpi-quad.conf` has errors, `read_conf()` catches the exception, prints a traceback, and uses hardcoded defaults. The defaults differ from the shipped config file values (e.g., default `lv0=35` vs. config `lv0=45`).
