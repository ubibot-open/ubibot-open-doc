# UbiBot Open — System Deployment, Flashing & Bring-up Guide

This document is for anyone who wants to wire together the three repositories —
`ubibot-open-server`, `ubibot-open-ws1b`, and `ubibot-serial-sync` — and go from zero to a
complete "device online → reporting data → visible on the dashboard" pipeline. For the
device↔server protocol details, see the
[Hardware Communication Protocol](../protocol/hardware-communication-protocol.md); this document
only covers getting each component running and wiring them together.

> Repository naming note: the repo names used in the commands below reflect this org's
> (`ubibot-open`) actual current GitHub addresses (the core platform was historically named
> `ubibot-platform-open`, and the firmware `ubibot-ws1b`). If you've already renamed them to
> `ubibot-open-server` / `ubibot-open-ws1b` on your end, just clone from the new repo addresses
> instead — it doesn't affect the rest of the steps.

## 0. System Overview

```
┌────────────────────┐   USB serial                                  ┌────────────────────┐
│ ubibot-serial-sync  │ ───────────────────────────────────────────▶ │  ubibot-open-ws1b   │
│ Desktop serial      │  Provisioning: SetupWifi / SetupServer       │  firmware device     │
│ debug/provisioning  │  (protocol §1.2)                             │  (logs / debugging)  │
│                     │◀─────────────────────────────────────────── │                      │
└────────────────────┘                                               └──────────┬──────────┘
                                                                                 │ HTTP (protocol §2–§8)
                                                                                 │ POST /api/v1/auth/time
                                                                                 │ POST /api/v1/data/report
                                                                                 ▼
                                                                     ┌────────────────────────────┐
                                                                     │     ubibot-open-server      │
                                                                     │ Go backend + embedded React │
                                                                     │ admin console; single binary,│
                                                                     │ SQLite storage               │
                                                                     └─────────────┬──────────────┘
                                                                                   ▲
                                                                                   │ Browser http://<host>:8080
                                                                                   │
                                                                          ┌────────┴────────┐
                                                                          │  Admin's browser │
                                                                          └─────────────────┘
```

The three components are deployed independently and only interact over the network (device →
backend, HTTP) and a physical cable (PC → device, USB serial — used for §1.2 provisioning
commands and log debugging); no shared code or shared config files. The protocol has no plans for
Bluetooth provisioning (§1.1 is explicitly marked unsupported) — on-site device provisioning goes
exclusively through serial.

## 1. Prerequisites

| Component | Purpose | Requirements |
|---|---|---|
| `ubibot-open-server` | Device-facing backend + admin console, built into a single binary | Go 1.23+, Node.js/npm (needed to build the admin frontend) |
| `ubibot-open-ws1b` | WS1B reference firmware (target chip ESP32-C5) | ESP-IDF v6.0.2 |
| `ubibot-serial-sync` | Desktop serial debugging tool | Download the official prebuilt package — **no build required**; to build from source see that repo's `BUILD.md` (Qt 6 / CMake / C++17) |

## 2. Deploy and Start the Backend (ubibot-open-server)

### 2.1 Get the source

```bash
git clone https://github.com/ubibot-open/ubibot-open-server.git ubibot-open-server
cd ubibot-open-server
```

### 2.2 One-command build

```bash
./build.sh        # Linux / macOS
# .\build.ps1      # Windows
```

The build script does two things: builds the React admin console under `admin/`, embedding the
output into `server/internal/webui/dist`; then builds the Go backend, producing a single binary
`ubibot-server` at the repo root (`CGO_ENABLED=0`, a pure-Go SQLite driver, statically linked — no
extra runtime or database service to install).

### 2.3 Run it

```bash
./ubibot-server
```

Listens on `:8080` by default; the database file defaults to `./data/ubibot.db` (the directory
and file are created automatically on first run). Override with these environment variables:

