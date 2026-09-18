# Add a New Data Model

*[中文](03-backend-add-model-and-migration.zh-CN.md)*

Use this chapter when your endpoint needs a table that doesn't exist yet — if you're just adding a
field to `Device` or another existing model, skip straight to step 2. If the endpoint itself is
what you're after, [Add a New Admin API Endpoint](02-backend-add-admin-endpoint.md) covers that
layer; this chapter is about the model underneath it.

## 1. Define the struct

Every model lives in `server/internal/model/model.go` as a plain Go struct with GORM tags. There's
no separate migration DSL or `.sql` files — the struct tags *are* the schema. `Product` is a
recent, minimal example:

```go
// server/internal/model/model.go
type Product struct {
	ID uint `gorm:"primaryKey"`
	// column:pid pins the actual column name explicitly -- GORM's default
	// naming strategy maps the Go field PID to p_id (it treats "PID" as
	// "P"+"ID", the same way it already does for Device.PID), which would
	// otherwise silently mismatch every raw "pid" reference in this
	// package's queries (see store.ProductsByPIDs).
	PID         string `gorm:"column:pid;size:64;not null;uniqueIndex"`
	Name        string `gorm:"size:128;not null"`
	Description string `gorm:"size:512"`

	CreatedAt time.Time
	UpdatedAt time.Time
}

func (Product) TableName() string { return "products" }
```

Conventions to follow, all visible above and consistent with every other model in the file:

- `ID uint \`gorm:"primaryKey"\`` first.
- `size:N` on every string field — this isn't cosmetic, GORM/the sqlite driver need it to pick a
  column type.
- `not null` on anything that's actually required; `uniqueIndex` on anything that must be unique
  (GORM creates the index for you at migrate time).
- `CreatedAt time.Time` / `UpdatedAt time.Time` (GORM manages these automatically on
  create/save — you never set them yourself) if the row's age or last-modified time matters.
  `Device.PendingCmd string \`gorm:"type:text"\`` shows the pattern for a field holding an
  arbitrarily-long string (a raw JSON blob, in that case) rather than a short bounded one.
- An explicit `TableName()` method — GORM will pluralize the struct name into a table name on its
  own, but every model here defines this explicitly rather than relying on that inference.

## 2. Register it with AutoMigrate

There's exactly one place a new model needs to be added — `server/internal/store/db.go`'s
`Open()` function:

```go
if err := db.AutoMigrate(
	&model.Device{},
	&model.DeviceRecord{},
	// ...
	&model.DeviceFieldSetting{},
	&model.Product{},
); err != nil {
	return nil, fmt.Errorf("migrate: %w", err)
}
```

`AutoMigrate` creates the table (and adds new columns to an existing one) on every startup if it's
not already there — there's no separate "run migrations" step, and no down-migrations either. This
is deliberately simple (see the [Architecture Overview](../architecture/overview.md)'s "minimal by
default" principle) and it's also why every column addition needs to be additive: `AutoMigrate`
will add a new column with its zero value for existing rows, but it won't rename or drop one for
you, and it won't backfill a `not null` column that used to allow nulls.

## 3. Watch for the `PID` → `p_id` naming pitfall

This is a real bug that was actually hit and fixed in this codebase, not a hypothetical: GORM's
default naming strategy converts a Go field name to a column name by inserting underscores before
capital letters — and it treats a field named `PID` as the two words "P" + "ID", producing the
column name `p_id`, not `pid`. Any code that queries by the literal string `"pid"` — like
`store.ProductsByPIDs`'s `Where("pid IN ?", pids)` — would silently fail (`no such column: pid`)
against a `PID` field without the explicit tag shown in step 1:

```go
PID string `gorm:"column:pid;size:64;not null;uniqueIndex"`
```

**`Device.PID` has this exact same latent issue and does not have the tag** — it's never been hit
because no code in this repo currently does a raw `Where("pid = ?", ...)` against the `devices`
table (every existing lookup goes through `SN`, or through Go-side matching like `toDeviceDTO`'s
product resolution). If you ever add a query that filters `Device` by `pid` directly, add the same
`gorm:"column:pid"` tag first — don't rediscover this the hard way.

The general lesson: any all-caps or acronym-heavy field name (`PID`, `SN`, `ID` combined with
something else, `URL`, `API`, ...) is worth double-checking against GORM's actual column name
before writing a raw string-based query against it. When in doubt, write a throwaway test that
actually runs the query against a real in-memory SQLite database (see the next chapter's test
patterns) rather than assuming the column name — that's how this bug was originally caught.

## Checklist

1. Add the struct to `model.go` following the conventions above (tags, `CreatedAt`/`UpdatedAt` if
   relevant, explicit `TableName()`).
2. Add it to the `AutoMigrate(...)` call in `store/db.go`.
3. If any field has an all-caps/acronym name and you'll ever query it by raw column name, tag it
   explicitly (`gorm:"column:..."`)  — don't rely on the default naming strategy's guess.
4. Continue with [Add a New Admin API Endpoint](02-backend-add-admin-endpoint.md) for the store
   methods, handler, and route that actually expose this model.

## Next

[Add a New Admin Console Page](04-admin-add-page.md).
