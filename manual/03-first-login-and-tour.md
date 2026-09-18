# First Login and a Tour of the Console

*[中文](03-first-login-and-tour.zh-CN.md)*

You should already be logged in from the end of [chapter 2](02-deploy-server.md). This chapter is
a guided walk through every area of the console before any devices exist — so you know what
you're looking at once one comes online.

## The top bar

**Screenshot placeholder:** `03-first-login-and-tour-01.png` — the top bar, with each control
below visible.

Left to right:

- **Menu fold toggle** — collapses the left sidebar to icons only, for more screen space.
- **Breadcrumb** — shows where you are in the menu tree; also clickable.
- **Theme toggle** — switches between light and dark mode.
- **Language switcher** — English, 简体中文, and 日本語 are built in; switching here changes every
  label in the console immediately, no reload needed.
- **Notification bell** — a popover listing recent system notifications (e.g. an alert firing),
  with mark-as-read.
- **Account dropdown** — your username, a link to account settings, and log out.

## The left menu

### Dashboard

The landing page. Four stat cards — total devices, devices currently online, open alerts, and
records received today — plus a 7-day chart of daily telemetry volume. On a brand-new install
every number here is zero.

**Screenshot placeholder:** `03-first-login-and-tour-02.png` — the empty dashboard.

### Data Warehouse

Two related but distinct pages:

- **My Data** — every device you have, as a searchable list or grid, each row showing its latest
  reading and online/offline status; filter by keyword or online state, export the current view
  to CSV, click a device to drill into a per-field history chart.
- **Monitor** — pick one device and a date/time range to browse its raw historical records in a
  table — the tool you reach for when you need exact values for a specific window, not just the
  latest reading.

### Device Management

- **Device** — the full device list (same data source `My Data` reads from, but oriented around
  managing devices rather than browsing their readings): rename, enable/disable, delete, send a
  command, edit per-field display settings and alert rules, and the batch import/export tools —
  covered in [chapter 8](08-managing-devices-and-products.md).
- **Product** — display-only name/description metadata for a device model, matched onto devices
  by `pid`. Also covered in chapter 8.

### Alert

Two tabs: the alert rules you've configured per device/field, and the alert events they've fired
(with a resolve action). Covered in [chapter 10](10-alerts.md).

### System

A group of administrative pages:

| Page | What it's for |
|---|---|
| Admin | Manage other admin accounts (create, change role, deactivate) |
| Role | Define roles and which of the four permissions (`device:read`, `device:write`, `alert:manage`, `system:manage`) each one grants |
| Log | The audit log — every mutating action any admin has taken, who did it, and when |
| API Key | Issue and revoke keys for the read-only `/api/open/v1/*` third-party API — see [chapter 11](11-users-roles-and-api-keys.md) |
| Files | Uploaded file assets (e.g. images referenced elsewhere in the console) |
| Dict | Dictionary entries — shared lookup lists used elsewhere in the console |
| Params | Runtime-tunable system parameters (e.g. the device-facing API's rate limit, the offline-detection grace period) — changing one here takes effect immediately, no restart |
| Monitor | Server health: Go runtime version, goroutine count, memory use, uptime, database file size |
| Icons | The icon library used as the default display icon for `field1`-`field20` when a device hasn't customized its own |

**Screenshot placeholder:** `03-first-login-and-tour-03.png` — the System group's menu expanded.

## Next

With real hardware: [Flash the WS1B Reference Firmware](04-flash-firmware.md). Without: skip to
[No Hardware? Use the Simulator](07-no-hardware-path.md).
