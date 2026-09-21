# UbiBot Open Doc

*[English](README.md)*

UbiBot Open 项目（`ubibot-open` 组织）的配置与技术文档仓库：存放跨多个仓库、不属于任何单一代码仓库的文档——设备通信协议、系统部署/烧录/上电指南等等。

## 目录

- [架构总览](architecture/overview.md) —— 五个仓库是怎么拼起来的：系统图、数据流、后端/固件内部目录结构，以及这套代码依赖的设计原则。
- [硬件通信协议](protocol/hardware-communication-protocol.md) —— 设备↔服务端 HTTP 协议的权威定义：配网、时间同步、数据上报、错误码等等。
- [系统部署、烧录与上电指南](guides/deployment-flashing-guide.md) —— 从零开始到"后端部署完成 → 固件烧录完成 → 串口调试 → 设备上线并上报 → 仪表盘可见"的实操参考。
- [用户手册](manual/README.zh-CN.md)（[English](manual/README.md)）—— 和部署指南走的是同一条路径，但是教程式的、配截图，外加日常使用管理控制台的内容（设备、产品、指令、告警、用户、API 密钥）。
- [开发者手册](dev-guide/README.zh-CN.md)（[English](dev-guide/README.md)）—— 任务导向的教程，讲怎么修改平台代码：加一个 API 端点、加一个数据模型、加一个后台页面，或者加一个新的固件指令。
- [管理后台 API 参考](api/admin-api.md) / [开放 API 参考](api/open-api.md) —— 后端暴露的每一个 HTTP 路由，分别对应管理控制台自己用的接口和面向第三方的只读接口（暂无中文版）。

这个仓库大部分内容只有英文版；这个页面、**用户手册**和**开发者手册**是例外，都在英文原文旁边配了
中文翻译——每个有翻译的文件顶部都有一行语言切换链接（`*[中文](...)*` / `*[English](...)*`），指向
对应的另一个语言版本。

## 相关仓库

| 仓库 | 说明 |
|---|---|
| [ubibot-open-server](https://github.com/ubibot-open/ubibot-open-server) | 面向设备的后端 + 管理控制台（Go + React），打包成单个可执行文件部署 |
| [ubibot-open-ws1b](https://github.com/ubibot-open/ubibot-ws1b) | WS1B 设备的开源参考固件（ESP-IDF，ESP32-C5） |
| [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync) | 跨平台桌面串口调试工具 |
| [ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator) | 纯 C、可在主机上构建的设备模拟器，协议跟真实固件完全一致——不需要硬件 |


## 快速开始

按下面 4 步，从零走到"服务端跑起来 → 硬件烧录完成 → 通过网络上报 → 数据在仪表盘可见"。每一步的
详细说明、可选参数和故障排查见
[部署、烧录与上电指南](guides/deployment-flashing-guide.md)——或者看
[用户手册](manual/README.zh-CN.md)，同样的步骤但配了截图。手头没有硬件？直接跳到第 4 步前面的
"没有硬件？"——用[设备模拟器](https://github.com/ubibot-open/ubibot-open-simulator)自己就能走完
第 1 步和第 4 步。

### 1. 部署服务端

```bash
git clone https://github.com/ubibot-open/ubibot-open-server.git ubibot-open-server
cd ubibot-open-server
./build.sh        # Windows 用 .\build.ps1；需要 Go 1.23+ 和 Node.js/npm
./ubibot-server    # 默认监听 :8080
```

在浏览器里打开 `http://localhost:8080`。第一次运行时，日志会打印一个一次性生成的 `admin` 密码
（类似 `no admin account found — created "admin" with a generated password: ...`）——现在就把它
记下来，只会显示这一次。详见指南 §2。

### 2. 构建并烧录硬件

```bash
git clone https://github.com/ubibot-open/ubibot-ws1b.git ubibot-open-ws1b
cd ubibot-open-ws1b
idf.py set-target esp32c5
idf.py menuconfig      # 见下面"配置硬件"；保存退出后继续
idf.py build
idf.py -p <port> flash monitor
```

需要 **ESP-IDF v6.0.2**（目标芯片 **ESP32-C5**），并且已经激活其环境（`export.sh`/`export.ps1`）。
`<port>` 比如 Windows 上的 `COM5`，或者 Linux 上的 `/dev/ttyUSB0`。

### 3. 配置硬件

在上一步的 `idf.py menuconfig` 里，进入 `UbiBot WS1B Configuration` 菜单，保存之前至少设置好
这 3 项：

| 设置项 | 该填什么 |
|---|---|
| WiFi SSID / 密码 | 现场实际要连接的那个 WiFi |
| 服务器地址 / 端口 | 第 1 步里的服务器地址，默认端口 `8080` |
| 设备序列号（SN） | 每台设备唯一——不同设备之间不要重复 |

完整的设置项列表（国家代码、产品 ID 等）见指南 §3.2。设置完之后，回到第 2 步继续
`build`/`flash`。

### 4. 查看数据

设备上线并成功上报之后，回到第 1 步打开的管理控制台——你刚配置的那个 SN 会自动出现在设备列表里，
不需要提前创建设备。点进去看最新读数，或者去"数据仓库"页面看每台设备的最新记录。

没看到设备？照着指南的
[端到端验证](guides/deployment-flashing-guide.md#5-end-to-end-verification-device-online-and-reporting)
走一遍——三个核对点（WiFi 已连接 → 上报成功 → 仪表盘可见）。

> **没有硬件？** [ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator)
> 是一个纯 C 的设备模拟器，协议行为跟真实固件完全一致，让你不用走第 2-3 步就能验证服务端和仪表盘。
> 详见指南 §6。

## 贡献

欢迎修复和补充文档——见
[组织级 CONTRIBUTING.md](https://github.com/ubibot-open/.github/blob/main/CONTRIBUTING.md)
（暂无中文版）。

## License

[Apache License 2.0](LICENSE)。
