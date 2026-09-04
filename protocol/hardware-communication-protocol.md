# UbiBot Open Platform Hardware Communication Protocol

### 0. Project Scope & Rationale for This Revision

This project (ubibot-open) is an open-source IoT platform meant for internal/educational use — it
does not aim for completeness, reliability, or production-grade security. The primary goal is:
let a user connect a piece of hardware and see data with as few steps and as little configuration
as possible. Any complex auth flow, state machine, or command-dispatch channel that isn't strictly
necessary for the core "get data flowing" path has been stripped out.

Compared to earlier versions, this revision:

- Targets a trusted internal-network environment, so device identity authentication has also been
  simplified away: there's no more DeviceSecret or signature computation — a device identifies
  itself with a plain-text pid+sn, with no need to pre-create the device on the platform or go
  through a "self-activation approval / key binding" flow. Power it on, get it online, and it can
  report data.
- Removes the separate "activation" step (device activation / token session): a data-upload
  request identifies itself with pid+sn directly and completes the write in one request. A
  lightweight, unauthenticated time-sync endpoint is kept, so a device with no local clock can
  fetch the current time.
- Removes the original, much larger command-dispatch channel: config polling, custom probe
  configuration, and OTA are all still out of scope, not necessary for the core "get data flowing"
  goal and rarely needed in teaching/demo scenarios. A deliberately narrow exception was added
  back later (§9): an admin can queue a `reboot` or `set_interval` command, delivered piggybacked
  on the device's own next report with no ack — not a general push channel, just enough to avoid
  a physical reflash for those two things.
- Uploaded data no longer uses named fields like `temperature`/`humidity` — it's unified into
  `field1`–`field20` (20 max), which the platform stores as-is without caring about their business
  meaning. `field1`/`field2`/`field3` carry a conventional default meaning (temperature/humidity/
  light); the rest are entirely up to the user to define.

The rest of this document describes the new, simplified protocol.

### 1. Device Provisioning (first-time network setup)

A device fresh from the factory or after a factory reset has no local WiFi configuration and
needs to be provisioned — connected to WiFi and pointed at the correct server address — before it
can report data using the HTTP protocol described in the sections below.

#### 1.1 Bluetooth Provisioning

**Not currently supported.** No Bluetooth (BLE) based provisioning method is provided; on-site
WiFi and server configuration is done exclusively through serial provisioning (§1.2) below.

#### 1.2 Serial Provisioning

The device accepts a set of text commands over its USB/UART serial port. Paired with
`ubibot-serial-sync` (or any serial terminal at the same baud rate as the firmware's log output —
115200 by default on the ESP-IDF reference firmware), this lets you set WiFi and server address
on site without recompiling or reflashing. Commands are single-line, UTF-8-encoded JSON sent over
the serial port.

**Set the network**

```json
{"command":"SetupWifi","ssid":"","password":"","type":""}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| command | string | Yes | Fixed value `SetupWifi` |
| ssid | string | Yes | WiFi network name |
| password | string | No | WiFi password; may be omitted or empty for an open network |
| type | string | Yes | WiFi authentication/encryption type (the exact accepted values are defined by the firmware implementation, e.g. `WPA2`/`OPEN`) |

**Set the server address**

```json
{"command":"SetupServer","host":"127.0.0.1","port":8080}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| command | string | Yes | Fixed value `SetupServer` |
| host | string | Yes | Server IP address or domain name — the backend §2 (Transport) talks to |
| port | number | Yes | Server listening port |

**Response (ACK)**: after processing each command, the device writes back one line of JSON over
the same serial port:

```json
{ "c": 0, "msg": "wifi saved" }
```

| Field | Description |
| --- | --- |
| c | 0 = success; 1 = validation failed (missing field, wrong type, too long, etc.); 2 = validation passed but the persistent save failed (e.g. a storage write error) |
| msg | A short explanation — success or failure reasons may both appear here; not intended as a machine-parseable enum |

