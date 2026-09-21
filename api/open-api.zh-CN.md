# 开放 API 参考

*[English](open-api.md)*

一个很小的、只读的、用 API 密钥认证的接口面，面向第三方集成——就是组织首页宣传的那句"用于二次
开发的 RESTful API"。如果你要做的是内置管理控制台的替代品，或者想自动化某个管理员原本要点鼠标
才能做的操作，你想要的可能是[管理后台 API 参考](admin-api.zh-CN.md)；这份文档只暴露一个
只读的外部客户端真正会需要的东西：设备列表和它们的历史数据。

## 获取一个 API 密钥

1. 登录管理控制台。
2. 进入 系统 → 开放 API，创建一个密钥（只需要一个名字）。
3. 响应里包含明文密钥，**只显示这一次**——立刻复制下来。服务器只会存它的哈希值，之后再也没有
   办法取回来。如果丢了，撤销它，再创建一个新的。

## 认证

每个请求都必须在 `X-Api-Key` 请求头里带上密钥——不是 `Authorization`，那个是留给管理控制台
会话用的：

```bash
curl -H "X-Api-Key: <your key>" http://localhost:8080/api/open/v1/devices
```

| 情况 | 响应 |
|---|---|
| 请求头缺失 | `401` `{"code":"api_key_missing","message":"missing api key", ...}` |
| 密钥无效或已撤销 | `401` `{"code":"api_key_invalid_or_revoked","message":"invalid or revoked api key", ...}` |

密钥除了"有效"之外没有角色或权限范围的概念——跟管理员会话不一样，它上面没有叠加 RBAC。可以
随时在同一个 系统 → 开放 API 页面撤销一个密钥；撤销后立刻失效。

## 响应信封

每个响应（成功或失败）都是 JSON，会自动附带一个 `timestamp`（Unix 秒）字段，外加下面各接口
文档里列出的字段——为了避免重复写 20 遍，这一点不会在每个接口下面再重复说明。

## 接口

### `GET /api/open/v1/devices`

列出设备——形状比管理控制台看到的要小得多（没有 `pid`、`status`、`pending_command`、
`product_name`；只够识别一台设备、知道它是否在上报）。

**查询参数**

| 参数 | 默认值 | 说明 |
|---|---|---|
| `page` | 1 | 页码 |
| `page_size` | 20 | 每页行数 |

**响应**

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

某台设备的历史数据，通过它的数字 `id`（来自上面的列表）。

**查询参数**

| 参数 | 默认值 | 说明 |
|---|---|---|
| `start` | 0（不限） | Unix 秒，`ts` 的下界（含） |
| `end` | 0（不限） | Unix 秒，`ts` 的上界（含） |
| `page` | 1 | 页码 |
| `page_size` | 20 | 每页行数 |

**响应**

```json
{
  "list": [
    { "ts": 1788950400, "d": { "field1": 25.6, "field2": 60.2 } }
  ],
  "total": 1
}
```

`d` 里具体有哪些 `field1`..`field20` key，取决于那条记录当时携带了什么（协议 §6）——没有固定
schema，原样反映设备上报的内容。

```bash
curl -H "X-Api-Key: <your key>" \
  "http://localhost:8080/api/open/v1/devices/1/records?start=1788950000&end=1788960000"
```

