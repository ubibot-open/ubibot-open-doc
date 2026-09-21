# UbiBot Open —— 架构总览

*[English](overview.md)*

这是一份"整个系统是怎么拼起来的"的文档。想要一步步的搭建说明，见
[部署、烧录与上电指南](../guides/deployment-flashing-guide.zh-CN.md)；想要精确的线格式，见
[硬件通信协议](../protocol/hardware-communication-protocol.zh-CN.md)。这个页面假定你两份都还没读过，
只是想先了解一下整个系统的大致形状。

## 五个仓库

| 仓库 | 角色 | 技术栈 |
|---|---|---|
| [ubibot-open-server](https://github.com/ubibot-open/ubibot-open-server) | 面向设备的 HTTP 后端 + 管理控制台，编译成一个可执行文件 | Go、React/TypeScript |
| [ubibot-open-ws1b](https://github.com/ubibot-open/ubibot-open-ws1b) | WS1B 传感设备的参考固件 | C、ESP-IDF（ESP32-C5） |
| [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync) | 用于查看串口日志、调试和设备配网的桌面工具 | C++、Qt 6/QML |
| [ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator) | 从零写的、可在主机上构建的设备模拟器，协议行为跟固件完全一致——不需要硬件，也是整个设备侧协议约 7 个文件的精简范例 | C |
| ubibot-open-doc（本仓库） | 上面提到的一切：协议规范、部署指南、这份总览 | Markdown |

每个仓库都是独立部署、独立打版本号的。把它们联系在一起的只有两样东西：线上的通信协议（设备 ↔
后端，走 HTTP）和一根物理 USB 线（电脑 ↔ 设备，走串口）——没有共享代码，没有共享构建，没有共享
配置文件。

## 系统图

```
┌──────────────────────┐    USB serial: logs, debugging,     ┌───────────────────────┐
│  ubibot-serial-sync   │    provisioning (protocol §1.2)     │   ubibot-open-ws1b     │
│  (desktop tool)       │ ───────────────────────────────────▶│   firmware, on device  │
│                       │◀─────────────────────────────────── │                        │
└──────────────────────┘                                      └───────────┬────────────┘
                                                                           │ HTTP (protocol §2–§9)
                                                                           │ POST /api/v1/auth/time
                                                                           │ POST /api/v1/data/report
                                                                           ▼
┌──────────────────────┐    Bearer token, admin session       ┌───────────────────────┐
│   Admin's browser     │ ───────────────────────────────────▶│   ubibot-open-server   │
│  (the bundled React   │◀─────────────────────────────────── │  Go backend + embedded │
│   admin console)      │      /api/admin/*                   │  React admin console;  │
└──────────────────────┘                                      │  single binary, SQLite │
                                                                └───────────┬────────────┘
┌──────────────────────┐    X-Api-Key header                              │
│  Third-party client   │ ───────────────────────────────────▶ /api/open/v1/* (read-only)
│  (your own script/app)│◀─────────────────────────────────── ┘
└──────────────────────┘
```

三个独立的"调用方"各自通过自己的认证方式和自己的路由前缀访问后端——设备从不涉及管理员会话，
管理员会话也从不涉及 API 密钥：

- **设备**调用不需要认证、以 pid+sn 标识身份的 `/api/v1/*` 路由（协议 §2–§9）。
- **管理控制台**（以及任何直接驱动同一套接口的人）用 `POST /api/admin/login` 拿到的 bearer
  token 调用 `/api/admin/*`，按路由分别受 RBAC 权限管控。
- **第三方集成**用 `X-Api-Key` 请求头调用只读的 `/api/open/v1/*` 路由，密钥从管理控制台签发和
  撤销。

完整的路由表见 [api/admin-api.md](../api/admin-api.zh-CN.md) 和
[api/open-api.md](../api/open-api.zh-CN.md)。

## 数据流：从设备到仪表盘

1. **身份**：设备知道自己的 `pid`（同型号所有设备共用）和 `sn`（每台唯一）——要么在构建时通过
   `idf.py menuconfig` 固化进去，要么之后通过串口（`SetupDevice`，协议 §1.2）设置，不需要重新
   烧录。没有注册这一步：只要有一个后端没见过的 `sn` 第一次上报，后端就会创建一条 `Device`
   记录，除非管理员已经通过批量导入提前注册过它。
2. **网络配置**：WiFi 凭证和服务器地址走的是同一套逻辑——一个 `menuconfig` 默认值，可以在运行时
   通过串口（`SetupWifi`/`SetupServer`）覆盖。
3. **时间同步**（仅当设备的时钟看起来不对时）：`POST /api/v1/auth/time` 返回服务器的时钟；不需要
   认证，因为它不会泄露任何设备专属的信息。
4. **上报**：`POST /api/v1/data/report` 携带一条或多条带时间戳的 `field1`..`field20` 读数。后端
   会 upsert 这台设备、存储读数（60 秒窗口内的多条记录会合并成一条，协议 §5），如果管理员排了
   一个指令，还会在响应里嵌入一个 `cmd` 对象（协议 §9：`reboot` 或 `set_interval`）。设备会在
   下一次休眠之前执行它；不管怎样都不会回应答给服务器。
5. **仪表盘**：管理控制台查询/轮询的就是 report handler 刚刚写入的那些 SQLite 行——设备列表、
   "数据仓库"（内联展示每台设备的最新读数）、单台设备的历史记录，以及基于同一份数据评估出来的
   阈值/离线告警。

## 后端内部结构

```
server/
  cmd/server/       main.go —— 参数/环境变量解析，类似 NVS 的启动引导（初始化第一个管理员账号、
                     默认系统参数），把所有东西接在一起
  internal/
    api/            HTTP handler + 路由（net/http 的 ServeMux，Go 1.22 的方法+{通配符}路由——
                     没有用框架）
    auth/           密码哈希（bcrypt）和管理员会话 token
    model/          GORM 行类型——持久化的表结构
    protocol/       面向设备的线格式类型（跟协议文档几乎一一对应）
    store/          持久化——所有查询都在这一层，handler 从不直接碰 gorm.DB
    webui/          通过 go:embed 把 admin/ 构建出来的 Vite 产物内嵌进去
admin/              React 管理控制台（Vite + TypeScript + Ant Design），单独构建，
                     由 build.sh/build.ps1 内嵌进服务端可执行文件
```

读代码之前值得先知道的几件事：

- **单个 SQLite 文件，GORM `AutoMigrate`。** 没有迁移目录——给 `model` 结构体加一个字段就够了；
  GORM 会在下次启动时非破坏性地把这一列加到已有的表上。（重命名一个 Go 结构体字段，可能会用一种
  值得仔细核实的方式悄悄改变它映射的列名——`Product.PID` → `column:pid` 标签就是一个真实案例，
  GORM 的命名策略跟字面上看字段名猜出来的结果不一样。）
- **一个可以替换的时钟，而不是到处撒 `time.Now()`。** handler 读的是 `s.Now()`；测试会把它换成
  一个固定的函数，这样跟时间窗口有关的逻辑（±5 分钟的上报校验、离线告警的宽限期）就是确定性的，
  不用跟系统时钟赛跑。
- **测试打的是真实路由**，不是直接打 store——每个测试都是 `httptest.NewRecorder()` +
  `router.ServeHTTP()`，对着一个内存里的 SQLite 数据库，所以测试走的是跟真实请求一样的
  认证/权限/handler 链路。

## 固件内部结构（ubibot-open-ws1b）

```
main/
  main.c            app_main() 的接线逻辑 + 唯一一个长期存在的任务：连接 → （可能的话时间同步）→
                     采样 → 上报 → 休眠，通过深度睡眠 + 定时器唤醒无限循环
  provisioning.c    协议 §1.2 —— 在运行时覆盖 WiFi/服务器/sn 的串口指令，持久化到 NVS，
                     每次开机都重新读回内存（深度睡眠会清空内存）
  command.c         协议 §9 —— 服务端下发的 reboot/set_interval 指令，跟 provisioning.c
                     一样的"NVS 覆盖"模式，单独做成一个模块是因为这是不同的发起方（服务器，
                     不是技术人员）在下发不同类型的指令
  json_payload.c    构建两个面向设备的请求体（时间同步、数据上报）
  osi.c             main/ 其余部分依赖的一层薄 FreeRTOS 封装（task/queue/mutex）
app_driver/         外设驱动：WiFi（wifi_connect.c）、HTTP 客户端（http_client.c）、
                    传感器（sht30/ltr308/ub_dt_p1/stk8323）、电源/睡眠管理
```

一个值得注意的反复出现的模式：**每一个运行时可覆盖的设置都是"Kconfig 默认值，如果曾经写过 NVS
值就用 NVS 覆盖它，每次开机都重新加载进一个内存变量"。** `provisioning.c` 和 `command.c` 各自
独立实现了同样的这套形状，用来应对不同的触发方（技术人员通过串口 vs. 服务器通过 report 响应），
而不是共享一个通用的"设置"抽象——这是刻意重复的，为的是让每个模块的影响范围保持独立（各自的
理由见每个文件自己的头部注释）。

## 这套代码依赖的设计原则

这些原则没有在别的地方统一写下来，值得在这里明确说一遍：

- **线的两端都默认最小化。** 设备协议没有签名、没有会话 token、没有通用的指令推送通道（协议
  文档 §0）——同样的思路后来也延伸到了管理控制台的登录环节（没有验证码/失败锁定/强制登出）,
  这个功能被提出来讨论过,但没有采纳。这是一个面向内部/教学场景、力求"配置越少越好用"的工具，
  不是一个加固过的多租户 SaaS。
- **宁可"发了不管"，也不做确认重试。** 串口配网的应答只是提供信息，不代表送达确认；指令下发
  （协议 §9）干脆完全没有应答。这样实现起来更简单、更容易推理，代价是没有送达保证——这是一个
  写在明面上的权衡取舍，不是藏起来的。
- **展示用的元数据永远不做外键。** `Product` 和某台设备解析出来的 `product_name`，是在读取时
  按 `pid` 字符串相等匹配的，而不是 `Device` 表上的一个 `product_id` 列——删除或重命名一个
  Product，永远不可能让设备记录变成孤儿或者被级联影响。
- **只有真的需要了才扩大范围。** OTA、BLE 配网，以及给一款假想中的第二款硬件设计的传感器抽象层，
  目前都是刻意不在范围内的——不是因为难，而是因为这个项目目前没有任何东西真的需要它们（协议
  文档"移除的能力"附录里有更长的一份清单，记录了哪些东西是刻意简化掉的）。

## 接下来看什么

- [硬件通信协议](../protocol/hardware-communication-protocol.zh-CN.md) —— 精确的
  设备↔服务器线格式。
- [部署、烧录与上电指南](../guides/deployment-flashing-guide.zh-CN.md) —— 三个组件的
  实操搭建说明。
- [管理后台 API 参考](../api/admin-api.zh-CN.md) / [开放 API 参考](../api/open-api.zh-CN.md)
  —— 后端暴露的每一个 HTTP 路由。
- [CONTRIBUTING.md](https://github.com/ubibot-open/.github/blob/main/CONTRIBUTING.md)（暂无
  中文版）—— 怎么提出或提交一个改动。
