# Firmware: Add a Server-Issued Command

*[中文](07-firmware-add-server-command.zh-CN.md)*

Protocol §9's command channel is deliberately narrow — an admin can queue `reboot` or
`set_interval`, fire-and-forget, delivered piggybacked on the device's next report — but adding a
third action follows a fixed round trip across three repos. This chapter walks the whole loop
using `set_interval` (the more interesting of the two existing actions) as the reference.

## The round trip, end to end

```
admin console button/form
        │  POST /api/admin/devices/{id}/commands  {"action":"set_interval","seconds":600}
        ▼
   admin API handler  ──── validates, then ────▶  Device.PendingCmd  (one column, one device, one queue slot)
        │                                                   │
        │                                     device's next POST /api/v1/data/report
        │                                                   ▼
        │                                   Report() pops PendingCmd, embeds it as "cmd"
        ▼                                                   │
  (admin sees pending_command                                ▼
   on the device until delivered)              firmware's Command_HandleResponse() acts on it
```

There is no ack anywhere in this diagram — the admin API call succeeds the moment the command is
queued, and the firmware never reports back whether it actually applied it. That's intentional
(see [Changing the Protocol](09-protocol-changes.md) for the reasoning); build your new action
with the same expectation.

## 1. Backend: queuing and delivering

One column on `Device` holds the queue (`server/internal/model/model.go`):

```go
// PendingCmd is a queued, not-yet-delivered command for this device
// (docs §9), stored as the exact JSON object to embed in the next
// report response's "cmd" field (e.g. {"action":"reboot"}); empty
// means nothing queued. At most one command at a time — setting a new
// one overwrites whatever hadn't been delivered yet. Delivery clears
// it (see store.PopPendingCommand): fire-and-forget, no ack.
PendingCmd string `gorm:"type:text"`
```

`SendDeviceCommand` (`server/internal/api/admin_handlers.go`) is a `switch` on the requested
action — this is exactly where a third action gets a new `case`:

```go
var cmdJSON []byte
switch req.Action {
case "reboot":
	cmdJSON, _ = json.Marshal(map[string]any{"action": "reboot"})
case "set_interval":
	if req.Seconds < minReportIntervalSeconds || req.Seconds > maxReportIntervalSeconds {
		adminErr(w, 400, fmt.Sprintf("seconds must be between %d and %d", minReportIntervalSeconds, maxReportIntervalSeconds))
		return
	}
	cmdJSON, _ = json.Marshal(map[string]any{"action": "set_interval", "seconds": req.Seconds})
default:
	adminErr(w, 400, "unsupported action")
	return
}
// ...s.Store.SetPendingCommand(uint(id), string(cmdJSON))
```

Notice `set_interval` bounds `seconds` to 60-86400 **on the backend** — the code comment explains
why: "the firmware doesn't enforce this range itself — it takes whatever seconds value it's told —
so it's on the admin API to keep it sane before it's ever queued." Any new action with a numeric
or otherwise unbounded parameter should validate it here for the same reason: assume the firmware
will apply whatever it receives without question.

Delivery is generic and needs no changes for a new action — `Report()`
(`server/internal/api/device_handlers.go`) just pops whatever is queued and embeds it verbatim:

```go
resp := protocol.ReportResponse{C: protocol.CodeOK, T: now.Unix()}
if cmdJSON, err := s.Store.PopPendingCommand(dev.ID); err != nil {
	log.Printf("pop pending command for device %d: %v", dev.ID, err)
} else if cmdJSON != "" {
	resp.Cmd = json.RawMessage(cmdJSON)
}
```

## 2. Firmware: acting on it

`main/command.c`'s `Command_HandleResponse()` is the mirror image of the backend's `switch` — an
`if`/`else if` chain on `cmd.action`:

```c
// main/command.c
if (strcmp(action->valuestring, "reboot") == 0)
{
  esp_restart();  // never returns
}
else if (strcmp(action->valuestring, "set_interval") == 0)
{
  cJSON *seconds = cJSON_GetObjectItem(cmd, "seconds");
  if (!cJSON_IsNumber(seconds) || (seconds->valuedouble <= 0))
  {
    ESP_LOGW(TAG, "set_interval command missing a positive \"seconds\"");
    return;
  }
  command_save_report_interval((uint32_t)seconds->valuedouble);
}
else
{
  ESP_LOGW(TAG, "unrecognized cmd.action: %s", action->valuestring);
}
```

A new action gets its own `else if` branch here, following `set_interval`'s shape if it needs to
persist something (open NVS, write, commit, close, then update the RAM-resident value — the exact
same pattern [Add a Serial Provisioning Command](06-firmware-add-serial-command.md) covers for
`provisioning.c`), or `reboot`'s (nothing to persist, just act) if it doesn't.

Two wiring points, easy to get right since there's exactly one of each:

- **Where it's invoked**: `Command_HandleResponse()` is called from
  `Parse_Response()` in `app_driver/net/http_client.c`, on every parsed HTTP response — it's a
  no-op when there's no `cmd` field (e.g. a time-sync response), so nothing extra is needed here
  for a new action.
- **Where a persisted value is read back**: if your action changes some ongoing behavior (like
  `set_interval` does), add a `Command_Get*` accessor to `command.h`/`.c` and call it from wherever
  that behavior actually happens — `main.c`'s sleep call is the existing example:

  ```c
  Enter_Sleep(Command_GetReportIntervalSeconds());  // interval is the server-commanded value if one was ever set, else DEFAULT_FN
  ```

## 3. The test

`command_handlers_test.go` verifies the whole round trip through the real HTTP surface on both
ends — queue via the admin API, confirm it shows as `pending_command`, report as the device would,
confirm the response carries it, confirm it's gone on the report after that:

```go
// server/internal/api/command_handlers_test.go
func TestSendDeviceCommand_RebootDeliveredOnceThenCleared(t *testing.T) {
	env := newTestEnv(t)
	adminAuth := env.createSuperAdmin(t, "admin", "s3cret-pw")

	rec, body := env.do(t, "POST", fmt.Sprintf("/api/admin/devices/%d/commands", env.dev.ID),
		map[string]any{"action": "reboot"}, adminAuth)
	// ...assert rec.Code == 200

	// Visible as pending before it's ever delivered.
	rec, body = env.do(t, "GET", fmt.Sprintf("/api/admin/devices/%d", env.dev.ID), nil, adminAuth)
	// ...assert dev["pending_command"] != nil

	// First report after queuing: the response carries the command.
	rec, body = env.do(t, "POST", "/api/v1/data/report",
		report(testSN, env.now.Unix(), map[string]any{"field1": 20}), nil)
	// ...assert body["cmd"] == {"action":"reboot"}
}
```

There's no equivalent test on the firmware side (no host-run test suite for `ubibot-open-ws1b`, as
noted in [Environment Setup](01-environment-setup.md)) — verify a new action by building, flashing
to real hardware (or reading the code path carefully if you don't have a unit handy — the
simulator doesn't implement the command channel, since it's an admin-console-triggered feature,
not something a bare device simulator needs to exercise), queuing the command from the admin
console, and watching the device's serial log after its next report.

## Checklist

1. Add the action to `SendDeviceCommand`'s `switch` (backend), validating any parameter with a
   sensible bound — the firmware won't.
2. Add the same action to `Command_HandleResponse`'s `if`/`else if` chain (firmware).
3. If it changes ongoing behavior, add a `Command_Get*` accessor and call it from where that
   behavior lives.
4. Add a test following `TestSendDeviceCommand_RebootDeliveredOnceThenCleared`'s shape.
5. Document the new action in `protocol/hardware-communication-protocol.md` §9.
6. If the admin console needs a new form field to trigger it, see
   [Add a New Admin Console Page](04-admin-add-page.md) for the UI-side pattern.

## Next

[Iterate on the Backend with the Simulator](08-testing-with-the-simulator.md).
