# Admin API Reference

*[中文](admin-api.zh-CN.md)*

This is the API the bundled admin console itself talks to — every route below lives under
`/api/admin/*` on the same address the console's UI is served from (`http://<host>:8080` by
default). Useful if you're building an alternative frontend, scripting a bulk operation, or just
want to know what a button in the console actually calls. If you only need read-only access to
devices/telemetry from outside, the [Open API](open-api.md) (a much smaller, API-key-authenticated
surface) is very likely a better fit — don't reach for admin credentials just to read data.

## Conventions

**Auth**: `POST /api/admin/login` returns a bearer token; every other route requires
`Authorization: Bearer <token>`. A missing header is `401` `bearer_token_missing`; an
expired/invalid token is `401` `session_invalid_or_expired`.

**Permissions**: most routes additionally require one of four permission codes, checked against
the caller's role (`super_admin`'s role bypasses every check):

| Permission | Guards |
|---|---|
| `device:read` | Reading devices, records, field settings, alert rules/events, icons |
| `device:write` | Renaming/enabling/disabling/deleting a device, sending/cancelling a command, editing field settings |
| `alert:manage` | Creating/deleting alert rules, resolving alert events |
| `system:manage` | Users, roles, audit log, API keys, files, dictionaries, params, icon uploads, products, system metrics |

A handful of routes (login itself, `me`, notifications, dashboard, dictionary listing) only
require *being* logged in (`RequireAdmin`) with no specific permission code — noted per-route
below. Failing a permission check is `403` `forbidden`.

**Response envelope**: every response, success or error, is JSON with a `timestamp` (Unix
seconds) field injected automatically — omitted from the examples below to avoid repeating it 60
times; assume it's always there.

**Errors**: `{"code": "<stable_code>", "message": "<English text>", "timestamp": ...}`. `code` is
safe to match on for localizing an error message client-side; `message` is always English and not
meant to be shown as-is. Common codes: `invalid_id`, `invalid_request_body`, `forbidden`,
`internal_error`, `device_not_found`, `file_not_found`. Most `400`s have a specific code named
after the missing/invalid field (e.g. `name_required`, `invalid_status`); anything not in the
lookup table falls back to the generic `error`.

