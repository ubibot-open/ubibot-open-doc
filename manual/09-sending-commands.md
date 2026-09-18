# Sending Commands to a Device

*[中文](09-sending-commands.zh-CN.md)*

Open a device's **Detail** page (from the device list) to find the **Device Commands** card, with
a note right above the controls: *"Delivered on the device's next report — fire-and-forget, no
delivery confirmation."* Read that literally — see "What "sent" actually means" below before you
rely on this for anything time-critical.

**Screenshot placeholder:** `09-sending-commands-01.png` — the Device Commands card, showing the
Reboot button and the set-interval form.

## Reboot

Click **Reboot**, confirm the popup. That's it — no parameters.

## Change report interval

Enter a value in seconds (60-86400, i.e. 1 minute to 1 day — the form rejects anything outside
that range) and click **Set Interval**.

## What "sent" actually means

Clicking either button doesn't talk to the device directly — there's no channel for that. It
queues the command on the server, and:

- The device picks it up the next time **it** calls in — piggybacked on its own next
  `POST /api/v1/data/report`, not delivered proactively. If the device reports every 5 minutes,
  expect up to a 5-minute delay; if it's offline, expect it to sit queued until it reconnects.
- **At most one command queued per device.** Sending a second one before the first is delivered
  simply replaces it — there's no queue of several pending commands.
- **The server has no way to confirm the device actually received or applied it.** The moment the
  command is handed to the device in a report response, the platform considers it delivered and
  clears it — a dropped network packet or a device that crashes right after looks identical to a
  successful delivery, from the admin console's point of view.

## Cancelling a queued command

While a command hasn't been delivered yet, the device's basic-info card shows a **Queued Command**
row with what's pending and a **Cancel** link.

**Screenshot placeholder:** `09-sending-commands-02.png` — the "Queued Command" row with its tag
and Cancel link.

Cancelling only works before delivery — once it's gone out on a report response, there's nothing
left to cancel (the platform has already cleared it, whether or not the device acted on it).

## Next

[Alerts](10-alerts.md).
