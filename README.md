# UbiBot Open Doc

Configuration and technical documentation repository for the UbiBot Open project (`ubibot-open` org): documentation that spans multiple repos and doesn't belong in any single code repo — the device communication protocol, the system deployment/flashing/bring-up guide, and so on.

## Contents

- [Hardware Communication Protocol](protocol/hardware-communication-protocol.md) — the authoritative definition of the device↔server HTTP protocol: provisioning, time sync, data upload, error codes, and more.
- [System Deployment, Flashing & Bring-up Guide](guides/deployment-flashing-guide.md) — a hands-on manual for going from zero to "backend deployed → firmware flashed → serial debugging → device online and reporting → visible on the dashboard".

## Related Repositories

| Repository | Description |
|---|---|
| [ubibot-open-server](https://github.com/ubibot-open/ubibot-open-server) | Device-facing backend + admin console (Go + React), deployed as a single binary |
| [ubibot-open-ws1b](https://github.com/ubibot-open/ubibot-ws1b) | Open-source reference firmware for the WS1B device (ESP-IDF, ESP32-C5) |
| [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync) | Cross-platform desktop serial debugging tool |


## Quick Start

Follow these 4 steps to go from zero to "server running → hardware flashed → reporting over the
network → data visible on the dashboard". See the
[Deployment, Flashing & Bring-up Guide](guides/deployment-flashing-guide.md) for details, optional
parameters, and troubleshooting for each step. No hardware on hand? Skip ahead to "No hardware?"
right before step 4 — the built-in simulator lets you complete steps 1 and 4 on their own.

### 1. Deploy the server

```bash
git clone https://github.com/ubibot-open/ubibot-open-server.git ubibot-open-server
cd ubibot-open-server
./build.sh        # Windows: .\build.ps1; requires Go 1.23+ and Node.js/npm
./ubibot-server    # listens on :8080 by default
```

Open `http://localhost:8080` in a browser. On first run, the log prints a one-time generated
`admin` password (e.g. `no admin account found — created "admin" with a generated password: ...`)
— write it down right away, it's only shown once. See guide §2 for details.

### 2. Build and flash the hardware

```bash
git clone https://github.com/ubibot-open/ubibot-ws1b.git ubibot-open-ws1b
cd ubibot-open-ws1b
idf.py set-target esp32c5
idf.py menuconfig      # see "Configure the hardware" below; save and exit, then continue
idf.py build
idf.py -p <port> flash monitor
```

Requires **ESP-IDF v6.0.2** (target chip **ESP32-C5**) with its environment activated
(`export.sh`/`export.ps1`). `<port>` is e.g. `COM5` on Windows or `/dev/ttyUSB0` on Linux.

### 3. Configure the hardware

In `idf.py menuconfig` from the previous step, go to the `UbiBot WS1B Configuration` menu and set
at least these 3 items before saving:

| Setting | What to set it to |
|---|---|
| WiFi SSID / Password | The WiFi you're actually connecting to on site |
| Data server host / port | The server address from step 1, default port `8080` |
| Device serial number (SN) | Unique per device — never reuse across devices |

See guide §3.2 for the full list of settings (country code, product ID, etc.). Once done, go back
to step 2 and continue with `build`/`flash`.

### 4. View the data

Once the device is online and reporting, go back to the admin console opened in step 1 — the SN
you just configured appears automatically in the device list, no need to create the device
beforehand. Open it to see the latest reading, or check the "Data Warehouse" page for the latest
record from every device.

Not seeing the device? Walk through the guide's
[end-to-end verification](guides/deployment-flashing-guide.md#5-end-to-end-verification-device-online-and-reporting)
— three checks (WiFi connected → report succeeded → visible on the dashboard).

> **No hardware?** `ubibot-open-server/simulation` ships a pure-C device simulator with protocol
> behavior identical to the real firmware, letting you verify the server and dashboard without
> steps 2–3. See guide §6.
