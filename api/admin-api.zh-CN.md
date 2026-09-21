# 管理后台 API 参考

*[English](admin-api.md)*

这是内置管理控制台自己在用的 API——下面每一个路由都挂在 `/api/admin/*` 下，跟控制台界面本身用的
是同一个地址（默认 `http://<host>:8080`）。如果你要做一个替代前端、写一个批量操作脚本，或者只是
想知道控制台里某个按钮实际调的是什么接口，这份文档会有用。如果你只是想从外部只读访问设备/遥测
数据，[开放 API](open-api.zh-CN.md)（一个小得多、用 API 密钥认证的接口面）很可能更适合——别为了
读数据就去申请管理员凭证。

## 约定

**认证**：`POST /api/admin/login` 返回一个 bearer token；其余每个路由都要求
`Authorization: Bearer <token>`。缺少这个请求头是 `401` `bearer_token_missing`；token
过期或无效是 `401` `session_invalid_or_expired`。

**权限**：大部分路由还要求调用者的角色具备四个权限码中的一个（`super_admin` 角色会跳过所有
检查）：

| 权限 | 管控范围 |
|---|---|
| `device:read` | 读取设备、记录、字段设置、告警规则/事件、图标 |
| `device:write` | 改名/启用/停用/删除设备，下发/取消指令，编辑字段设置 |
| `alert:manage` | 创建/删除告警规则，处理告警事件 |
| `system:manage` | 用户、角色、审计日志、API 密钥、文件、字典、参数、图标上传、产品、系统监控指标 |

少数几个路由（登录本身、`me`、通知、仪表盘、字典列表）只要求*已登录*（`RequireAdmin`），不需要
具体的权限码——下面每个路由会分别标注。权限校验不通过是 `403` `forbidden`。

**响应信封**：每个响应，不管成功还是失败，都是 JSON，会自动附带一个 `timestamp`（Unix 秒）
字段——下面的示例里省略了它，免得重复写 60 遍；默认它一直都在。

**错误**：`{"code": "<稳定的code>", "message": "<英文文本>", "timestamp": ...}`。`code` 可以
安全地拿来做客户端本地化错误提示的匹配依据；`message` 永远是英文，不是给用户直接看的。常见的
code：`invalid_id`、`invalid_request_body`、`forbidden`、`internal_error`、`device_not_found`、
`file_not_found`。大多数 `400` 都有一个以"缺失/无效的字段"命名的具体 code（比如
`name_required`、`invalid_status`）；查不到对应关系的就统一用通用的 `error`。

**分页**：只要某个路由返回的是 `"list"` + `"total"`，就用 `page`（默认 1）和 `page_size`
（默认 20）这两个查询参数——这两个参数本身没有强制的 `page_size` 上限，不过有些列表查询内部
还会再加一层上限（比如设备列表不管请求多少，每页最多返回 200 条）。

**审计日志**：下面每一个会改数据的路由都会往审计日志（`GET /api/admin/audit-logs`）写一条记录
（操作人、动作、目标类型/id、一句简短的说明、调用方 IP）——这里统一说明一次，不在每个路由下面
重复。

---

## 认证

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `POST /api/admin/login` | — | `{"username","password"}` | `{"token","expires_in","username"}`——失败时 `401 invalid_credentials` |
| `GET /api/admin/me` | 已登录 | — | `{"username"}` |

---

## 设备

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `GET /api/admin/devices` | `device:read` | `page`、`page_size` | `{"list":[deviceDTO...],"total"}` |
| `GET /api/admin/devices/data-warehouse` | `device:read` | `page`、`page_size` | `{"list":[deviceDTO + last_record + field_meta],"total"}`——每行内联着最新的遥测数据，字段名/单位/图标都提前解析好了，前端不需要再多发 N 个请求 |
| `GET /api/admin/devices/{id}` | `device:read` | 路径参数 `id` | `{"device":deviceDTO,"records":[recordDTO...]}`（最近 20 条记录） |
| `GET /api/admin/devices/{id}/records` | `device:read` | 路径参数 `id`；查询参数 `start`、`end`（unix 时间戳，都可选/不限），`page`、`page_size` | `{"list":[recordDTO...],"total"}` |
| `PATCH /api/admin/devices/{id}` | `device:write` | `{"name"}`（允许留空——清空后会退回显示 SN） | `deviceDTO`（不带外层包装） |
| `POST /api/admin/devices/{id}/status` | `device:write` | `{"status"}`（`1`=启用，`2`=停用） | `{"message":"ok"}` |
| `DELETE /api/admin/devices/{id}` | `device:write` | — | `{"message":"ok"}`——**不可逆**，会连同这台设备引用的每一条遥测记录和告警规则/事件一起删除 |

