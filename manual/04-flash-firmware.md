# Flash the WS1B Reference Firmware

*[中文](04-flash-firmware.zh-CN.md)*

Skip this chapter (and chapter 5) if you don't have a real WS1B device — go straight to
[No Hardware? Use the Simulator](07-no-hardware-path.md) instead.

## 1. Get the source

```bash
git clone https://github.com/ubibot-open/ubibot-ws1b.git ubibot-open-ws1b
cd ubibot-open-ws1b
```

If you haven't installed ESP-IDF v6.0.2 yet, see [Prerequisites](01-prerequisites.md) — install it
per Espressif's official documentation, then activate its environment (`. ./export.sh`, or
`export.ps1` on Windows) at the start of every new terminal session so the `idf.py` command is
available.

## 2. Set the target chip

```bash
idf.py set-target esp32c5
```

Only needed the first time you build (skip it if `build/` already exists from a previous build).

## 3. Configure device parameters

```bash
idf.py menuconfig
```

**Screenshot placeholder:** `04-flash-firmware-01.png` — the `idf.py menuconfig` terminal UI, with
the `UbiBot WS1B Configuration` menu open.

Navigate to the `UbiBot WS1B Configuration` menu and set:

| Submenu | Setting | Default | What to put here |
|---|---|---|---|
| WiFi Configuration | WiFi SSID | `TEST24` | The WiFi network name the device should connect to |
| | WiFi Password | (empty) | Leave empty for an open network |
| | WiFi country code | `01` | Your two-letter regulatory country code (e.g. `US`/`CN`), or `01` for the world-safe default |
| Server Configuration | Data server host | `192.168.2.71` | The IP/domain of the backend from [chapter 2](02-deploy-server.md) — the WS1B's LAN address, not `localhost` |
| | Data server port | `8080` | Must match `UBIBOT_LISTEN_ADDR`'s port if you changed it |
| Device Identity | Device product ID | `ubibot-ws1b` | Shared by every device of this model — leave as-is |
| | Device serial number | `RV41554WS1B` | Doesn't need to be unique per build — see the note below |

> **Note:** these menuconfig values only set the *factory default* — baked into the firmware at
> build time, used the first time the device boots. They can all be changed afterward, at runtime,
> over serial, with no rebuild/reflash — that's [chapter 5](05-provision-device-over-serial.md).
> In particular, since the serial number can be set later per unit, you can build and flash an
> entire batch of devices from this one build, then give each one its own real serial number
> afterward over serial.

Settings are saved to a local `sdkconfig` file (not committed to git).

## 4. Build and flash

```bash
idf.py build
idf.py -p <port> flash monitor
```

`<port>` is the device's serial port — e.g. `COM5` on Windows, `/dev/ttyUSB0` on Linux, or
`/dev/cu.usbserial-xxxx` on macOS. `flash` writes the firmware, then automatically drops into
`monitor` (a live serial log view), so you can watch the device boot, connect to WiFi, and start
reporting in real time.

**Screenshot placeholder:** `04-flash-firmware-02.png` — the `idf.py flash` terminal output
finishing successfully and dropping into `monitor`.

**Screenshot placeholder:** `04-flash-firmware-03.png` — the boot log in `monitor`, showing the
device connecting to WiFi and posting its first report.

Press `Ctrl+]` to exit `monitor` when you're done watching.

## Next

[Provision the Device Over Serial](05-provision-device-over-serial.md).
