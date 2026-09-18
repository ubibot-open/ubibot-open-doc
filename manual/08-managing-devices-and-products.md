# Managing Devices and Products

*[中文](08-managing-devices-and-products.zh-CN.md)*

**Device Management** has two pages: **Product** (display metadata for a device model) and
**Device** (the devices themselves — rename, enable/disable, delete, batch import/export).

## Products

A Product is purely a display label — a name and description for every device sharing a given
`pid`. Creating, renaming, or deleting one never touches any device row; it only changes what
shows up in the **Product** column on the device list.

Go to **Device Management → Product** and click **New Product**:

- **PID** — must match exactly what your devices actually report as their `pid` (see
  [chapter 4](04-flash-firmware.md)'s Device Identity config, or the simulator's `--pid`). This
  can't be changed after creation — delete and recreate if you got it wrong.
- **Name** / **Description** — whatever you want operators to see.

**Screenshot placeholder:** `08-managing-devices-and-products-01.png` — the New Product form.

Once created, every device reporting that `pid` immediately shows the product's name in its
**Product** column — no extra step, and no error if a device's `pid` doesn't match any Product
yet (it just shows no product name until one is created).

## Devices

**Device Management → Device** lists every device: ID, Name, SN, PID, Product, Status,
Online/Offline, and when it was last seen. Per-row actions:

- **Detail** — opens the device's own page (renaming, commands, alert rules, field settings — see
  [chapter 9](09-sending-commands.md) and [chapter 10](10-alerts.md)).
- **Rename** — sets the display `Name`; doesn't touch `sn`/`pid`.
- **Enable/Disable** — a disabled device is rejected by every device-facing endpoint; this is the
  only lever you have over an existing device short of deleting it outright.
- **Delete** — removes the device record.

**Screenshot placeholder:** `08-managing-devices-and-products-02.png` — the device list with a
few devices, showing the per-row actions.

### Batch import

Click **Import** to pre-register a whole batch of serial numbers ahead of time — useful right
after flashing a production run, before any of them have connected to WiFi yet. This is purely a
naming convenience: an unimported device still auto-registers itself the moment it first reports,
exactly the same as always; importing it first just means it already has the right name/PID from
the start instead of picking them up later.

Paste or upload a CSV with `sn,pid,name` per row (an optional header row starting with `sn` is
recognized and skipped; `name` may be left blank):

```csv
sn,pid,name
RV41554WS1B,ubibot-ws1b,Warehouse Sensor 1
RV41554WS1C,ubibot-ws1b,Warehouse Sensor 2
```

**Screenshot placeholder:** `08-managing-devices-and-products-03.png` — the import dialog with a
CSV pasted in and the parsed row count showing.

The dialog shows how many rows it parsed before you submit. After submitting, it reports how many
were **created**, **skipped** (an `sn` that already exists — not an error, just a no-op for that
row), and **failed** (missing `sn`/`pid`, or another problem) — a batch with a few bad rows still
succeeds for the rest instead of failing all-or-nothing.

**Screenshot placeholder:** `08-managing-devices-and-products-04.png` — the import result summary
after submitting.

### Export

Click **Export** to download every device (not paginated, unlike the on-screen list) as
`devices.csv`, columns: `id,pid,product_name,sn,name,status,online,last_seen_at,created_at`.

## Next

[Sending Commands to a Device](09-sending-commands.md).