**`deviceDTO`**：
```json
{
  "id": 1, "pid": "ubibot_open_dev_v1", "sn": "sn_ws1_20001_1", "name": "Warehouse Sensor 1",
  "status": 1, "online": true, "last_seen_at": 1788950400, "created_at": 1788900000,
  "pending_command": { "action": "reboot" },
  "product_name": "WS1B Sensor"
}
```
当没有排队中的指令 / 没有匹配的 Product 时，`pending_command` 和 `product_name` 会被整个省略
（不是 `null`）——见下面的[指令](#指令)和[产品](#产品)。

**`recordDTO`**：`{"ts": 1788950400, "d": {"field1": 25.6, "field2": 60.2}}`——`d` 原样反映
设备实际上报了哪些 `field1`..`field20` key（协议 §5/§6），没有固定 schema。

设备从来不会通过这个 API 直接创建——它要么在第一次成功上报的那一刻出现（协议 §5/§7），要么通过
[批量导入](#批量设备管理)提前建好。

---

## 批量设备管理

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `POST /api/admin/devices/import` | `device:write` | `{"rows":[{"sn","pid","name"}...]}`（最多 5000 行；`name` 可选） | `{"created":int,"skipped":["sn",...],"failed":["row 3: ...",...]}` |
| `GET /api/admin/devices/export.csv` | `device:read` | — | `text/csv` 格式的内容（不是 JSON），所有设备，不分页 |

导入是逐行处理的，不是全部成功或全部失败：重复的 `sn` 会落进 `skipped`（不算错误），缺必填字段
的行会带着原因落进 `failed`，批次里其余的行照样会导入成功。一台预先导入的设备，一旦真的开始
上报，行为就跟自动创建的设备完全一样——靠 `sn` 匹配。导出的列：
`id,pid,product_name,sn,name,status,online,last_seen_at,created_at`。

---

## 产品

设备型号的展示元数据（给某个 `pid` 起的名字/描述）——在读取时按 `pid` 匹配到设备上，从来不是
外键。删除或重命名一个 Product 从不会动任何设备记录。

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `GET /api/admin/products` | `device:read` | — | `{"list":[productDTO...]}` |
| `POST /api/admin/products` | `system:manage` | `{"pid","name","description"}`（`pid`/`name` 必填；`pid` 必须唯一） | `productDTO`（不带外层包装） |
| `PATCH /api/admin/products/{id}` | `system:manage` | `{"name","description"}`——`pid` 创建后**不可修改** | `{"message":"ok"}` |
| `DELETE /api/admin/products/{id}` | `system:manage` | — | `{"message":"ok"}`——只移除展示元数据 |

`productDTO`：`{"id","pid","name","description","created_at"}`。

---

## 指令

给设备排一个指令，随它*下一次*上报的响应捎带过去（协议 §9）——发了不管，没有应答，每台设备同一
时间最多排一个指令。

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `POST /api/admin/devices/{id}/commands` | `device:write` | `{"action":"reboot"}` 或 `{"action":"set_interval","seconds":N}`（`N` 必须在 60–86400 之间） | `{"message":"queued","cmd":{...}}` |
| `DELETE /api/admin/devices/{id}/commands` | `device:write` | — | `{"message":"ok"}`——取消一个还没送达的指令；没有排队中的指令就什么都不做 |

设备具体怎么、什么时候收到这个指令，见
[协议文档 §9](../protocol/hardware-communication-protocol.zh-CN.md#9-指令下发管理员触发可选)。

---

## 字段设置

针对某台设备，覆盖 `field1`..`field20` 某个 key 的显示方式（名称/单位/图标）——没有覆盖时，
回退到全局的图标模板库（[图标库](#图标库)）。

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `GET /api/admin/devices/{id}/field-settings` | `device:read` | 路径参数 `id` | `{"list":[fieldSettingDTO x20]}`——永远是 field1..field20 全部 20 个，每个都按"覆盖→模板→空"这个链路解析好 |
| `POST /api/admin/devices/{id}/field-settings/{key}` | `device:write` | 路径参数 `id`、`key`；`{"name","unit","svg"}`（`svg` ≤64KB，非空时必须包含 `<svg`） | `fieldSettingDTO`（不带外层包装，写入后的值） |
| `DELETE /api/admin/devices/{id}/field-settings/{key}` | `device:write` | 路径参数 `id`、`key` | `{"message":"ok"}`——清空覆盖，回退到模板 |

`fieldSettingDTO`：`{"key","name","unit","svg","is_custom"}`。

---

## 告警

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `GET /api/admin/devices/{id}/alert-rules` | `device:read` | 路径参数 `id` | `{"list":[alertRuleDTO...]}` |
| `POST /api/admin/devices/{id}/alert-rules` | `alert:manage` | `{"field","op","threshold"}`——`op` 是 `> >= < <= ==` 中的一个 | `alertRuleDTO`（不带外层包装） |
| `DELETE /api/admin/alert-rules/{id}` | `alert:manage` | — | `{"message":"ok"}` |
| `GET /api/admin/alert-events` | `device:read` | `page`、`page_size`、`device_id`、`status`（都是可选的过滤条件） | `{"list":[alertEventDTO...],"total"}` |
| `POST /api/admin/alert-events/{id}/resolve` | `alert:manage` | — | `{"message":"ok"}` |

`alertRuleDTO`：`{"id","device_id","field","op","threshold","enabled"}`。
`alertEventDTO`：`{"id","device_id","device_name","rule_id","type","message","status","triggered_at","resolved_at"}`
——`type` 是 `threshold` 或 `offline`，`status` 是 `open` 或 `resolved`。离线告警是由周期性扫描
触发的（不需要规则）；阈值告警需要一条规则。

---

## RBAC：角色、管理员用户、审计日志

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `GET /api/admin/roles` | `system:manage` | — | `{"list":[roleDTO...]}` |
| `POST /api/admin/roles` | `system:manage` | `{"name","code","permissions":[...]}`（`name`/`code` 必填） | `roleDTO`（不带外层包装） |
| `PATCH /api/admin/roles/{id}` | `system:manage` | `{"name","permissions"}`——`code` 不可修改 | `{"message":"ok"}` |
| `DELETE /api/admin/roles/{id}` | `system:manage` | — | `{"message":"ok"}` |
| `GET /api/admin/users` | `system:manage` | — | `{"list":[adminUserDTO...]}` |
| `POST /api/admin/users` | `system:manage` | `{"username","password","role_id"}`（都必填） | `adminUserDTO`（不带外层包装） |
| `PATCH /api/admin/users/{id}` | `system:manage` | `{"role_id","password"}`——两个字段任选一个或都传，各自独立生效 | `{"message":"ok"}` |
| `DELETE /api/admin/users/{id}` | `system:manage` | — | `{"message":"ok"}`——如果目标是自己，返回 `400 cannot_delete_own_account` |
| `GET /api/admin/audit-logs` | `system:manage` | `page`、`page_size` | `{"list":[auditLogDTO...],"total"}` |

`roleDTO`：`{"id","name","code","permissions":[...]}`——内置的 `super_admin` 角色 `permissions`
是 `["*"]`，其他角色是四个权限码里的一个明确列表。
`adminUserDTO`：`{"id","username","role_id","role_name","created_at"}`。
`auditLogDTO`：`{"id","username","action","target_type","target_id","detail","ip","created_at"}`。

---

## 通知

目前只有站内通知——没有邮件/短信/webhook 渠道。控制台是轮询这个接口，不是基于推送的。

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `GET /api/admin/notifications` | 已登录 | `page`、`page_size` | `{"list":[notificationDTO...],"total","unread"}` |
| `POST /api/admin/notifications/{id}/read` | 已登录 | — | `{"message":"ok"}` |
| `POST /api/admin/notifications/read-all` | 已登录 | — | `{"message":"ok"}` |

`notificationDTO`：`{"id","type","level","title","content","status","created_at"}`——`level` 是
`info`/`warning`/`critical`，`status` 是 `unread`/`read`。

---

## API 密钥（开放 API 管理）

管理用来给[开放 API](open-api.zh-CN.md)认证的密钥——密钥具体怎么用见那份文档。

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `GET /api/admin/api-keys` | `system:manage` | — | `{"list":[apiKeyDTO...]}` |
| `POST /api/admin/api-keys` | `system:manage` | `{"name"}` | `{"key":apiKeyDTO,"raw_key":"..."}` |
| `POST /api/admin/api-keys/{id}/revoke` | `system:manage` | — | `{"message":"ok"}` |

`apiKeyDTO`：`{"id","name","prefix","revoked","last_used_at","created_at"}`——**永远不包含明文
密钥**；创建响应里的 `raw_key` 是唯一一次能看到它的机会，服务端只存哈希值。丢了就只能撤销、
重新创建一个。

---

## 文件、字典、参数

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `GET /api/admin/files` | `system:manage` | — | `{"list":[fileAssetDTO...]}` |
| `POST /api/admin/files` | `system:manage` | `multipart/form-data`：字段 `category`（默认 `"other"`），文件字段 `file`（必填，≤32MB） | `fileAssetDTO`（不带外层包装） |
| `DELETE /api/admin/files/{id}` | `system:manage` | — | `{"message":"ok"}` |
| `GET /api/admin/dict` | 已登录（不需要具体权限码） | 查询参数 `type`（可选过滤条件） | `{"list":[dictEntryDTO...]}` |
| `POST /api/admin/dict` | `system:manage` | `{"type","key","label","sort"}` | `dictEntryDTO`（不带外层包装） |
| `PATCH /api/admin/dict/{id}` | `system:manage` | `{"label","sort"}`——`type`/`key` 不可修改 | `{"message":"ok"}` |
| `DELETE /api/admin/dict/{id}` | `system:manage` | — | `{"message":"ok"}` |
| `GET /api/admin/params` | `system:manage` | — | `{"list":[systemParamDTO...]}` |
| `PATCH /api/admin/params/{key}` | `system:manage` | `{"value","description"}` | `systemParamDTO`（不带外层包装） |

`fileAssetDTO`：`{"id","category","filename","size","sha256","created_at"}`——上传的文件存在
服务器的文件目录下，写入时会算好 SHA-256 哈希。
`dictEntryDTO`：`{"id","type","key","label","sort"}`——给下拉选项用的通用 key/label 查找表；
`type` 给条目分组（比如一个 `command_type` 字典）。

**系统参数是一个开放式的键值存储**——`PATCH` 接受任意 `key`，但只有两个真的会被读回来影响
运行时行为：

| Key | 效果 | 默认值 |
|---|---|---|
| `rate_limit_per_minute` | 设备侧接口每个 IP 的限速，立刻生效（不用重启） | `120` |
| `offline_grace_minutes` | 设备多久没动静就会被标记为离线，立刻生效 | `2` |

---

## 图标库

全局的默认字段模板（按 `field1`..`field20` 这个 *key 名字*配置名称/单位/图标，不是按设备）——
当某台设备自己的[字段设置](#字段设置)没有覆盖时，就回退到这里。

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `GET /api/admin/icons` | `device:read` | — | `{"list":[iconDTO...]}` |
| `POST /api/admin/icons` | `system:manage` | `{"key","name","unit","svg"}`（`key`/`name` 必填；`svg` ≤64KB，非空时必须包含 `<svg`）——upsert，对同一个 `key` 重新上传会替换它 | `iconDTO`（不带外层包装） |
| `DELETE /api/admin/icons/{key}` | `system:manage` | 路径参数 `key` | `{"message":"ok"}` |

`iconDTO`：`{"key","name","unit","svg","created_at"}`。SVG 是原始标签内联在 JSON 请求体里传的
（不是 multipart）——小到不值得再走一遍文件上传的流程。

---

## 系统监控与仪表盘

| 方法与路径 | 权限 | 请求 | 响应 |
|---|---|---|---|
| `GET /api/admin/system/metrics` | `system:manage` | — | `{"go_version","goroutines","heap_alloc_bytes","uptime_seconds","db_size_bytes","device_total","open_alerts","unread_notifications"}` |
| `GET /api/admin/dashboard/summary` | 已登录 | — | `{"device_total","device_online","open_alerts","today_records"}` |
| `GET /api/admin/dashboard/trends` | 已登录 | — | `{"days":[{"day":"2026-01-15","count":42}, ...]}`（最近 7 天） |

---

## 这里没有的东西

没有 OTA，没有 MQTT/CoAP，没有多租户/客户账号，没有邮件/短信/webhook 告警渠道，登录没有
验证码/失败锁定——原因见
[协议文档的范围说明](../protocol/hardware-communication-protocol.zh-CN.md#0-项目范围与本版本修订说明)
和[架构总览](../architecture/overview.zh-CN.md#这套代码依赖的设计原则)：这是一个面向内部/教学
场景的工具，把保持简单放在了比覆盖商业平台的每一个功能更优先的位置。
