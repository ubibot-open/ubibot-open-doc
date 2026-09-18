# No Hardware? Use the Simulator

*[中文](07-no-hardware-path.zh-CN.md)*

This chapter replaces chapters 4-6 entirely. [ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator)
is a small C program that speaks the exact same HTTP protocol a real WS1B device would — no
WiFi, no serial port, no physical hardware — so you reach the same end state (a device online and
reporting) with just your laptop.

## 1. Build it

```bash
git clone https://github.com/ubibot-open/ubibot-open-simulator.git
cd ubibot-open-simulator
cmake -S . -B build
cmake --build build
```

On Windows with MinGW: `cmake -S . -B build -G "MinGW Makefiles"` instead of the plain `cmake -S .
-B build` above.

**Screenshot placeholder:** `07-no-hardware-path-01.png` — a successful build's terminal output.

## 2. Run it against your backend

With `ubibot-server` from [chapter 2](02-deploy-server.md) still running:

```bash
./build/ub_device_sim --host 127.0.0.1 --port 8080
```

With no arguments beyond `--host`/`--port`, it identifies itself as `sn_ws1_20001_1` — the exact
same demo device serial number the backend already seeds on first run (see
[chapter 2](02-deploy-server.md)), so it "just works" without any provisioning step. It then loops
forever: sample three sensors (`field1` temperature, `field2` humidity, `field3` light — random
walks, not real readings), report them, sleep, repeat. `Ctrl+C` to stop.

**Screenshot placeholder:** `07-no-hardware-path-02.png` — the simulator's console output showing
it reporting successfully.

> **Note:** want to see the auto-registration behavior for yourself instead of using the
> pre-seeded demo device? Pass your own `--sn something-else` — the protocol has no
> pre-registration step, so the server creates a brand-new device record the moment this
> simulated device's first report arrives.

## 3. Verify it, the same three checks

This stands in for [chapter 6](06-verify-device-online.md)'s checks, just without a WiFi step:

1. **The simulator's own console** shows it posting a report and getting `{"c":0,...}` back — no
   separate log viewer needed, unlike real hardware.
2. **The backend's log** (the terminal running `ubibot-server`) shows nothing alarming — errors
   would show up there if the report were rejected.
3. **The admin console's Device Management → Device** page shows `sn_ws1_20001_1` (or whichever
   `--sn` you passed) online, with a fresh reading visible on **Data Warehouse → My Data**.

**Screenshot placeholder:** `07-no-hardware-path-03.png` — the device list showing the simulated
device online.

## Next

You've reached the same point real hardware would get you to. Continue to Part B:
[Managing Devices and Products](08-managing-devices-and-products.md).