**When it takes effect**: once saved successfully, the new value is used **immediately, within
the current run** — no extra reboot needed; the next attempt to connect to the network / report
data already uses it. It's also written to non-volatile storage (e.g. NVS), so it survives power
loss and deep-sleep wake-ups until overwritten again.

**Provisioning window**: the device doesn't need to listen for serial commands all the time. The
reference implementation only opens a time-limited window at **power-on** (closing early after a
period of silence following the last byte received, with a hard cap regardless of activity); once
the window closes, the device proceeds with its normal connect/sleep cycle. A periodic timer
wake-up (as opposed to power-on) does not reopen the window, so routine reporting cycles aren't
kept waiting. The exact duration, and whether to open the window outside of power-on, is left to
the firmware implementation — this protocol does not mandate it.

The response format and effective-timing behavior above reflect the `ubibot-open-ws1b` reference
firmware's implementation (see
[main/provisioning.c](https://github.com/ubibot-open/ubibot-ws1b/blob/main/main/provisioning.c)
in that repository). Other firmware implementations are encouraged to follow the same convention
for consistency, but it isn't mandatory.

### 2. Transport

| Item | Requirement |
| --- | --- |
| Protocol | HTTP |
| Method | POST |
| Data format | JSON, UTF-8 |
| Content-Type | application/json |

Note: this protocol performs no authentication whatsoever — it fully trusts the pid+sn in the
request. It's only suitable for a trusted internal-network environment; do not apply this scheme
directly to a deployment that needs real security or is exposed on the public internet.

### 3. Device Identity

A device identifies itself with just two fields, both sent in plain text, with no key/signature
required:

| Field | Description |
| --- | --- |
| pid (ProductID) | Product model, used to distinguish device types |
| sn (SerialNumber) | The device's unique serial number |

### 4. Time Sync

When a device has no local clock (first power-on, or an RTC reset from power loss), it can call
this endpoint to fetch the server's current time. This endpoint performs no identity check at all
— no signature required or verified.

**POST /api/v1/auth/time**

Request:

```json
{ "pid": "ubibot_open_dev_v1", "sn": "sn_ws1_20001_1"}
```

Response:

```json
{ "c": 0, "t": 1788950400 }
```

| Field | Description |
| --- | --- |
| t | Server's current Unix timestamp, in seconds |

### 5. Data Upload (the only device-facing data endpoint)

**POST /api/v1/data/report**

Request body: a single report can carry multiple timestamped readings (e.g. several rounds of
data buffered while the device was offline):