| Env var | Default | Description |
|---|---|---|
| `UBIBOT_LISTEN_ADDR` | `:8080` | Listen address |
| `UBIBOT_DB_PATH` | `./data/ubibot.db` | SQLite database file path |
| `UBIBOT_ADMIN_PASSWORD` | (randomly generated if unset) | Only takes effect when there isn't a single admin account in the database yet (i.e. first run); the username is always `admin` |

On first run, if `UBIBOT_ADMIN_PASSWORD` wasn't set ahead of time, the log prints a line like:

```
no admin account found — created "admin" with a generated password: 3f9a1b2c4d5e6f
```

**This password is only printed once** — write it down immediately. Alternatively, set
`UBIBOT_ADMIN_PASSWORD=<your password>` before the very first run to skip the random password.

The first run also seeds a demo device (`pid=ubibot_open_dev_v1`, `sn=sn_ws1_20001_1`), which you
can see right away in the admin console's device list — a way to confirm the backend itself is
working without waiting on real hardware to come online.

### 2.4 Log in to the admin console

Open `http://localhost:8080` in a browser (the API and admin console are served from the same
process and port), and log in with the `admin` account/password from the previous step.

### 2.5 Sending a command to a device (optional)

Once a device has reported at least once and shows up in the device list, open its detail page —
the **Device Commands** card lets you queue a **reboot** or a new **report interval** (60–86400
seconds). This is the protocol §9 command channel: whatever you queue is delivered piggybacked on
that device's *next* report, then cleared — there's no ack, so the platform can't confirm the
device actually received or applied it. If in doubt, just queue it again.

### 2.6 Products and batch device management (optional)

For more than a handful of devices:

- **Products** (Device Management → Products) — register a `pid` once with a display name and
  description, and every device reporting that `pid` shows a resolved product name in the device
  list/detail instead of a raw `pid`. Purely display metadata: creating, renaming, or deleting a
  product never touches any device row.
- **Batch import** (Devices → Batch Import) — paste or upload a CSV of `sn,pid,name` rows (name
  optional) to pre-register a production batch ahead of time, before any of them have ever
  reported. Also purely a convenience: a device that's never imported still appears on its own the
  moment it first reports.
- **Export** (Devices → Export CSV) — downloads the entire fleet (not just the current page) as
  CSV.

## 3. Flash the WS1B Reference Firmware (ubibot-open-ws1b)

### 3.1 Get the source and install ESP-IDF

```bash
git clone https://github.com/ubibot-open/ubibot-ws1b.git ubibot-open-ws1b
cd ubibot-open-ws1b
```

Requires **ESP-IDF v6.0.2** (target chip **ESP32-C5**). Install it per Espressif's official
documentation, then activate its environment (`. ./export.sh`, or `export.ps1` on Windows / the
"Command Prompt"/"PowerShell" shortcuts ESP-IDF installs) at the start of every new terminal
session so the `idf.py` command is available.

### 3.2 Configure device parameters

```bash
idf.py set-target esp32c5   # needed the first time you build; skip if build/ already exists
idf.py menuconfig
```

Go to the `UbiBot WS1B Configuration` menu and set as needed:

| Submenu | Setting | Default | Description |
|---|---|---|---|
| WiFi Configuration | WiFi SSID | `TEST24` | The WiFi network name the device connects to |
| | WiFi Password | (empty) | Leave empty to connect to an open (no-password) network |
| | WiFi country code | `01` | Two-letter regulatory country code (e.g. `CN`/`US`), or `01` for the world-safe default |
| Server Configuration | Data server host | `192.168.2.71` | IP/domain of the backend deployed in step 2 |
| | Data server port | `8080` | Must match the backend's `UBIBOT_LISTEN_ADDR` port |
| Device Identity | Device product ID | `ubibot-ws1b` | Can be shared across devices of the same model |
| | Device serial number | `RV41554WS1B` | **Must be changed to a unique value per physical device** — don't reuse one SN across a production batch |

