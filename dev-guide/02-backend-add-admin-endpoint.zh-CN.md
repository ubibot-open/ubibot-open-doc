# 新增一个后台 API 端点

*[English](02-backend-add-admin-endpoint.md)*

这个代码库里每一个 `/api/admin/*` 端点都是由同样四层构成的：**store**（数据库查询）、
**handler**（HTTP 请求/响应、权限校验、审计日志）、**router**（把 handler 接到某个"方法+路径"
上）、以及一个像真实客户端那样通过路由去调用它的**测试**。与其编一个玩具示例，这一章直接走一遍
Product 这套 CRUD 端点——`server/internal/store/product.go`、
`server/internal/api/product_handlers.go`——实际上是怎么搭起来的，因为它们本来就是一个完整的、
已经合并进去的真实例子，形状跟你要写的东西一模一样。照着它抄就行。

## 1. store 层

store 拥有所有数据库查询——handler 从来不直接碰 `gorm.DB`。一个 store 文件就是挂在 `*Store` 上
的一堆普通方法：

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

如果你的端点是读写一个已经存在的 model，你可能根本不需要新增 store 方法——先去 `internal/store/`
里找找有没有现成的方法能满足需求，再考虑加新的。如果你同时也要引入一个全新的 model（就像
`Product` 当初那样），先看《[新增数据模型](03-backend-add-model-and-migration.zh-CN.md)》。

## 2. handler 层

handler 负责解析请求、调用 store、然后通过这个代码库的两个小工具把响应写回去：`writeAPIJSON`
（所有响应统一用的信封格式）和用来解析请求/处理错误的 `decodeJSON`/`adminErr`（都在
`server/internal/api/httpx.go` 和 `middleware.go` 里）。一个 DTO 结构体精确控制了到底哪些字段会
出现在接口上——它故意不直接用 GORM 的 model 本身，这样以后 model 的存储结构变了，也不会悄悄改变
对外的 API：

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

上面这段代码里，每个会改数据的 handler 都做了两件事：

- **`s.audit(r, action, targetType, targetID, detail)`** —— 记录是谁做了什么，写进审计日志
  （`GET /api/admin/audit-logs`）。要在改动成功之后、写响应之前调用它。
- **在调用 store 之前先做校验**，不满足条件就返回 `400`，并且带上一句能让人看懂缺了什么/哪里不
  对的说明——store 层不负责生成给用户看的错误文案。

## 3. router 层

`server/internal/api/router.go` 里每个路由一行，外面套上它需要的权限校验：

```go
mux.HandleFunc("GET /api/admin/products", s.RequirePermission(model.PermDeviceRead, s.ListProducts))
mux.HandleFunc("POST /api/admin/products", s.RequirePermission(model.PermSystemManage, s.CreateProduct))
mux.HandleFunc("PATCH /api/admin/products/{id}", s.RequirePermission(model.PermSystemManage, s.UpdateProduct))
mux.HandleFunc("DELETE /api/admin/products/{id}", s.RequirePermission(model.PermSystemManage, s.DeleteProduct))
```

`RequirePermission(code, handler)`（在 `middleware.go` 里）会在真正调用你的 handler 之前，先检查
调用方的 bearer token 和它所属角色的权限——一共只有四个权限码可选（`model.PermDeviceRead`、
`PermDeviceWrite`、`PermAlertManage`、`PermSystemManage`；见
`server/internal/model/model.go`）。选哪个要看这个操作实际做了什么，`router.go` 里每组路由旁边
都已经写了选择的理由——比如 Product 的读接口挂在 `device:read` 上，因为它是某个设备的展示数据；
它的写接口挂在 `system:manage` 上，因为它是共享的参考数据，不是对某一台具体设备的改动。如果某个
端点压根不需要管理员会话就能访问（比较少见，大多数情况都不该这样），用 `RequireAdmin` 代替
`RequirePermission`（表示"登录了就行，不管什么角色"），或者如果是给第三方用的只读接口，用
`RequireApiKey`。

同时记得把新路由更新进 `ubibot-open-doc` 的 `api/admin-api.md`——那份参考文档的定位就是要完整、
准确地列出这个后台暴露的每一个路由。

## 4. 测试

测试文件和 handler 放在一起（`product_handlers_test.go`），用跟真实 HTTP 客户端一样的方式去驱动
路由——`env.do(t, method, path, body, authHeaders)`——而不是直接调用 handler 函数，这样测试也顺带
覆盖了路由和权限校验：

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
	// ...对 body 做断言，然后用同样的方式测 update/delete/list。
}
```

`newTestEnv(t)`（见 `server/internal/api/handlers_test.go`）会给每个测试起一个全新的内存 SQLite
数据库，并预先建好一台演示设备，所以大多数测试不用自己手动搭 fixture。`env.createSuperAdmin`
（在 `p1_handlers_test.go` 里）会创建一个拥有全部权限的角色并登录，返回可以直接作为 `env.do`
最后一个参数传进去的 headers map。

迭代的时候只跑这一个测试：

```bash
cd server
go test ./internal/api/ -run TestProductCRUDAndDeviceResolution -v
```

## 给你自己的端点用的检查清单

1. 有没有现成的 store 方法已经能满足需求？没有的话就加一个（如果需要新表/新字段，看
   《[新增数据模型](03-backend-add-model-and-migration.zh-CN.md)》）。
2. 写 handler：一个请求 DTO、一个响应 DTO，校验输入，调用 store，如果会改数据就调用
   `s.audit(...)`，用 `writeAPIJSON`/`adminErr` 写响应。
3. 在 `router.go` 里注册路由，权限码要跟这个操作实际做的事情匹配。
4. 把新路由的方法、路径、权限、请求/响应格式更新进 `api/admin-api.md`。
5. 在 handler 同一个文件（或旁边新建的 `_test.go`）里加测试，通过 `env.do` 像真实客户端那样调用。
6. 验证：在 `server/` 目录下跑 `go build ./... && go vet ./... && go test ./...`。
7. 如果管理控制台也需要一个界面来用这个端点，看
   《[新增后台页面](04-admin-add-page.zh-CN.md)》。

## 下一步

[新增数据模型](03-backend-add-model-and-migration.zh-CN.md)。
