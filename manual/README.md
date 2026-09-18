# UbiBot Open — User Manual

*[中文](README.zh-CN.md)*

A tutorial-style walkthrough from zero to a working platform, with screenshots for every step.
If you already know your way around a shell and just want the commands, the
[deployment, flashing & bring-up guide](../guides/deployment-flashing-guide.md) is a faster,
denser reference covering the same ground. This manual is for the first time through.

> **Status:** Chapters 1–4 are written. The rest are outline-only stubs for now — the writing is
> happening one chapter at a time; see each page for its current state.

## Part A — Getting Started

Follow these in order. No WS1B hardware yet? Skip straight from chapter 3 to chapter 7 — the
simulator gets you the same end state.

1. [Overview](00-overview.md)
2. [Prerequisites](01-prerequisites.md)
3. [Deploy the Backend Server](02-deploy-server.md)
4. [First Login and a Tour of the Console](03-first-login-and-tour.md)
5. [Flash the WS1B Reference Firmware](04-flash-firmware.md) — has real hardware
6. [Provision the Device Over Serial](05-provision-device-over-serial.md) — has real hardware
7. [Verify the Device Is Online](06-verify-device-online.md) — has real hardware
8. [No Hardware? Use the Simulator](07-no-hardware-path.md) — no hardware

## Part B — Using the Admin Console

Day-to-day operation, once the platform is up and at least one device (real or simulated) is
reporting.

9. [Managing Devices and Products](08-managing-devices-and-products.md)
10. [Sending Commands to a Device](09-sending-commands.md)
11. [Alerts](10-alerts.md)
12. [Users, Roles, and API Keys](11-users-roles-and-api-keys.md)
13. [System Settings and Monitor](12-system-settings-and-monitor.md)
14. [Troubleshooting](13-troubleshooting.md)

## Screenshots

Each chapter's images live under [`assets/`](assets/), named `<chapter-slug>-01.png`,
`<chapter-slug>-02.png`, etc., in the order they appear on the page. Screenshots are shared
between the English and Chinese versions of a chapter unless the on-screen text itself is what's
being illustrated.

## See also

- [Developer Handbook](../dev-guide/README.md) — for modifying the platform's code, not just
  operating it.
- [Architecture Overview](../architecture/overview.md), [Hardware Communication Protocol](../protocol/hardware-communication-protocol.md),
  [Admin API Reference](../api/admin-api.md), [Open API Reference](../api/open-api.md).