Settings are saved to the local `sdkconfig` file (gitignored, never committed); defaults are
defined in `main/Kconfig.projbuild`.

> **These are only factory defaults**: the menuconfig settings in this section are baked into
> `sdkconfig` at build time, and only serve as the default WiFi/server address the first time the
> device is flashed. After flashing, they can also be overridden at **runtime** over serial using
> the [Hardware Communication Protocol](../protocol/hardware-communication-protocol.md) §1.2
> `SetupWifi`/`SetupServer` commands, without recompiling or reflashing — see section 4. Bluetooth
> provisioning (protocol §1.1) remains explicitly unsupported; serial provisioning is currently
> the only runtime provisioning method.

### 3.3 Build and flash

```bash
idf.py build
idf.py -p <port> flash monitor
```

`<port>` is e.g. `COM5` on Windows, `/dev/ttyUSB0` on Linux, or `/dev/cu.usbserial-xxxx` on
macOS. `flash` automatically drops into `monitor` (serial log view) once done, so you can watch
the device boot, connect to WiFi, and report data in real time. Press `Ctrl+]` to exit monitor.

## 4. View Logs, Debug, and Provision the Device Over Serial (ubibot-serial-sync)

Purpose: view the device's serial log and send/receive data manually, without an ESP-IDF install
(especially useful for a device that's already been flashed in production and isn't convenient to
hook up to an ESP-IDF environment) — a cross-platform GUI that's nicer to use than
`idf.py monitor`. The protocol §1.2 serial provisioning commands (`SetupWifi`/`SetupServer`) are
also sent through this tool's manual-send box.

### 4.1 Get it

Prefer downloading the prebuilt package directly — no Qt or build tools needed:

| Platform | Download |
|---|---|
| Windows | [UbiBotSerialAssistant-windows-x64.zip](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest/download/UbiBotSerialAssistant-windows-x64.zip) |
| macOS | [UbiBotSerialAssistant-macos.dmg](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest/download/UbiBotSerialAssistant-macos.dmg) |
| Linux | [UbiBotSerialAssistant-linux-x86_64.AppImage](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest/download/UbiBotSerialAssistant-linux-x86_64.AppImage) (`chmod +x`, then run directly) |

The first run will be flagged by the OS as "unknown publisher" (Windows SmartScreen / macOS
Gatekeeper) — click through it (or right-click → Open on macOS), same as any other unsigned
open-source software; this is expected.

### 4.2 Connect to the device

1. Open the app, and in the "Serial Connection" panel select the port for your device, with the
   baud rate matching the firmware's UART log output (115200 by default on ESP-IDF).
2. Open the port — the data monitor panel on the right shows a live, color-coded TX/RX/SYS/ERR
   log, similar to `idf.py monitor` but without needing ESP-IDF installed.
3. To send something manually, use the manual-send box (ASCII/HEX supported), or pick a preset
   from the "Device Command Library" panel by model.

> The device command library doesn't yet include a preset command set for WS1B (the library is
> purely JSON-config-driven, so adding a model is just a JSON file, no code change) — use the
> manual-send box for provisioning commands too, for now.

### 4.3 Provision Over Serial (no reflash needed)

