# Alerts

*[中文](10-alerts.zh-CN.md)*

Alerts have two separate places in the console: **rules** are configured per device, on that
device's own Detail page; **events** — occurrences of a rule firing, or a device going offline —
are reviewed and resolved from the standalone **Alert** page.

## Creating a rule

Open a device's **Detail** page (**Device Management → Device → [a device] → Detail**) and find
the **Alerts** card. Add a rule with three parts:

- **Field** — the field key to watch, typed as text (e.g. `field1`) — whatever key that device
  actually reports.
- **Operator** — one of `>`, `>=`, `<`, `<=`, `==`.
- **Threshold** — the number to compare against.

**Screenshot placeholder:** `10-alerts-01.png` — the Alerts card on a device's Detail page, with
the rule-creation form and an existing rule listed below it.

A rule is checked against the **latest value of that field** every time the device reports —
there's no separate schedule or polling; a violating reading triggers (or keeps open) an alert
event the moment it's received.

> **Note:** a still-violating reading doesn't spam duplicate events. If an event for a given
> rule is already open, another violating reading just leaves it open — you'll never see two open
> events for the same rule on the same device at once.

## Offline alerts

Not something you configure — a device going offline (no report within the grace period) raises
an alert event automatically, checked by a periodic background sweep rather than tied to any
report. It shows up in the same **Alert** page as a rule-triggered one.

## Reviewing and resolving events

Go to **Alert** (top-level menu item, not inside a device). Filter by device and by status
(**Open** / **Resolved** / **All**); click **Resolve** on an open one once you've dealt with it.

**Screenshot placeholder:** `10-alerts-02.png` — the Alert page, showing a mix of open and
resolved events with the filters visible.

## Next

[Users, Roles, and API Keys](11-users-roles-and-api-keys.md).
