# UbiBot Open — Architecture Overview

*[中文](overview.zh-CN.md)*

This is the "how does it all fit together" document. For step-by-step setup see the
[deployment, flashing & bring-up guide](../guides/deployment-flashing-guide.md); for the exact
wire format see the [hardware communication protocol](../protocol/hardware-communication-protocol.md).
This page assumes you've read neither and just want the shape of the system first.

## The five repositories

| Repo | Role | Stack |
|---|---|---|
| [ubibot-open-server](https://github.com/ubibot-open/ubibot-open-server) | Device-facing HTTP backend + the admin console, compiled into one binary | Go, React/TypeScript |
| [ubibot-open-ws1b](https://github.com/ubibot-open/ubibot-open-ws1b) | Reference firmware for the WS1B sensor device | C, ESP-IDF (ESP32-C5) |
| [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync) | Desktop tool for serial log viewing, debugging, and device provisioning | C++, Qt 6/QML |
| [ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator) | From-scratch, host-buildable device simulator speaking the exact same protocol as the firmware — no hardware needed, and a compact worked example of the whole device-side protocol in ~7 files | C |
| ubibot-open-doc (this repo) | Everything above: protocol spec, deployment guide, this overview | Markdown |

Each repo is deployed and versioned independently. The only things that couple them are the wire
protocol (device ↔ backend, over HTTP) and a physical USB cable (PC ↔ device, over serial) — no
shared code, no shared build, no shared config file.

## System diagram

```
┌──────────────────────┐    USB serial: logs, debugging,     ┌───────────────────────┐
│  ubibot-serial-sync   │    provisioning (protocol §1.2)     │   ubibot-open-ws1b     │
│  (desktop tool)       │ ───────────────────────────────────▶│   firmware, on device  │
│                       │◀─────────────────────────────────── │                        │
└──────────────────────┘                                      └───────────┬────────────┘
                                                                           │ HTTP (protocol §2–§9)
                                                                           │ POST /api/v1/auth/time
                                                                           │ POST /api/v1/data/report
                                                                           ▼
┌──────────────────────┐    Bearer token, admin session       ┌───────────────────────┐
│   Admin's browser     │ ───────────────────────────────────▶│   ubibot-open-server   │
│  (the bundled React   │◀─────────────────────────────────── │  Go backend + embedded │
│   admin console)      │      /api/admin/*                   │  React admin console;  │
└──────────────────────┘                                      │  single binary, SQLite │
                                                                └───────────┬────────────┘
┌──────────────────────┐    X-Api-Key header                              │
│  Third-party client   │ ───────────────────────────────────▶ /api/open/v1/* (read-only)
│  (your own script/app)│◀─────────────────────────────────── ┘
└──────────────────────┘
```

Three independent "callers" hit the backend, each through its own auth mechanism and its own
route prefix — a device never touches an admin session, and an admin session never touches an API
key:

- **Devices** call the unauthenticated, pid+sn-identified `/api/v1/*` routes (protocol §2–§9).
- **The admin console** (and anyone driving the same API directly) calls `/api/admin/*` with a
  bearer token from `POST /api/admin/login`, gated per-route by RBAC permissions.
- **Third-party integrations** call the read-only `/api/open/v1/*` routes with an `X-Api-Key`
  header, issued and revocable from the admin console.

See [api/admin-api.md](../api/admin-api.md) and [api/open-api.md](../api/open-api.md) for the
full route tables.

## Data flow: device to dashboard

1. **Identity**: a device knows its own `pid` (shared by every unit of that model) and `sn`
   (unique per unit) — either baked in at build time via `idf.py menuconfig`, or set afterward
   over serial (`SetupDevice`, protocol §1.2) without reflashing. There's no registration step:
   the backend creates a `Device` row the moment an unrecognized `sn` first reports, unless an
   admin already pre-registered it via batch import.
2. **Network config**: WiFi credentials and the server address are the same story — a
   `menuconfig` default, overridable at runtime over serial (`SetupWifi`/`SetupServer`).
3. **Time sync** (only if the device's clock looks wrong): `POST /api/v1/auth/time` returns the
   server's clock; no auth, since it reveals nothing device-specific.
4. **Report**: `POST /api/v1/data/report` carries one or more timestamped `field1`..`field20`
   readings. The backend upserts the device, stores the readings (merging entries within a
   60-second window into one record, protocol §5), and — if an admin has queued one — embeds a
   `cmd` object in the response (protocol §9: `reboot` or `set_interval`). The device acts on it
   before its next sleep cycle; there's no ack back to the server either way.
5. **Dashboard**: the admin console polls/queries the same SQLite rows the report handler just
   wrote — device list, "数据仓库" (each device's latest reading inlined), per-device history,
   threshold/offline alerts evaluated against the same data.

## Backend internal layout

```
server/
  cmd/server/       main.go — flag/env parsing, NVS-equivalent bootstrap (seed the first admin
                     account, default system params), wires everything together
  internal/
    api/            HTTP handlers + the router (net/http's ServeMux, Go 1.22 method+{wildcard}
                     routing — no framework)
    auth/           password hashing (bcrypt) and admin session tokens
    model/          GORM row types — the durable schema
    protocol/       device-facing wire types (mirrors the protocol doc almost 1:1)
    store/          persistence — every query lives here, handlers never touch gorm.DB directly
    webui/          embeds admin/'s Vite build output into the Go binary via go:embed
admin/              the React admin console (Vite + TypeScript + Ant Design), built separately
                     and embedded into the server binary by build.sh/build.ps1
```

A few things worth knowing before reading the code:

- **Single SQLite file, GORM `AutoMigrate`.** There's no migrations directory — adding a column
  to a `model` struct is enough; GORM adds it to the existing table non-destructively on next
  start. (Renaming a Go struct field can silently change its mapped column name in ways worth
  double-checking — see the `Product.PID` → `column:pid` tag for a case where GORM's naming
  strategy didn't do what a plain reading of the field name would suggest.)
- **A mockable clock, not `time.Now()` sprinkled everywhere.** Handlers read `s.Now()`; tests
  swap it for a fixed function so time-window logic (the ±5-minute report check, offline-alert
  grace periods) is deterministic instead of racing the wall clock.
- **Tests hit the real router**, not the store directly — `httptest.NewRecorder()` +
  `router.ServeHTTP()` against an in-memory SQLite DB per test, so a test exercises the same auth/
  permission/handler chain a real request would.

## Firmware internal layout (ubibot-open-ws1b)

```
main/
  main.c            app_main() wiring + the one long-lived task: connect → (maybe time-sync) →
                     sample → report → sleep, repeated forever via deep sleep + timer wake-up
  provisioning.c    protocol §1.2 — the serial commands that override WiFi/server/sn at runtime,
                     persisted in NVS, read back into RAM every boot (deep sleep wipes RAM)
  command.c         protocol §9 — the server-issued reboot/set_interval commands, same
                     NVS-override pattern as provisioning.c, kept as a separate module because
                     it's a different actor (server, not a technician) issuing a different kind
                     of instruction
  json_payload.c    builds the two device-facing request bodies (time-sync, data-report)
  osi.c             thin FreeRTOS wrappers (task/queue/mutex) the rest of main/ builds on
app_driver/         peripheral drivers: WiFi (wifi_connect.c), HTTP client (http_client.c),
                    sensors (sht30/ltr308/ub_dt_p1/stk8323), power/sleep management
```

The recurring pattern worth noticing: **every runtime-overridable setting is "Kconfig default,
overridden by an NVS value if one was ever written, reloaded into a RAM variable every boot."**
`provisioning.c` and `command.c` both implement this same shape independently for different
triggers (a technician over serial vs. the server via a report response) rather than sharing one
generic "settings" abstraction — deliberately duplicated to keep each module's blast radius small
(see each file's own header comment for its own rationale).

## Design principles this codebase leans on

These aren't written down in one place elsewhere, so worth stating explicitly:

- **Minimal by default, on both sides of the wire.** The device protocol has no signing, no
  session tokens, no general command-push channel (protocol doc §0) — and the same reasoning was
  extended to the admin console's login (no CAPTCHA/lockout/forced-logout) when it came up as a
  candidate feature. This is an internal/educational tool aimed at being usable with as little
  setup as possible, not a hardened multi-tenant SaaS.
- **Fire-and-forget over ack/retry.** Serial provisioning acks are informational only; command
  delivery (protocol §9) has no ack at all. Simpler to implement and reason about, at the
  explicit cost of no delivery guarantee — documented as a trade-off, not hidden.
- **Display metadata is never a foreign key.** `Product` and a device's resolved `product_name`
  are matched by `pid` string equality at read time, not a `product_id` column on `Device` —
  deleting or renaming a Product can never orphan or cascade into device rows.
- **Scope grows only when something concrete needs it.** OTA, BLE provisioning, and a
  sensor-abstraction layer for a hypothetical second device model are all deliberately out of
  scope for now — not because they're hard, but because nothing in this project currently needs
  them (see the protocol doc's "capabilities removed" appendix for the longer list of things
  simplified away on purpose).

## Where to go next

- [Hardware Communication Protocol](../protocol/hardware-communication-protocol.md) — the exact
  device↔server wire format.
- [Deployment, Flashing & Bring-up Guide](../guides/deployment-flashing-guide.md) — hands-on
  setup for all three components.
- [Admin API Reference](../api/admin-api.md) / [Open API Reference](../api/open-api.md) — every
  HTTP route the backend exposes.