The device only opens its provisioning window at **power-on** (closing early after a few seconds
of silence following the last byte received, with a hard cap regardless of activity — see
[main/provisioning.c](https://github.com/ubibot-open/ubibot-ws1b/blob/main/main/provisioning.c)
for the exact durations); a periodic timer wake-up does not reopen it. So connect over serial and
send this **right after power-cycling or resetting the device**:

```jsonc
// Change WiFi
{"command":"SetupWifi","ssid":"MyHomeWiFi","password":"12345678","type":"WPA2"}
// Change the server address
{"command":"SetupServer","host":"192.168.2.71","port":8080}
```

After processing each command, the device writes back one line of JSON ack in the data monitor
panel: success looks like `{"c":0,"msg":"wifi saved"}`, a validation error is
`{"c":1,"msg":"<reason>"}`, and a save failure is `{"c":2,"msg":"<reason>"}`. Once saved, it
**takes effect immediately** — no reboot needed — and is written to storage that survives power
loss, so every subsequent boot uses the latest config until it's overwritten again. Sending a
command after the window has closed gets no response at all (no ack, no effect); just power-cycle
or reset the device and try again.

## 5. End-to-End Verification: Device Online and Reporting

Check the following in order; if any step doesn't check out, troubleshoot using the corresponding
section:

1. **Device side**: `idf.py monitor` (or serial-sync's log panel) shows the device printing
   something like "WiFi connected" and getting an IP address.
   → Not seeing it: check whether the WiFi SSID/password/country code from step 3.2 are correct,
   or whether the router/AP is reachable.
2. **Network & protocol**: the log shows a call to `POST /api/v1/data/report` with a
   `{"c":0,...}` response.
   → Stuck with no response or a timeout: confirm the backend from step 2 is running, that
   `Data server host/port` actually points at where the backend is listening, and that the host
   machine's firewall allows that port through.
3. **Visible on the dashboard**: back in the admin console (`http://<backend address>:8080`)
   device list, the SN you configured shows up automatically — the protocol has a device
   auto-register the moment it first reports successfully, with no need to pre-create it in the
   console. Its latest reading is visible on the "Data Warehouse" page too.

If all three check out, the "device online → reporting → visible on the dashboard" pipeline is
working end to end.

## 6. Testing Without Real Hardware (optional)

The `ubibot-open-server` repo ships a pure-C device simulator (`simulation/` directory) with
protocol behavior identical to the real firmware — useful when you don't have hardware on hand,
or just want to quickly verify a backend/dashboard change:

```bash
cd ubibot-open-server/simulation
cmake -S . -B build
cmake --build build
./build/ub_device_sim --host 127.0.0.1 --port 8080
```

The defaults match the demo device seeded on the backend's first run (`sn_ws1_20001_1`); use
`--pid`/`--sn` to use a different device identity instead — the protocol doesn't require
pre-registering a device, so the server auto-registers it on first report. `--tick-ms` adjusts
the main loop's heartbeat interval (1000ms by default); `Ctrl+C` stops the simulator.

## 7. Troubleshooting

| Symptom | Error code | Cause & fix |
|---|---|---|
| Report rejected, `c=1002` | Timestamp outside the ±5-minute window | The device's local clock has drifted; confirm it calls `POST /api/v1/auth/time` to recalibrate after receiving this error |
| Report rejected, `c=1003` | Malformed request body | Check the firmware's JSON serialization against protocol §5 |
| Report rejected, `c=1103` | Device disabled by an admin | Re-enable it from the device's detail page in the admin console |
| Report rejected, `c=1900` | Rate limited | Reporting too frequently; wait for the next cycle, or check whether the device is retrying abnormally |
| Device logs can never reach the backend | No response / timeout | Confirm `Data server host/port` is correct, the backend process is running, the host machine's firewall allows the `UBIBOT_LISTEN_ADDR` port through, and the device and backend can reach each other on the network |
| Forgot the admin console password | — | Stop the `ubibot-server` process, open the database with `sqlite3` and run `DELETE FROM admin_users;` to clear the admin table, then start the service again — it re-runs the "first run" logic from §2.3 and creates a fresh `admin` account (device data is unaffected) |
| Sent `SetupWifi`/`SetupServer` over serial, nothing happened | — | The provisioning window only opens at **power-on** and only for a limited time (see §4.3); sending after it closes gets no ack — power-cycle or reset the device and send again promptly |

---

References:
[Hardware Communication Protocol](../protocol/hardware-communication-protocol.md) ·
[ubibot-open-server](https://github.com/ubibot-open/ubibot-open-server) ·
[ubibot-open-ws1b](https://github.com/ubibot-open/ubibot-ws1b) ·
[ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync)