```json
{
  "pid": "ubibot_open_dev_v1",
  "sn": "sn_ws1_20001_1",
  "ts": 1788950400,
  "payloads": [
    { "ts": 1788950400, "field1": 25.6, "field2": 60.2 },
    { "ts": 1788951000, "field1": 25.8, "field2": 59.9, "field3": 812 }
  ]
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| pid | string | Yes | Product model |
| sn | string | Yes | Device serial number — the sole basis for identifying the device (no signature check) |
| ts | Int64 | Yes | Unix timestamp (seconds) when this request was made, used for a basic time-window check (optional check, see §8) |
| payloads | array | Yes | One or more records. Typically one record corresponds to one sample point and carries every field for that moment; it's also fine to split a single sampling round across multiple records, each carrying only one or a few fields (see "Merging by time window" below) — both styles are handled equivalently by the server |
| payloads[].ts | Int64 | Yes | Unix timestamp (seconds) for this sample point (may be earlier than the outer `ts`, for backfilling data buffered while offline) |
| payloads[].field1~field20 | number | Yes | `field1`–`field20` → numeric value, see §6. Omit unused field numbers entirely — don't send an empty value |

**Merging by time window**: it is not required that "one record = one complete sample point" —
fields from the same sampling round may be split across multiple records (e.g. a sensor read
sequentially, with each field's `ts` off by a few seconds). The server groups `payloads` by `ts`:
after sorting, it starts a new group at the first ungrouped record, and any subsequent record
whose `ts` is within 60 seconds of that group's anchor (the first record's `ts` in the group)
joins the same group; anything more than 60 seconds past the anchor starts a new group, treated as
an independent round of sampling. Records within one group are merged and saved as a single data
point (`ts` is the group's anchor; if the same field number appears more than once within a group,
the last occurrence wins). **Anything more than 60 seconds apart is never merged**, even if it's
only a few seconds past the previous group's window — each round is its own independent data
point, and being in the same request doesn't mean records can be blindly merged into one.

For example, take this request (a common real-hardware pattern — one field per record — where two
sampling rounds are far apart in time):

```json
{
  "pid": "ubibot-ws1b",
  "sn": "RV41554WS1B",
  "ts": 1514767409,
  "payloads": [
    { "ts": 1514767395, "field1": 29.291221618652344 },
    { "ts": 1514767395, "field2": 40.831615447998047 },
    { "ts": 1514767395, "field3": 439.95001220703125 },
    { "ts": 1514767395, "field4": 0.014166667126119137 },
    { "ts": 1514767395, "field6": 26.9375 },
    { "ts": 1514767395, "field7": 27.625 },
    { "ts": 1514767409, "field5": -56 }
  ]
}
```

The largest `ts` gap across these 7 records is 1514767409 − 1514767395 = 14 seconds, within the
60-second window, so they're all merged and saved as **1 data point** (`ts=1514767395`, containing
7 fields: field1–field4, field6, field7, field5).

Now compare that to two sampling rounds spaced further apart (the second one 2 minutes after the
first):

```json
{
  "pid": "ubibot-ws1b",
  "sn": "RV41554WS1B",
  "ts": 1514767515,
  "payloads": [
    { "ts": 1514767395, "field1": 29.29, "field2": 40.83 },
    { "ts": 1514767515, "field1": 29.31, "field2": 40.79 }
  ]
}
```

The two `ts` values are 120 seconds apart, past the 60-second merge window, so they're saved as
**2 separate data points** (`ts=1514767395` and `ts=1514767515` each on their own) rather than
merged into one.

Response:

```json
{ "c": 0, "t": 1788950400 }
```

| Field | Description |
| --- | --- |
| c | Business status code — 0 for success, see §8 for other values |
| t | Server's current Unix timestamp (seconds); the device may use it to calibrate its local clock (optional, not required) |

Server behavior:
- If `sn` has never been seen before, a device record is created automatically (pid/sn/first-seen
  time, etc.); afterward it's treated as an existing device.
- If a device has been manually disabled by an admin, this endpoint rejects all of that device's
  data (see §8 — disabling is an admin action on an existing device, not a precondition for a new
  device to be onboarded).

### 6. Data Fields (field1 – field20)

Named sensor fields like `temperature`/`humidity` are no longer used — everything is unified into
numbered fields, up to 20. The platform stores the numeric value under each number as-is and
doesn't care what physical quantity each number represents, except for these 3 fields which carry
a conventional default meaning:

| Field | Default meaning |
| --- | --- |
| field1 | Temperature |
| field2 | Humidity |
| field3 | Light level |
| field4 – field20 | Defined entirely by the user (e.g. CO2, soil pH, battery voltage, etc.) — the platform makes no assumptions |

This default meaning is only a convention used for admin-console display (e.g. icon, unit) — it
is not enforced at the protocol level. A device is free to report only `field1`, or start from
`field4` onward; the platform saves whatever it's given either way.

### 7. Device Management (admin-console side, not part of the device-facing protocol)

The following is not something a device needs to implement — it's a description of admin-console
behavior, to help make sense of the "auto-create/disable" behavior mentioned in §5:

- A device automatically appears in the admin console's device list the first time it
  successfully reports data — no need to create it beforehand.
- An admin can rename a device, view its historical data, enable/disable it, or delete it along
  with all of its data from the console.
- Once a device is disabled, all of its subsequent report requests are rejected (data is no
  longer processed); re-enabling it restores normal behavior.
- Deleting a device permanently deletes it along with all of its historical data.

### 8. Error Handling

| HTTP status | c | Scenario | Device behavior |
| --- | --- | --- | --- |
| 200 | 0 | Success | Process normally |
| 400 | 1002 | Timestamp outside the ±5-minute window | Check the local clock, or call the §4 time-sync endpoint to calibrate and retry |
| 400 | 1003 | Malformed request body | Check the firmware's serialization logic; retry next cycle |
| 401 | 1103 | Device has been disabled by an admin | Stop retrying and raise an alert (LED/log); needs manual review |
| 429 | 1900 | Rate limited | Drop this report; wait for the next cycle |
| 5xx | 5000 | Server-side failure | Retry on the next upload cycle |

### 9. Command Delivery (admin-triggered, optional)

This is the one exception to §0/§2's "no channel from the server to the device" stance, and it's
deliberately narrow: an admin can queue **at most one** command per device from the admin console,
and it is delivered piggybacked on that device's **next `POST /api/v1/data/report` response** —
no new endpoint, no device-initiated polling, no push. A device that never looks for this field
is completely unaffected; everything about §2–§8 is unchanged.

**Response with a queued command** (§5's normal response, with an added `cmd` field):

```json
{ "c": 0, "t": 1788950400, "cmd": { "action": "reboot" } }
```

```json
{ "c": 0, "t": 1788950400, "cmd": { "action": "set_interval", "seconds": 600 } }
```

| Field | Description |
| --- | --- |
| cmd | Present only when a command is queued for this device — omitted entirely (not `null`) otherwise |
| cmd.action | `reboot` or `set_interval` — see below |
| cmd.seconds | Only present for `set_interval`: the new report interval, in seconds |

| action | Device behavior |
| --- | --- |
| `reboot` | Restart immediately |
| `set_interval` | Persist the new interval (e.g. to NVS) and apply it starting with the *next* sleep/report cycle — the one already under way finishes with whatever interval was active when it started |

**Delivery semantics — read this before building on it**:
- **At most one command queued per device.** Queuing a new one overwrites whatever hadn't been
  delivered yet; there is no queue/backlog of multiple commands.
- **Delivered once, then cleared — fire-and-forget.** The moment a command is embedded in a
  report response, the platform considers it delivered and clears it, whether or not the device
  actually receives or applies it (a dropped response, a device that crashes before acting on it,
  etc. all look the same to the platform: "delivered"). There is no ack, retry, or delivery
  confirmation of any kind.
- **No bound on how long a command can sit queued.** It's delivered on whatever the device's next
  report happens to be — immediately if it reports every few minutes, or not for a long time if
  it reports rarely or is offline.
- `seconds` bounds (how small/large a `set_interval` value is accepted) are an **admin-API-level**
  concern, not part of this wire format — a device should trust whatever value the field carries.

The response format and effective-timing behavior above reflect the `ubibot-open-ws1b` reference
firmware's implementation (see
[main/command.c](https://github.com/ubibot-open/ubibot-ws1b/blob/main/main/command.c) in that
repository). Other firmware implementations are encouraged to follow the same convention for
consistency, but it isn't mandatory.

### Appendix: Capabilities Removed in This Revision

To match the scope adjustments above, the following capabilities that existed in earlier versions
of this protocol have been removed entirely and are no longer part of this project's scope:
- Device identity authentication (HMAC signature, the DeviceSecret derivation formula) —
  targeting a trusted internal-network environment, this has been simplified further down to
  plain-text pid+sn device identification, with no signature/key check at all.
- Session tokens and their renewal.
- The original, much larger channel for the server to push control commands to a device,
  including config polling (formerly `/device/poll`), custom probe read configuration (formerly
  `set_probe`), and firmware OTA updates. **§9 later reintroduced a deliberately narrow `cmd`
  field** (reboot / set report interval only, piggybacked on the report response, no ack) — the
  rest of the original channel remains out of scope.
- The self-service device activation approval / key-binding flow (including encrypted submission
  and RSA key pairs).
