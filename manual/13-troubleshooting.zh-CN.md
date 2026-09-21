# 常见问题排查

*[English](13-troubleshooting.md)*

按"具体是哪里坏了"分组。如果你的问题不在下面，先看看服务器自己的终端输出——大部分故障都会在
那里打出有用的信息。

## 登录不了

| 症状 | 解决办法 |
|---|---|
| 忘记管理员密码 | 停掉 `ubibot-server`，删掉 admin 表，重新启动——这会重新走一遍[第 2 章](02-deploy-server.zh-CN.md)里"第一次运行"的逻辑，创建一个全新的 `admin` 账号和一个新生成的密码（设备数据不受影响）：`sqlite3 data/ubibot.db "DELETE FROM admin_users;"`（如果你设置过 `UBIBOT_DB_PATH`，路径要相应调整） |
| 登录页面加载不出来/一片空白 | 看服务器终端里有没有 `embedded web UI not built — serving API only` 这句话。如果有，说明这个可执行文件里根本没构建进管理前端——跑一下 `./build.sh`/`.\build.ps1`（不是单纯的 `go build`），然后重启 |

## 设备一直不上线

按顺序过一遍[第 6 章](06-verify-device-online.zh-CN.md)的三个核对点——设备连上 WiFi，然后成功
上报，然后出现在仪表盘上。如果上报发出去了但被拒绝了，响应里的 `c` 码能告诉你原因：

| `c` 码 | 原因 | 解决办法 |
|---|---|---|
| `1002` | 时间戳超出了 ±5 分钟的窗口 | 设备的时钟漂移了——确认它在收到这个错误之后会调用时间同步接口重新校准 |
| `1003` | 请求体格式不对 | 固件序列化那边有 bug——对照[协议 §5](../protocol/hardware-communication-protocol.zh-CN.md#5-数据上报唯一面向设备的数据接口) 检查 |
| `1103` | 设备被管理员停用了 | 去设备的详情页重新启用它——见[第 8 章](08-managing-devices-and-products.zh-CN.md) |
| `1900` | 被限速了 | 上报频率超过了配置的上限（[第 12 章](12-system-settings-and-monitor.zh-CN.md)的参数页面）；等下一个周期，或者检查是不是有异常的重试循环 |

完全没有响应（连拒绝都没有）？确认你配置的服务器地址/端口是对的，`ubibot-server` 真的在运行，
并且设备和运行后端的那台机器之间的网络连接没有被防火墙或者错误的局域网地址挡住。

## 发了配网指令，什么反应都没有

配网窗口（[第 5 章](05-provision-device-over-serial.zh-CN.md)）只在**刚上电**的时候打开，而且
只开一小段时间（静默 5 秒，或者总共 60 秒，哪个先到就按哪个算）——窗口关了之后再发指令，什么
响应都不会有。给设备重新上电或者复位，然后赶紧再发一次。

## 用模拟器的时候连接被拒绝

确认 `ubibot-server` 真的在运行，并且监听在你传给 `ub_device_sim --host`/`--port` 的那个地址上
——见[第 7 章](07-no-hardware-path.zh-CN.md)。如果你以为设备会用预置好的那个演示 `sn`，结果在
列表里找不到，通常是 `--sn` 打错了，不是连接问题——去设备列表看看实际到达的是哪个 `sn`。

## 去哪儿找日志

| 哪一部分 | 在哪儿看 |
|---|---|
| 后台 | 跑 `ubibot-server`（或 `go run ./cmd/server`）的那个终端 |
| 管理控制台 | 浏览器的开发者工具（网络面板看失败的 API 请求） |
| 真实 WS1B 固件 | `idf.py monitor`，或者 ubibot-serial-sync 的日志面板 |
| 模拟器 | 它自己的终端——直接打印出来，没有单独的日志查看工具 |

## 还是没解决？

完整的错误码表在协议文档的
[§8 错误处理](../protocol/hardware-communication-protocol.zh-CN.md#8-错误处理)里；admin API
的错误响应格式（包括管理控制台用来映射成翻译文案的那些稳定 `code` 字符串）在
[api/admin-api.md](../api/admin-api.zh-CN.md)里。如果这两份都没解释清楚你遇到的问题，去对应的仓库
开一个 issue。

---

用户手册到这里就结束了。如果你准备好开始修改平台的代码，而不只是使用它，请看
《[开发者手册](../dev-guide/README.zh-CN.md)》。
