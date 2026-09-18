# Troubleshooting

*[中文](13-troubleshooting.zh-CN.md)*

Grouped by what's actually broken. If your symptom isn't here, check the server's own terminal
output first — most failures log something useful there.

## Can't log in

| Symptom | Fix |
|---|---|
| Forgot the admin password | Stop `ubibot-server`, delete the admin table, restart it — this re-runs the "first run" logic from [chapter 2](02-deploy-server.md) and creates a fresh `admin` account with a newly generated password (device data is untouched): `sqlite3 data/ubibot.db "DELETE FROM admin_users;"` (adjust the path if you set `UBIBOT_DB_PATH`) |
| Login page never loads / blank page | Check the server's terminal for `embedded web UI not built — serving API only`. If you see it, the admin frontend was never built into this binary — run `./build.sh`/`.\build.ps1` (not just `go build`) and restart |

## Device never comes online

Work through [chapter 6](06-verify-device-online.md)'s three checks in order — device connects to
WiFi, then reports successfully, then shows up on the dashboard. If a report is being sent but
rejected, the response's `c` code tells you why:

| `c` code | Cause | Fix |
|---|---|---|
| `1002` | Timestamp outside the ±5-minute window | The device's clock has drifted — confirm it calls the time-sync endpoint to recalibrate after seeing this error |
| `1003` | Malformed request body | A firmware serialization bug — check it against [protocol §5](../protocol/hardware-communication-protocol.md#5-data-upload-the-only-device-facing-data-endpoint) |
| `1103` | Device disabled by an admin | Re-enable it from the device's Detail page — see [chapter 8](08-managing-devices-and-products.md) |
| `1900` | Rate limited | Reporting too frequently for the configured limit ([chapter 12](12-system-settings-and-monitor.md)'s Params page); wait for the next cycle, or check for an abnormal retry loop |

No response at all (not even a rejection)? Confirm the server host/port you provisioned is
correct, `ubibot-server` is actually running, and nothing (a firewall, a wrong LAN address) is
blocking the connection between the device and the machine running the backend.

## Sent a provisioning command, nothing happened

The provisioning window ([chapter 5](05-provision-device-over-serial.md)) only opens at
**power-on**, for a limited time (5 seconds of silence, or 60 seconds total, whichever comes
first) — sending after it's closed gets no response at all. Power-cycle or reset the device and
send the command again promptly.

## Using the simulator, connection refused

Confirm `ubibot-server` is actually running and listening on the host/port you passed to
`ub_device_sim --host`/`--port` — see [chapter 7](07-no-hardware-path.md). A device that's
supposed to reuse the pre-seeded demo `sn` but doesn't show up under that name usually means a
typo in `--sn`, not a connection problem — check the device list for whatever `sn` actually
arrived.

## Where to look for logs

| Piece | Where |
|---|---|
| Backend | The terminal `ubibot-server` (or `go run ./cmd/server`) is running in |
| Admin console | Your browser's developer console (network tab for failed API calls) |
| Real WS1B firmware | `idf.py monitor`, or ubibot-serial-sync's log panel |
| Simulator | Its own terminal — it prints directly, no separate log viewer |

## Still stuck?

The full error code table lives in the protocol doc's
[§8 Error Handling](../protocol/hardware-communication-protocol.md#8-error-handling); the admin
API's error response shape (including the stable `code` strings the admin console maps to
translated messages) is in [api/admin-api.md](../api/admin-api.md). Open an issue on the relevant
repo if neither explains what you're seeing.

---

This is the end of the User Manual. See the [Developer Handbook](../dev-guide/README.md) if
you're ready to start changing the platform's code instead of just operating it.
