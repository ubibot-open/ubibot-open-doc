# Verify the Device Is Online

*[中文](06-verify-device-online.zh-CN.md)*

Three checks, in order — if one fails, fix it before moving to the next.

## 1. Device side: connected to WiFi

In `idf.py monitor` (still running from [chapter 4](04-flash-firmware.md)) or
`ubibot-serial-sync`'s log panel, look for a line like "WiFi connected" followed by the device
getting an IP address.

**Screenshot placeholder:** `06-verify-device-online-01.png` — the serial log showing a
successful WiFi connection.

**Not seeing it?** Double check the WiFi SSID/password/country code you provisioned in
[chapter 5](05-provision-device-over-serial.md), and that the router/access point is actually
reachable and broadcasting. See [chapter 13](13-troubleshooting.md) if it still doesn't connect.

## 2. Network & protocol: a report is accepted

Still in the same log, look for a call to `POST /api/v1/data/report` followed by a
`{"c":0,...}` response.

**Screenshot placeholder:** `06-verify-device-online-02.png` — the log line(s) showing the report
request and its `{"c":0,...}` response.

**Stuck with no response or a timeout?** Confirm the backend from [chapter 2](02-deploy-server.md)
is still running, that the server host/port you provisioned actually points at the machine it's
running on (its LAN IP, reachable from the device — not `localhost`), and that a firewall on that
machine isn't blocking the port.

## 3. Visible on the dashboard

Back in the admin console (`http://<backend address>:8080`), open **Device Management → Device**.
The serial number you provisioned should already be there — the protocol auto-registers a device
the moment it first reports successfully, with nothing to pre-create in the console.

**Screenshot placeholder:** `06-verify-device-online-03.png` — the device list, with the new
device visible and marked online.

Click into it, or check **Data Warehouse → My Data** — its latest reading should be visible
there too.

**Screenshot placeholder:** `06-verify-device-online-04.png` — the device's detail page or its
row in "My Data", showing a real reading.

If all three check out, the full pipeline — device online → reporting → visible on the dashboard —
is working end to end. Continue to Part B to explore the rest of the console, starting with
[Managing Devices and Products](08-managing-devices-and-products.md).

## Next

[Managing Devices and Products](08-managing-devices-and-products.md).
