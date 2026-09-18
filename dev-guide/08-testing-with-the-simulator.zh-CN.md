# 用模拟器迭代后端

*[English](08-testing-with-the-simulator.md)*

烧录真实 WS1B 硬件来测一个后端改动，意味着一套"构建-烧录-等待-盯着串口日志"的流程，按分钟计。
[ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator) 把这个流程压缩到
秒级——它就是一个跟后端跑在同一台机器上的普通进程，重启它、或者同时跑好几个，都几乎不花代价。

## 基本循环

```bash
# 终端 1：后端，用 go run 实现热重载
cd ubibot-open-server/server && go run ./cmd/server

# 终端 2：一台模拟设备
cd ubibot-open-simulator && ./build/ub_device_sim --host 127.0.0.1 --port 8080
```

改后端代码需要在终端 1 里 `Ctrl+C` 然后重新跑一次 `go run ./cmd/server`；终端 2 里的模拟器完全
不用动——除非你改动的是*协议本身*（请求/响应的格式），而不是纯后端逻辑（新的 admin 端点、一个
store 查询、一个数据库列）。模拟器在后端重启期间会一直继续上报，所以下一次上报自然就会打到你
改过的新代码上，设备那一侧什么都不用做。

## 同时模拟多台设备

多开几个实例，每个用自己的 `--sn`（如果想让它们共用一个 Product，就用同一个 `--pid`）：

```bash
./build/ub_device_sim --host 127.0.0.1 --port 8080 --sn test-a &
./build/ub_device_sim --host 127.0.0.1 --port 8080 --sn test-b &
./build/ub_device_sim --host 127.0.0.1 --port 8080 --sn test-c &
```

这是获得一份看起来真实的设备列表最快的办法，用来测任何跟列表/分页/CSV 相关的东西——
[批量导入导出](../manual/08-managing-devices-and-products.zh-CN.md)、跨多台设备的
[Product](02-backend-add-admin-endpoint.zh-CN.md) 解析、仪表盘的在线数量统计等等——不用等真实
硬件，也不用手动往 SQLite 文件里插数据。

## 关于采样/上报的时间间隔

每台模拟设备每 30 秒采样一次、每 300 秒上报一次
（`app/src/ub_device.c` 里的 `UB_DEFAULT_SAMPLE_INTERVAL_SEC`/`UB_DEFAULT_REPORT_INTERVAL_SEC`）
——这是真实流逝的时间，不是模拟时间，也不受 `--tick-ms` 影响（那个参数只控制主循环*多久检查
一次*是不是到了该采样/上报的时间点，也就是反应有多及时——调小它不会让上报变快）。如果你测的东西
恰好需要更快的上报节奏（比如后端的离线检测扫描，或者想看
[服务端下发指令](07-firmware-add-server-command.zh-CN.md)被送达而不想等五分钟），可以临时把
`UB_DEFAULT_REPORT_INTERVAL_SEC` 调小然后重新构建——别把这个改动提交上去，它只是为了你本地这次
测试。

## 手动做协议层面的测试

有两个工具专门用来直接摸协议本身，都不参与正常构建：

- **`test/mock_server.py`** —— 一个极简的 Python HTTP 服务器，只实现了那两个设备接口，用来在
  完全不涉及真实后端的情况下测试*模拟器自己*的行为（失败重试、缓冲区处理）。
- **`test/test_integration.c`** —— 用模拟器自己的 C 协议编解码代码，通过一个真实的 socket 连接
  跑对着一个真的在运行的 `ubibot-open-server`（先 `go run ./cmd/server`，然后跑
  `./build/test_integration [host] [port]`）——这是纯单元测试没有真实 TCP 连接就够不到的东西。
  它没有接进 `ctest`/`make test`，因为需要那个真的在跑的后端；要手动运行它。

## 模拟器帮不上忙的地方

它没有实现[串口配网指令](06-firmware-add-serial-command.zh-CN.md)（根本没有串口这回事），也
没有实现[服务端下发指令通道](07-firmware-add-server-command.zh-CN.md)（`ub_device.c` 里没有
`Command_HandleResponse` 的对应实现）——这两个要端到端测试，还是得靠真实固件跑在真实硬件上。它
只是系统里"设备到后端"这一半的替身，不是固件本身的替身。

## 下一步

[修改通信协议](09-protocol-changes.zh-CN.md)。
