# UbiBot Open — Developer Handbook

*[中文](README.zh-CN.md)*

Task-oriented walkthroughs for changing the platform's code, not just running it. Start with the
[Architecture Overview](../architecture/overview.md) if you haven't already — it explains how the
pieces fit together; this handbook assumes that context and gets straight to "how do I add X".

> **Status:** Outline only. Each linked chapter below is a stub — the writing is happening one
> chapter at a time; see each page for its current state.

## Chapters

0. [Overview](00-overview.md)
1. [Environment Setup](01-environment-setup.md)
2. [Add a New Admin API Endpoint](02-backend-add-admin-endpoint.md)
3. [Add a New Data Model](03-backend-add-model-and-migration.md)
4. [Add a New Admin Console Page](04-admin-add-page.md)
5. [Firmware: Wire In a New Sensor](05-firmware-add-sensor.md)
6. [Firmware: Add a Serial Provisioning Command](06-firmware-add-serial-command.md)
7. [Firmware: Add a Server-Issued Command](07-firmware-add-server-command.md)
8. [Iterate on the Backend with the Simulator](08-testing-with-the-simulator.md)
9. [Changing the Protocol](09-protocol-changes.md)
10. [Contributing and PR Conventions](10-contributing-and-pr-conventions.md)

## See also

- [User Manual](../manual/README.md) — for operating the platform, not modifying it.
- [Architecture Overview](../architecture/overview.md), [Hardware Communication Protocol](../protocol/hardware-communication-protocol.md),
  [Admin API Reference](../api/admin-api.md), [Open API Reference](../api/open-api.md).
- [org-wide CONTRIBUTING.md](https://github.com/ubibot-open/.github/blob/main/CONTRIBUTING.md).
