# Changing the Protocol

*[中文](09-protocol-changes.zh-CN.md)*

"The protocol" means the device↔backend HTTP wire format documented in
[protocol/hardware-communication-protocol.md](../protocol/hardware-communication-protocol.md) §1-9
— request/response JSON shapes, error codes, the serial provisioning commands, and the §9 command
channel. Changing it is different from every other chapter in this handbook: it's the one place
where a firmware repo and the backend repo have to actually agree with each other, not just each
build and test cleanly on their own.

## There is no version field, on purpose

Check the wire formats yourself (`server/internal/protocol/protocol.go`, or the JSON examples
throughout the protocol doc) — there's no version number, no capability negotiation, nothing. §0's
stated rationale is to strip out anything not strictly necessary for "get data flowing with as
little configuration as possible," and a versioning scheme is exactly the kind of machinery that
gets you nothing on that path. The consequence is entirely on you as the person changing the
protocol: **the backend and every device's firmware are assumed to agree on the wire format at all
times**, with nothing in the protocol itself to detect or handle a mismatch.

## Additive vs. breaking — this distinction is the whole chapter

**Additive** (safe to roll out in either order — backend first or firmware first — because an
older implementation on either side simply ignores what it doesn't recognize):

- A new optional field on an existing request/response, following §9's `cmd` field as the
  model: *present only when relevant, omitted entirely (not sent as `null`) otherwise* — a device
  that has never heard of `cmd` just never looks for the key, and cJSON parsing an object that
  has an extra field it doesn't ask for is a no-op.
- A new `action` for the §9 command channel (see [chapter 7](07-firmware-add-server-command.md)) —
  old firmware hits the `else` branch of its `if`/`else if` chain and logs "unrecognized
  cmd.action", nothing worse.
- A new error `c` code (§8) — firmware that doesn't specifically handle it should already have a
  generic fallback branch (retry next cycle) rather than an exhaustive switch that breaks on an
  unknown value; if it doesn't, that's a firmware bug worth fixing regardless.
- A new serial provisioning command (§1.2, see [chapter 6](06-firmware-add-serial-command.md)) —
  nothing about existing commands changes.

**Breaking** (requires the backend and *every device's firmware* to update together — there is no
safe order, and no way to run a mixed fleet mid-rollout):

- Renaming or removing a field an existing firmware relies on.
- Changing an existing field's type or meaning (e.g. `field3` stops meaning "light" for some
  device class).
- Changing an error code's meaning, or removing a device behavior the protocol currently mandates
  (e.g. the ±5-minute timestamp window in §8/§4).

If you're not sure which category a change falls into, ask: "does an unmodified copy of
`ubibot-open-ws1b`'s current firmware, talking to the new backend, keep working exactly as
before?" If yes, it's additive. If the answer is "no, or I'm not sure," treat it as breaking and
plan the rollout accordingly — for this project, that almost always means: update the firmware
first, get it out to devices, *then* ship the backend change, since a device that's slow to
receive a firmware update is far more common than a backend that's slow to deploy.

## The Appendix is a real changelog, not decoration

The protocol doc's own **Appendix: Capabilities Removed in This Revision** exists because this
protocol used to be considerably larger (device-secret signing, session tokens, config polling,
OTA, an activation/key-binding flow) and was deliberately cut down to the current minimal shape
(§0). Notice its own callout: *"§9 later reintroduced a deliberately narrow `cmd` field... the
rest of the original channel remains out of scope."* If you ever remove a capability, add it to
this list with the same honesty — and if you're ever tempted to add back something on this list,
treat that as a real design decision worth raising explicitly (see [Overview](00-overview.md)'s
note on scope), not just a normal feature addition.

## Where to actually make the edit

1. The relevant `§N` section in `protocol/hardware-communication-protocol.md` — request/response
   shape, a field table, device behavior.
2. The Appendix, if you removed something.
3. Both reference implementation links the doc cites for the affected section (e.g. §1.2 points at
   `ubibot-open-ws1b`'s `provisioning.c`, §9 at `command.c`) — these aren't required to match
   (the doc says "encouraged... but isn't mandatory"), but they're the ones actually exercised by
   this project's own tests, so keep them honest.
4. [api/admin-api.md](../api/admin-api.md), if the change also touches an admin-triggered surface
   (like §9's commands do).

## Next

[Contributing and PR Conventions](10-contributing-and-pr-conventions.md).
