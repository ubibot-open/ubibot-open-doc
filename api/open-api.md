# Open API Reference

*[中文](open-api.zh-CN.md)*

A small, read-only, API-key-authenticated surface for third-party integrations — the "RESTful
API for secondary development" the org profile advertises. If you're building the bundled admin
console's own alternative, or automating something an admin could otherwise click through, you
probably want the [Admin API Reference](admin-api.md) instead; this one only exposes what a
read-only external client should ever need: the device list and their telemetry history.

## Getting an API key

1. Log in to the admin console.
2. Go to System → Open API, and create a key (a name is all that's required).
3. The response includes the raw key **exactly once** — copy it immediately. The server only
   ever stores its hash; there is no way to retrieve it again later. If you lose it, revoke it
   and create a new one.

## Authentication

Every request must carry the key in an `X-Api-Key` header — not `Authorization`, which is
reserved for admin console sessions:

```bash
curl -H "X-Api-Key: <your key>" http://localhost:8080/api/open/v1/devices
```

| Situation | Response |
|---|---|
| Header missing | `401` `{"code":"api_key_missing","message":"missing api key", ...}` |
| Key invalid or revoked | `401` `{"code":"api_key_invalid_or_revoked","message":"invalid or revoked api key", ...}` |

A key has no notion of role or permission scope beyond "valid" — unlike admin sessions, there's
no RBAC layered on top of it. Revoke a key from the same System → Open API page at any time; a
revoked key starts failing immediately.

## Response envelope

Every response (success or error) is JSON with a `timestamp` (Unix seconds) field injected
automatically alongside whatever fields are documented below — not repeated per endpoint below to
avoid restating it 20 times.

## Endpoints

### `GET /api/open/v1/devices`

Lists devices — a deliberately smaller shape than the admin console sees (no `pid`, `status`,
`pending_command`, or `product_name`; just enough to identify a device and know if it's reporting).

**Query parameters**

| Param | Default | Description |
|---|---|---|
| `page` | 1 | Page number |
| `page_size` | 20 | Rows per page |

**Response**

```json
{
  "list": [
    { "id": 1, "sn": "sn_ws1_20001_1", "name": "Warehouse Sensor 1", "online": true, "last_seen_at": 1788950400 }
  ],
  "total": 1
}
```

```bash
curl -H "X-Api-Key: <your key>" "http://localhost:8080/api/open/v1/devices?page=1&page_size=50"
```

### `GET /api/open/v1/devices/{id}/records`

Historical telemetry for one device, by its numeric `id` (from the list above).

**Query parameters**

| Param | Default | Description |
|---|---|---|
| `start` | 0 (open) | Unix seconds, inclusive lower bound on `ts` |
| `end` | 0 (open) | Unix seconds, inclusive upper bound on `ts` |
| `page` | 1 | Page number |
| `page_size` | 20 | Rows per page |

**Response**

```json
{
  "list": [
    { "ts": 1788950400, "d": { "field1": 25.6, "field2": 60.2 } }
  ],
  "total": 1
}
```

`d` is whatever `field1`..`field20` keys that record happened to carry (protocol §6) — no fixed
schema, mirrors exactly what the device reported.

```bash
curl -H "X-Api-Key: <your key>" \
  "http://localhost:8080/api/open/v1/devices/1/records?start=1788950000&end=1788960000"
```

## What's not here

No write endpoints (rename/disable/delete/command a device), no alerts, no products — those are
admin-console actions with real side effects, gated by RBAC rather than a flat API key; see the
[Admin API Reference](admin-api.md) if you need them and are comfortable driving the same API the
console itself uses.
