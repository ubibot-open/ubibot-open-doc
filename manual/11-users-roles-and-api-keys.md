# Users, Roles, and API Keys

*[中文](11-users-roles-and-api-keys.zh-CN.md)*

Three related pages under **System**: who can log in (**Admin**), what they're allowed to do
(**Role**), and who outside the console can read data (**API Key**).

## Roles

Go to **System → Role** and click **New Role**. A role is:

- **Name** — a display label.
- **Code** — a short identifier (e.g. `operator`). **Immutable once created** — get it wrong and
  you'll need to delete and recreate the role rather than rename its code.
- **Permissions** — a checkbox group over the platform's four permission codes:
  `device:read`, `device:write`, `alert:manage`, `system:manage`. Grant only what a role actually
  needs — there's no implicit "read includes write" or any hierarchy between them.

**Screenshot placeholder:** `11-users-roles-and-api-keys-01.png` — the New Role form with the
permission checkboxes visible.

> **Note:** the platform seeds one built-in super-admin role on first run (see
> [chapter 2](02-deploy-server.md)) that always passes every permission check regardless of what's
> stored for it — a fixed escape hatch so a mistake in the roles you create can never lock every
> admin out at once.

## Admin accounts

Go to **System → Admin** and click **New Admin**. Set a **username**, **password**, and assign a
**role**. Editing an existing admin lets you reassign their role and/or reset their password
(leave the password field blank to leave it unchanged) independently.

**Screenshot placeholder:** `11-users-roles-and-api-keys-02.png` — the admin list with the
edit form open.

You can't delete the account you're currently logged in as — that would sign you out with no
obvious way back in short of another admin stepping in.

## Open API keys

Go to **System → API Key** and click **New Key** — the only field is a **name** to remind you
what it's for later. Submitting it shows the raw key **exactly once**, in a copyable text box:

**Screenshot placeholder:** `11-users-roles-and-api-keys-03.png` — the one-time key reveal, with
the copy button visible.

Copy it now — the server only ever stores its hash, so there is no way to retrieve it again after
closing this dialog. If you lose it, revoke the key from the list and create a new one. A key has
no role or permission scope of its own — it's either valid or revoked, with access to exactly the
read-only endpoints documented in the [Open API Reference](../api/open-api.md).

## Next

[System Settings and Monitor](12-system-settings-and-monitor.md).
