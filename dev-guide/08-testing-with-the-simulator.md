# Iterate on the Backend with the Simulator

*[中文](08-testing-with-the-simulator.zh-CN.md)*

Flashing real WS1B hardware to test a backend change means a build-flash-wait-watch-serial-log
loop measured in minutes. [ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator)
turns that into seconds — it's a plain process on the same machine as your backend, so restarting
it, or running several at once, costs nothing.

## The basic loop

```bash
# terminal 1: the backend, with hot reload via go run
cd ubibot-open-server/server && go run ./cmd/server

# terminal 2: a simulated device
cd ubibot-open-simulator && ./build/ub_device_sim --host 127.0.0.1 --port 8080
```

Changing backend code needs a `Ctrl+C` + re-run of `go run ./cmd/server` in terminal 1; the
simulator in terminal 2 doesn't need touching at all unless the change is to the *protocol itself*
(request/response shape) rather than to backend-only logic (a new admin endpoint, a store query, a
DB column) — the simulator keeps reporting through the restart, so you see the very next report
hit your new code without doing anything on the device side.

## Simulating several devices at once

Run more than one instance, each with its own `--sn` (same `--pid` if you want them to share a
Product):

```bash
./build/ub_device_sim --host 127.0.0.1 --port 8080 --sn test-a &
./build/ub_device_sim --host 127.0.0.1 --port 8080 --sn test-b &
./build/ub_device_sim --host 127.0.0.1 --port 8080 --sn test-c &
```

This is the fastest way to get a realistic-looking device list for testing anything
list/pagination/CSV-shaped — [batch import/export](../manual/08-managing-devices-and-products.md),
the [Product](02-backend-add-admin-endpoint.md) resolution across several devices, the dashboard's
online-count, and so on — without either waiting on real hardware or hand-inserting rows into the
SQLite file.

## About the sample/report timing

Each simulated device samples every 30s and reports every 300s
(`UB_DEFAULT_SAMPLE_INTERVAL_SEC`/`UB_DEFAULT_REPORT_INTERVAL_SEC` in `app/src/ub_device.c`) —
real elapsed time, not simulated time, and not affected by `--tick-ms` (that flag only controls
how often the main loop *checks* whether it's time to sample/report yet, i.e. how promptly it
reacts — lowering it doesn't make reports arrive faster). If you're testing something that
specifically needs faster reporting (e.g. the backend's offline-detection sweep, or watching a
[server-issued command](07-firmware-add-server-command.md) get delivered without a five-minute
wait), temporarily lower `UB_DEFAULT_REPORT_INTERVAL_SEC` and rebuild — don't commit that change,
it exists only for your local test run.

## Manual protocol-level testing

Two tools exist for poking at the protocol directly, both outside the normal build:

- **`test/mock_server.py`** — a minimal Python HTTP server implementing just the two device
  endpoints, useful for testing the *simulator's own* behavior (retry on failure, buffer handling)
  without a real backend in the loop at all.
- **`test/test_integration.c`** — drives the simulator's own C protocol codec over a real socket
  against a live `ubibot-open-server` (`go run ./cmd/server` first, then
  `./build/test_integration [host] [port]`) — the thing a pure unit test can't reach without an
  actual TCP connection. Not wired into `ctest`/`make test` since it needs that live server; run it
  manually.

## What the simulator can't help with

It doesn't implement the [serial provisioning commands](06-firmware-add-serial-command.md) (no
serial port to speak of) or the [server-issued command channel](07-firmware-add-server-command.md)
(`Command_HandleResponse` has no counterpart in `ub_device.c`) — those need real firmware on real
hardware to test end to end. It's a stand-in for the *device-to-backend* half of the system, not
for the firmware itself.

## Next

[Changing the Protocol](09-protocol-changes.md).
