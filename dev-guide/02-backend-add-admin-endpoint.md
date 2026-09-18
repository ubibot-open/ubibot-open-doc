# Add a New Admin API Endpoint

*[中文](02-backend-add-admin-endpoint.zh-CN.md)*

Every `/api/admin/*` endpoint in this codebase is built from the same four layers: **store**
(the database query), **handler** (HTTP request/response, permission check, audit log),
**router** (wiring the handler to a method+path), and a **test** that exercises it through the
router like a real client would. Rather than invent a toy example, this chapter walks through how
the Product CRUD endpoints — `server/internal/store/product.go`,
`server/internal/api/product_handlers.go` — were actually built, since they're a complete,
already-merged example of exactly this shape. Use them as your template.

## 1. The store layer

The store owns every database query — handlers never touch `gorm.DB` directly. A store file is
just plain methods on `*Store`:

```go
// server/internal/store/product.go
func (s *Store) CreateProduct(pid, name, description string) (*model.Product, error) {
	p := &model.Product{PID: pid, Name: name, Description: description}
	if err := s.db.Create(p).Error; err != nil {
		return nil, err
	}
	return p, nil
}

func (s *Store) ListProducts() ([]model.Product, error) {
	var rows []model.Product
	err := s.db.Order("id desc").Find(&rows).Error
	return rows, err
}
```

If your endpoint reads or writes an existing model, you may not need a new store method at all —
check `internal/store/` for one that already does what you need before adding another. If you're
also introducing a brand-new model (like `Product` was), see
[Add a New Data Model](03-backend-add-model-and-migration.md) first.

## 2. The handler layer

A handler decodes the request, calls the store, and writes the response through this codebase's
two small helpers: `writeAPIJSON` (the envelope every response uses) and `decodeJSON`/`adminErr`
for request parsing and errors (all in `server/internal/api/httpx.go` and `middleware.go`). A DTO
struct controls exactly what shape crosses the wire — it's deliberately not the GORM model itself,
so a later change to the model's storage shape doesn't silently change the API:

```go
// server/internal/api/product_handlers.go
type productDTO struct {
	ID          uint   `json:"id"`
	PID         string `json:"pid"`
	Name        string `json:"name"`
	Description string `json:"description"`
	CreatedAt   int64  `json:"created_at"`
}

type createProductRequest struct {
	PID         string `json:"pid"`
	Name        string `json:"name"`
	Description string `json:"description"`
}

func (s *Server) CreateProduct(w http.ResponseWriter, r *http.Request) {
	var req createProductRequest
	if err := decodeJSON(r, &req); err != nil || req.PID == "" || req.Name == "" {
		adminErr(w, 400, "pid and name are required")
		return
	}
	p, err := s.Store.CreateProduct(req.PID, req.Name, req.Description)
	if err != nil {
		adminErr(w, 400, "a product for this pid already exists, or the request was otherwise invalid")
		return
	}
	s.audit(r, "product.create", "product", p.ID, req.PID)
	writeAPIJSON(w, 200, productDTO{ID: p.ID, PID: p.PID, Name: p.Name, Description: p.Description, CreatedAt: p.CreatedAt.Unix()})
}
```

Two things every mutating handler does, visible above:

- **`s.audit(r, action, targetType, targetID, detail)`** — records who did what, for the audit log
  (`GET /api/admin/audit-logs`). Call it after the mutation succeeds, before writing the response.
- **Validate before calling the store**, and return `400` with a message explaining what's
  missing/invalid — the store layer isn't expected to produce user-facing error text.

## 3. The router layer

One line in `server/internal/api/router.go` per route, wrapped in the permission check it needs:

```go
mux.HandleFunc("GET /api/admin/products", s.RequirePermission(model.PermDeviceRead, s.ListProducts))
mux.HandleFunc("POST /api/admin/products", s.RequirePermission(model.PermSystemManage, s.CreateProduct))
mux.HandleFunc("PATCH /api/admin/products/{id}", s.RequirePermission(model.PermSystemManage, s.UpdateProduct))
mux.HandleFunc("DELETE /api/admin/products/{id}", s.RequirePermission(model.PermSystemManage, s.DeleteProduct))
```

`RequirePermission(code, handler)` (in `middleware.go`) checks the caller's bearer token and their
role's permissions before ever calling your handler — there are exactly four permission codes to
choose from (`model.PermDeviceRead`, `PermDeviceWrite`, `PermAlertManage`, `PermSystemManage`; see
`server/internal/model/model.go`). Pick based on what the action actually does, following the
reasoning already commented next to each route group in `router.go` — e.g. Product's read rides on
`device:read` because it's resolved display data for a device, while its write rides on
`system:manage` because it's shared reference data, not a mutation of one specific device. If an
endpoint should be reachable without an admin session at all (rare — most things shouldn't be),
use `RequireAdmin` instead of `RequirePermission` for "logged in, any role" or, for the read-only
third-party surface, `RequireApiKey`.

Also update `api/admin-api.md` in `ubibot-open-doc` with the new route — that reference is meant
to stay a complete, accurate list of every route this server exposes.

## 4. The test

Tests live alongside the handlers (`product_handlers_test.go`) and drive the router the same way a
real HTTP client would — `env.do(t, method, path, body, authHeaders)` — rather than calling the
handler function directly, so the test also exercises routing and the permission check:

```go
// server/internal/api/product_handlers_test.go
func TestProductCRUDAndDeviceResolution(t *testing.T) {
	env := newTestEnv(t)
	adminAuth := env.createSuperAdmin(t, "admin", "s3cret-pw")

	rec, body := env.do(t, "POST", "/api/admin/products",
		map[string]any{"pid": testPID, "name": "WS1B Sensor", "description": "Outdoor temp/humidity/light"}, adminAuth)
	if rec.Code != 200 {
		t.Fatalf("create product failed: %d %v", rec.Code, body)
	}
	// ...assert on body, then exercise update/delete/list the same way.
}
```

`newTestEnv(t)` spins up a fresh in-memory SQLite database per test (see
`server/internal/api/handlers_test.go`) and seeds one demo device, so most tests don't need to set
up fixtures by hand. `env.createSuperAdmin` (in `p1_handlers_test.go`) creates a role with every
permission and logs in, returning the headers map to pass as `env.do`'s last argument.

Run just this test while iterating:

```bash
cd server
go test ./internal/api/ -run TestProductCRUDAndDeviceResolution -v
```

## Checklist for your own endpoint

1. Does an existing store method already do what you need? If not, add one (or see
   [Add a New Data Model](03-backend-add-model-and-migration.md) if it needs a new table/field).
2. Write the handler: a request DTO, a response DTO, validate input, call the store, `s.audit(...)`
   if it mutates anything, `writeAPIJSON`/`adminErr` for the response.
3. Register the route in `router.go` with the permission code that matches what the action does.
4. Update `api/admin-api.md` with the new route's method, path, permission, and request/response
   shape.
5. Add a test in the same file as the handler (or a new `_test.go` next to it), driving it through
   `env.do` like a real client.
6. Verify: `go build ./... && go vet ./... && go test ./...` from `server/`.
7. If the admin console needs a UI for this, see [Add a New Admin Console Page](04-admin-add-page.md).

## Next

[Add a New Data Model](03-backend-add-model-and-migration.md).
