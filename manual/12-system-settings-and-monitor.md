# System Settings and Monitor

*[中文](12-system-settings-and-monitor.zh-CN.md)*

The rest of **System**: four small settings pages, plus a server health page. None of these are
part of getting a device online — they're supporting infrastructure, some more actively used than
others.

## Dict (dictionary entries)

**System → Dict** manages generic `type`/`key`/`label`/`sort` lookup entries — shared reference
data meant to be reused across the console instead of hardcoding a list of options in each page
that needs one.

**Screenshot placeholder:** `12-system-settings-and-monitor-01.png` — the Dict page with a few
entries.

> **Note, as of this writing:** no other page in the console actually reads dictionary entries
> yet — this page manages the data, but nothing currently populates a dropdown or list from it.
> It's infrastructure for future use, not (yet) wired into anything you'll notice elsewhere.

## Params (system parameters)

**System → Params** lists a fixed, small set of runtime-tunable values — you can edit an existing
one's value inline, but there's no "add a new parameter" button. Only two currently do anything
when changed:

| Key | Default | Effect |
|---|---|---|
| Rate limit per minute | 120 | Caps how many device-facing requests one IP can make per minute (protocol §8, error code 1900) |
| Offline grace period (minutes) | 2 | How long without a report before a device is considered offline |

Changing either takes effect **immediately** on the running server — no restart needed.

**Screenshot placeholder:** `12-system-settings-and-monitor-02.png` — the Params table with an
edit in progress.

## Files

**System → Files** is a generic upload registry — pick a category and a file (up to 32MB); it's
stored with its SHA-256 hash and shows up in the list. Delete removes it. Nothing in the console
currently references an uploaded file automatically (no "attach this file to X" flow) — it's a
place to park a file and get a stable reference/download link to it.

## Icons

**System → Icons** manages the default name/unit/icon shown for `field1`-`field20` when a device
hasn't customized one of its own field settings (see the per-device field settings mentioned in
[chapter 8](08-managing-devices-and-products.md)'s device detail page). The list shows every
built-in default field alongside any you've customized — click **New** to add a custom one for a
key that doesn't have one yet, or **Replace** on an existing row (its key is then locked — you're
replacing that key's icon, not renaming it).

**Screenshot placeholder:** `12-system-settings-and-monitor-03.png` — the icon library list,
showing a mix of built-in and custom entries.

Deleting a custom entry reverts that key back to the console's built-in default icon — it doesn't
affect any device that already has its *own* per-device override (field settings on a specific
device always take priority over this shared default).

## Monitor (system health)

**System → Monitor** — server-side health, not device data: Go runtime version, goroutine count,
memory use, process uptime, and the SQLite database file's size. Cheap to refresh often if you
want to keep an eye on it while testing something.

**Screenshot placeholder:** `12-system-settings-and-monitor-04.png` — the System Monitor page.

## Next

[Troubleshooting](13-troubleshooting.md).
