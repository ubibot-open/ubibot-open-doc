# 新增数据模型

*[English](03-backend-add-model-and-migration.md)*

当你的端点需要一张还不存在的表时，才需要看这一章——如果只是给 `Device` 或其他已有 model 加个
字段，直接跳到第 2 步就行。如果你真正想要的是端点本身，《[新增一个后台 API 端点](02-backend-add-admin-endpoint.zh-CN.md)》
讲的是那一层；这一章讲的是它下面的 model。

## 1. 定义结构体

每个 model 都放在 `server/internal/model/model.go` 里，是一个带 GORM 标签的普通 Go 结构体。没有
单独的迁移 DSL，也没有 `.sql` 文件——结构体标签本身**就是**表结构。`Product` 是一个最近加的、
很精简的例子：

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

要遵循的约定，上面都能看到，而且跟文件里其他每个 model 都是一致的：

- 先写 `ID uint \`gorm:"primaryKey"\``。
- 每个字符串字段都写 `size:N`——这不是装饰用的，GORM/sqlite 驱动需要它来决定列类型。
- 真正必填的字段加 `not null`；必须唯一的字段加 `uniqueIndex`（GORM 会在迁移时自动帮你建索引）。
- 如果这一行的创建时间/最后修改时间有意义，就加上 `CreatedAt time.Time` / `UpdatedAt time.Time`
  （GORM 在创建/保存时会自动帮你维护这两个字段——不需要自己手动赋值）。`Device.PendingCmd string
  \`gorm:"type:text"\`` 展示的是另一种模式：一个存放不定长字符串（这里是一段原始 JSON）的字段，
  而不是一个有明确长度上限的短字符串。
- 显式写一个 `TableName()` 方法——GORM 自己会把结构体名做复数变形来猜表名，但这个文件里的每个
  model 都不依赖那个自动推断，而是显式写清楚。

## 2. 注册进 AutoMigrate

新 model 只有一个地方需要加进去——`server/internal/store/db.go` 的 `Open()` 函数：

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

`AutoMigrate` 会在每次启动时创建还不存在的表（并给已有表加上新增的列）——没有单独的"跑一遍迁移"
这个步骤，也没有降级迁移。这是故意做得很简单的（见《[架构总览](../architecture/overview.md)》里
"默认最小化"这条原则），也正因为这样，每次加列都必须是"只加不改"的：`AutoMigrate` 会给已有的行
把新列填上零值，但它不会帮你重命名或删除某一列，也不会给一个之前允许为空、现在改成 `not null`
的列去补历史数据。

## 3. 小心 `PID` → `p_id` 这个命名坑

这是一个在这个代码库里真实踩到过、真实修复过的 bug，不是假设出来的：GORM 默认的命名策略是在大写
字母前插入下划线，把 Go 字段名转成列名——而它会把一个叫 `PID` 的字段当成"P"+"ID"两个词，转出来的
列名是 `p_id`，不是 `pid`。任何按字面字符串 `"pid"` 去查询的代码——比如 `store.ProductsByPIDs`
里的 `Where("pid IN ?", pids)`——如果 `PID` 字段没有像第 1 步那样显式打标签，就会悄无声息地失败
（报错 `no such column: pid`）：

```go
PID string `gorm:"column:pid;size:64;not null;uniqueIndex"`
```

**`Device.PID` 就有这个一模一样的潜在问题，而且它没有打这个标签**——只是因为这个仓库里目前没有
任何代码直接对 `devices` 表按 `pid` 做 `Where("pid = ?", ...)` 查询（现有的查找都是走 `SN`，或者
像 `toDeviceDTO` 解析 product 那样在 Go 代码这一侧做匹配），所以这个坑一直没被真正踩中。如果你
以后要加一个直接按 `pid` 过滤 `Device` 的查询，先把同样的 `gorm:"column:pid"` 标签加上——别等踩了
坑才发现。

更普遍的经验：任何全大写或者带缩写的字段名（`PID`、`SN`、`ID` 跟别的词拼在一起、`URL`、`API`……）
在你要拿它写一个基于原始字符串的查询之前，都值得先去核实一下 GORM 实际生成的列名是什么。拿不准
的时候，写一个临时的测试，真的对着一个内存里的 SQLite 数据库跑一下这个查询（见下一章的测试写法），
而不是凭猜测认定列名——这个 bug 当初就是这么被发现的。

## 检查清单

1. 按上面的约定把结构体加进 `model.go`（标签、必要时的 `CreatedAt`/`UpdatedAt`、显式的
   `TableName()`）。
2. 把它加进 `store/db.go` 的 `AutoMigrate(...)` 调用里。
3. 如果有字段是全大写/带缩写的名字，而你以后会按原始列名去查询它，现在就显式打上标签
   （`gorm:"column:..."`）——不要依赖默认命名策略的猜测。
4. 接下来看《[新增一个后台 API 端点](02-backend-add-admin-endpoint.zh-CN.md)》，把真正暴露这个
   model 的 store 方法、handler 和路由写出来。

## 下一步

[新增后台页面](04-admin-add-page.zh-CN.md)。