**Pagination**: `page` (default 1) and `page_size` (default 20) query params, wherever a route
returns a `"list"` + `"total"` — no fixed upper bound on `page_size` is enforced by these params
themselves, though some list queries also cap internally (e.g. device listing tops out at 200 per
page regardless of what's requested).

**Audit log**: every mutating route below writes an entry (actor, action, target type/id, a short
detail string, caller IP) to the audit log (`GET /api/admin/audit-logs`) — noted once here rather
than per-route.

---

## Auth

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `POST /api/admin/login` | — | `{"username","password"}` | `{"token","expires_in","username"}` — `401 invalid_credentials` on failure |
| `GET /api/admin/me` | logged in | — | `{"username"}` |

---

## Devices

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `GET /api/admin/devices` | `device:read` | `page`, `page_size` | `{"list":[deviceDTO...],"total"}` |
| `GET /api/admin/devices/data-warehouse` | `device:read` | `page`, `page_size` | `{"list":[deviceDTO + last_record + field_meta],"total"}` — each row's latest telemetry inlined, field names/units/icons pre-resolved so the frontend doesn't need N extra requests |
| `GET /api/admin/devices/{id}` | `device:read` | path `id` | `{"device":deviceDTO,"records":[recordDTO...]}` (last 20 records) |
| `GET /api/admin/devices/{id}/records` | `device:read` | path `id`; query `start`,`end` (unix, both optional/open-ended), `page`, `page_size` | `{"list":[recordDTO...],"total"}` |
| `PATCH /api/admin/devices/{id}` | `device:write` | `{"name"}` (empty allowed — clears back to showing the SN) | `deviceDTO` (bare) |
| `POST /api/admin/devices/{id}/status` | `device:write` | `{"status"}` (`1`=enabled, `2`=disabled) | `{"message":"ok"}` |
| `DELETE /api/admin/devices/{id}` | `device:write` | — | `{"message":"ok"}` — **irreversible**, deletes the device plus every telemetry record and alert rule/event referencing it |

**`deviceDTO`**:
```json
{
  "id": 1, "pid": "ubibot_open_dev_v1", "sn": "sn_ws1_20001_1", "name": "Warehouse Sensor 1",
  "status": 1, "online": true, "last_seen_at": 1788950400, "created_at": 1788900000,
  "pending_command": { "action": "reboot" },
  "product_name": "WS1B Sensor"
}
```
`pending_command` and `product_name` are omitted entirely (not `null`) when there's nothing
queued / no matching Product — see [Commands](#commands) and [Products](#products) below.

**`recordDTO`**: `{"ts": 1788950400, "d": {"field1": 25.6, "field2": 60.2}}` — `d` mirrors
whatever `field1`..`field20` keys the device actually reported (protocol §5/§6), no fixed schema.

A device is never created directly through this API — it appears the moment it successfully
reports (protocol §5/§7), or via [batch import](#batch-device-management) ahead of time.

---

## Batch device management

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `POST /api/admin/devices/import` | `device:write` | `{"rows":[{"sn","pid","name"}...]}` (up to 5000 rows; `name` optional) | `{"created":int,"skipped":["sn",...],"failed":["row 3: ...",...]}` |
| `GET /api/admin/devices/export.csv` | `device:read` | — | `text/csv` body (not JSON), all devices, no pagination |

Import is per-row, not all-or-nothing: a duplicate `sn` lands in `skipped` (not an error), a row
missing a required field lands in `failed` with a reason, and the rest of the batch still goes
through. A pre-imported device behaves exactly like an auto-created one once it actually reports
— matched by `sn`. Export columns: `id,pid,product_name,sn,name,status,online,last_seen_at,created_at`.

---

## Products

Display metadata for a device type/model (a name/description for a `pid`) — resolved onto
devices by matching `pid` at read time, never a foreign key. Deleting or renaming a Product never
touches any device row.

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `GET /api/admin/products` | `device:read` | — | `{"list":[productDTO...]}` |
| `POST /api/admin/products` | `system:manage` | `{"pid","name","description"}` (`pid`/`name` required; `pid` must be unique) | `productDTO` (bare) |
| `PATCH /api/admin/products/{id}` | `system:manage` | `{"name","description"}` — `pid` is **immutable** after creation | `{"message":"ok"}` |
| `DELETE /api/admin/products/{id}` | `system:manage` | — | `{"message":"ok"}` — removes only the display metadata |

`productDTO`: `{"id","pid","name","description","created_at"}`.

---

## Commands

Queues a command delivered piggybacked on a device's *next* report response (protocol §9) —
fire-and-forget, no ack, at most one command queued per device at a time.

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `POST /api/admin/devices/{id}/commands` | `device:write` | `{"action":"reboot"}` or `{"action":"set_interval","seconds":N}` (`N` must be 60–86400) | `{"message":"queued","cmd":{...}}` |
| `DELETE /api/admin/devices/{id}/commands` | `device:write` | — | `{"message":"ok"}` — cancels a not-yet-delivered command; a no-op if nothing was queued |

See the [protocol doc §9](../protocol/hardware-communication-protocol.md#9-command-delivery-admin-triggered-optional)
for exactly how/when the device receives this.

---

## Field settings

Per-device overrides for how a `field1`..`field20` key displays (name/unit/icon) — falls back to
the global icon-template library ([Icon library](#icon-library)) when not overridden.

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `GET /api/admin/devices/{id}/field-settings` | `device:read` | path `id` | `{"list":[fieldSettingDTO x20]}` — always all of field1..field20, each resolved through the override→template→empty chain |
| `POST /api/admin/devices/{id}/field-settings/{key}` | `device:write` | path `id`,`key`; `{"name","unit","svg"}` (`svg` ≤64KB, must contain `<svg` if non-empty) | `fieldSettingDTO` (bare, post-write) |
| `DELETE /api/admin/devices/{id}/field-settings/{key}` | `device:write` | path `id`,`key` | `{"message":"ok"}` — clears the override, falling back to the template |

`fieldSettingDTO`: `{"key","name","unit","svg","is_custom"}`.

---

## Alerts

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `GET /api/admin/devices/{id}/alert-rules` | `device:read` | path `id` | `{"list":[alertRuleDTO...]}` |
| `POST /api/admin/devices/{id}/alert-rules` | `alert:manage` | `{"field","op","threshold"}` — `op` one of `> >= < <= ==` | `alertRuleDTO` (bare) |
| `DELETE /api/admin/alert-rules/{id}` | `alert:manage` | — | `{"message":"ok"}` |
| `GET /api/admin/alert-events` | `device:read` | `page`,`page_size`,`device_id`,`status` (all optional filters) | `{"list":[alertEventDTO...],"total"}` |
| `POST /api/admin/alert-events/{id}/resolve` | `alert:manage` | — | `{"message":"ok"}` |

`alertRuleDTO`: `{"id","device_id","field","op","threshold","enabled"}`.
`alertEventDTO`: `{"id","device_id","device_name","rule_id","type","message","status","triggered_at","resolved_at"}`
— `type` is `threshold` or `offline`, `status` is `open` or `resolved`. Offline alerts are raised
by a periodic sweep (no rule needed); threshold alerts need a rule.

---

## RBAC: roles, admin users, audit log

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `GET /api/admin/roles` | `system:manage` | — | `{"list":[roleDTO...]}` |
| `POST /api/admin/roles` | `system:manage` | `{"name","code","permissions":[...]}` (`name`/`code` required) | `roleDTO` (bare) |
| `PATCH /api/admin/roles/{id}` | `system:manage` | `{"name","permissions"}` — `code` immutable | `{"message":"ok"}` |
| `DELETE /api/admin/roles/{id}` | `system:manage` | — | `{"message":"ok"}` |
| `GET /api/admin/users` | `system:manage` | — | `{"list":[adminUserDTO...]}` |
| `POST /api/admin/users` | `system:manage` | `{"username","password","role_id"}` (all required) | `adminUserDTO` (bare) |
| `PATCH /api/admin/users/{id}` | `system:manage` | `{"role_id","password"}` — either/both, applied independently | `{"message":"ok"}` |
| `DELETE /api/admin/users/{id}` | `system:manage` | — | `{"message":"ok"}` — `400 cannot_delete_own_account` if you target yourself |
| `GET /api/admin/audit-logs` | `system:manage` | `page`,`page_size` | `{"list":[auditLogDTO...],"total"}` |

`roleDTO`: `{"id","name","code","permissions":[...]}` — `permissions` is `["*"]` for the built-in
`super_admin` role, or an explicit list of the four permission codes otherwise.
`adminUserDTO`: `{"id","username","role_id","role_name","created_at"}`.
`auditLogDTO`: `{"id","username","action","target_type","target_id","detail","ip","created_at"}`.

---

## Notifications

In-app only today — no email/SMS/webhook channel. The console polls this rather than anything
push-based.

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `GET /api/admin/notifications` | logged in | `page`,`page_size` | `{"list":[notificationDTO...],"total","unread"}` |
| `POST /api/admin/notifications/{id}/read` | logged in | — | `{"message":"ok"}` |
| `POST /api/admin/notifications/read-all` | logged in | — | `{"message":"ok"}` |

`notificationDTO`: `{"id","type","level","title","content","status","created_at"}` — `level` is
`info`/`warning`/`critical`, `status` is `unread`/`read`.

---

## API keys (Open API management)

Managing the keys that authenticate against the [Open API](open-api.md) — see that doc for how a
key is actually used.

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `GET /api/admin/api-keys` | `system:manage` | — | `{"list":[apiKeyDTO...]}` |
| `POST /api/admin/api-keys` | `system:manage` | `{"name"}` | `{"key":apiKeyDTO,"raw_key":"..."}` |
| `POST /api/admin/api-keys/{id}/revoke` | `system:manage` | — | `{"message":"ok"}` |

`apiKeyDTO`: `{"id","name","prefix","revoked","last_used_at","created_at"}` — **never includes the
raw key**; `raw_key` on the create response is the only time it's ever shown, only the hash is
stored server-side. Losing it means revoking and creating a new one.

---

## Files, dictionaries, params

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `GET /api/admin/files` | `system:manage` | — | `{"list":[fileAssetDTO...]}` |
| `POST /api/admin/files` | `system:manage` | `multipart/form-data`: field `category` (default `"other"`), file field `file` (required, ≤32MB) | `fileAssetDTO` (bare) |
| `DELETE /api/admin/files/{id}` | `system:manage` | — | `{"message":"ok"}` |
| `GET /api/admin/dict` | logged in (no permission code) | query `type` (optional filter) | `{"list":[dictEntryDTO...]}` |
| `POST /api/admin/dict` | `system:manage` | `{"type","key","label","sort"}` | `dictEntryDTO` (bare) |
| `PATCH /api/admin/dict/{id}` | `system:manage` | `{"label","sort"}` — `type`/`key` immutable | `{"message":"ok"}` |
| `DELETE /api/admin/dict/{id}` | `system:manage` | — | `{"message":"ok"}` |
| `GET /api/admin/params` | `system:manage` | — | `{"list":[systemParamDTO...]}` |
| `PATCH /api/admin/params/{key}` | `system:manage` | `{"value","description"}` | `systemParamDTO` (bare) |

`fileAssetDTO`: `{"id","category","filename","size","sha256","created_at"}` — uploaded files are
stored on disk under the server's file directory, SHA-256-hashed on the way in.
`dictEntryDTO`: `{"id","type","key","label","sort"}` — a generic key/label lookup table for
dropdown options; `type` groups entries (e.g. a `command_type` dictionary).

**System params are an open-ended key/value store** — `PATCH` accepts any `key`, but only two are
ever read back into live behavior:

| Key | Effect | Default |
|---|---|---|
| `rate_limit_per_minute` | Device-facing endpoints' per-IP rate limit, applied live (no restart) | `120` |
| `offline_grace_minutes` | How long a device can go quiet before it's marked offline, applied live | `2` |

---

## Icon library

Global default field templates (name/unit/icon per `field1`..`field20` *key name*, not per
device) — the fallback a device's own [field settings](#field-settings) resolve through when it
has no override of its own.

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `GET /api/admin/icons` | `device:read` | — | `{"list":[iconDTO...]}` |
| `POST /api/admin/icons` | `system:manage` | `{"key","name","unit","svg"}` (`key`/`name` required; `svg` ≤64KB, must contain `<svg` if non-empty) — upsert, re-uploading a `key` replaces it | `iconDTO` (bare) |
| `DELETE /api/admin/icons/{key}` | `system:manage` | path `key` | `{"message":"ok"}` |

`iconDTO`: `{"key","name","unit","svg","created_at"}`. SVG travels as raw markup inline in the
JSON body (not multipart) — small enough that it wasn't worth a file upload round trip.

---

## System monitor & dashboard

| Method & Path | Perm | Request | Response |
|---|---|---|---|
| `GET /api/admin/system/metrics` | `system:manage` | — | `{"go_version","goroutines","heap_alloc_bytes","uptime_seconds","db_size_bytes","device_total","open_alerts","unread_notifications"}` |
| `GET /api/admin/dashboard/summary` | logged in | — | `{"device_total","device_online","open_alerts","today_records"}` |
| `GET /api/admin/dashboard/trends` | logged in | — | `{"days":[{"day":"2026-01-15","count":42}, ...]}` (last 7 days) |

---

## What's not here

No OTA, no MQTT/CoAP, no multi-tenant/customer accounts, no email/SMS/webhook alert channels, no
CAPTCHA/lockout on login — see the
[protocol doc's scope section](../protocol/hardware-communication-protocol.md#0-project-scope--rationale-for-this-revision)
and the [architecture overview](../architecture/overview.md#design-principles-this-codebase-leans-on)
for why: this is an internal/educational tool that prioritizes staying simple over covering every
feature a commercial platform would.
