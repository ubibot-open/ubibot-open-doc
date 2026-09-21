# UbiBot Open —— 系统部署、烧录与上电指南

*[English](deployment-flashing-guide.md)*

这份文档面向任何想把三个仓库——`ubibot-open-server`、`ubibot-open-ws1b`、`ubibot-serial-sync`——
接到一起、从零走到一条完整的"设备上线 → 上报数据 → 仪表盘可见"链路的人。设备↔服务器的协议细节见
[硬件通信协议](../protocol/hardware-communication-protocol.zh-CN.md)；这份文档只讲怎么把
每个组件跑起来、以及怎么把它们接在一起。

> 仓库命名说明：下面命令里用的仓库名，反映的是本组织（`ubibot-open`）目前实际的 GitHub 地址
> （核心平台历史上曾叫 `ubibot-platform-open`，固件曾叫 `ubibot-ws1b`）。如果你这边已经把它们
> 改名成了 `ubibot-open-server` / `ubibot-open-ws1b`，直接从新的仓库地址拉取就行——不影响后面
> 的步骤。

## 0. 系统总览

```
┌────────────────────┐   USB serial                                  ┌────────────────────┐
│ ubibot-serial-sync  │ ───────────────────────────────────────────▶ │  ubibot-open-ws1b   │
│ Desktop serial      │  Provisioning: SetupWifi / SetupServer       │  firmware device     │
│ debug/provisioning  │  (protocol §1.2)                             │  (logs / debugging)  │
│                     │◀─────────────────────────────────────────── │                      │
└────────────────────┘                                               └──────────┬──────────┘
                                                                                 │ HTTP (protocol §2–§8)
                                                                                 │ POST /api/v1/auth/time
                                                                                 │ POST /api/v1/data/report
                                                                                 ▼
                                                                     ┌────────────────────────────┐
                                                                     │     ubibot-open-server      │
                                                                     │ Go backend + embedded React │
                                                                     │ admin console; single binary,│
                                                                     │ SQLite storage               │
                                                                     └─────────────┬──────────────┘
                                                                                   ▲
                                                                                   │ Browser http://<host>:8080
                                                                                   │
                                                                          ┌────────┴────────┐
                                                                          │  Admin's browser │
                                                                          └─────────────────┘
```

这三个组件是独立部署的，只通过网络（设备 → 后端，走 HTTP）和一根物理数据线（电脑 → 设备，走
USB 串口——用于 §1.2 的配网指令和日志调试）交互；没有共享代码或共享配置文件。这个协议没有蓝牙
配网的计划（§1.1 明确标注为不支持）——现场的设备配网只走串口这一条路。

## 1. 准备工作

| 组件 | 用途 | 要求 |
|---|---|---|
| `ubibot-open-server` | 面向设备的后端 + 管理控制台，打包成单个可执行文件 | Go 1.23+，Node.js/npm（构建管理前端需要） |
| `ubibot-open-ws1b` | WS1B 参考固件（目标芯片 ESP32-C5） | ESP-IDF v6.0.2 |
| `ubibot-serial-sync` | 桌面串口调试工具 | 下载官方预编译包——**不需要自己构建**；要从源码构建见该仓库的 `BUILD.md`（Qt 6 / CMake / C++17） |

## 2. 部署并启动后端（ubibot-open-server）

### 2.1 拉取源码

```bash
git clone https://github.com/ubibot-open/ubibot-open-server.git ubibot-open-server
cd ubibot-open-server
```

### 2.2 一条命令构建

```bash
./build.sh        # Linux / macOS
# .\build.ps1      # Windows
```

构建脚本做两件事：先构建 `admin/` 下的 React 管理控制台，把产物内嵌进
`server/internal/webui/dist`；然后构建 Go 后端，在仓库根目录产出一个单独的可执行文件
`ubibot-server`（`CGO_ENABLED=0`，纯 Go 的 SQLite 驱动，静态链接——不需要额外装运行时或数据库
服务）。

### 2.3 运行

```bash
./ubibot-server
```

默认监听 `:8080`；数据库文件默认是 `./data/ubibot.db`（目录和文件首次运行时自动创建）。可以用
下面这些环境变量覆盖：

| 环境变量 | 默认值 | 说明 |
|---|---|---|
| `UBIBOT_LISTEN_ADDR` | `:8080` | 监听地址 |
| `UBIBOT_DB_PATH` | `./data/ubibot.db` | SQLite 数据库文件路径 |
| `UBIBOT_ADMIN_PASSWORD` | （不设置就随机生成） | 只在数据库里还一个管理员账号都没有时（也就是首次运行）才生效；用户名固定是 `admin` |

首次运行时，如果没有提前设置 `UBIBOT_ADMIN_PASSWORD`，日志会打印类似这样一行：

```
no admin account found — created "admin" with a generated password: 3f9a1b2c4d5e6f
```

**这个密码只会打印这一次**——立刻记下来。或者在第一次运行之前就设置好
`UBIBOT_ADMIN_PASSWORD=<你的密码>`，跳过随机密码这一步。

首次运行还会预置一台演示设备（`pid=ubibot_open_dev_v1`，`sn=sn_ws1_20001_1`），你可以立刻在
管理控制台的设备列表里看到它——不用等真实硬件上线，就能确认后端本身是好的。

### 2.4 登录管理控制台

在浏览器里打开 `http://localhost:8080`（API 和管理控制台是同一个进程、同一个端口提供的），用
上一步的 `admin` 账号/密码登录。

### 2.5 给设备下发指令（可选）

一旦某台设备至少成功上报过一次、出现在设备列表里，打开它的详情页——**设备指令**卡片能让你排一个
**重启**或者一个新的**上报间隔**（60–86400 秒）。这就是协议 §9 的指令通道：你排的东西会随这台
设备*下一次*上报捎带送达，然后被清空——没有应答，所以平台没法确认设备到底有没有真的收到或执行。
不放心的话，再排一次就是了。

### 2.6 产品与批量设备管理（可选）

设备数量一多，这几个功能会有用：

- **产品**（设备管理 → 产品）——给一个 `pid` 注册一次展示名称和描述，之后所有上报这个 `pid` 的
  设备，在设备列表/详情里显示的就是解析出来的产品名，而不是一个裸的 `pid`。纯粹是展示元数据：
  创建、重命名或删除一个产品，从来不会动任何设备记录。
- **批量导入**（设备 → 批量导入）——粘贴或上传一份 `sn,pid,name` 格式的 CSV（name 可选），提前
  批量注册一批产线设备，即使它们还一次都没上报过。同样纯粹是为了方便：从没导入过的设备，一旦
  自己第一次上报，照样会自动出现。
- **导出**（设备 → 导出 CSV）——把整个设备群（不只是当前页）下载成 CSV。

## 3. 烧录 WS1B 参考固件（ubibot-open-ws1b）

### 3.1 拉取源码并安装 ESP-IDF

```bash
git clone https://github.com/ubibot-open/ubibot-ws1b.git ubibot-open-ws1b
cd ubibot-open-ws1b
```

需要 **ESP-IDF v6.0.2**（目标芯片 **ESP32-C5**）。按 Espressif 官方文档安装好，然后每开一个新
终端会话都要先激活它的环境（`. ./export.sh`，Windows 上是 `export.ps1`，或者 ESP-IDF 安装时带的
"Command Prompt"/"PowerShell" 快捷方式），之后才能用 `idf.py` 命令。

### 3.2 配置设备参数

```bash
idf.py set-target esp32c5   # 只有第一次构建时需要；build/ 已经存在的话可以跳过
idf.py menuconfig
```

进入 `UbiBot WS1B Configuration` 菜单，按需设置：

| 子菜单 | 设置项 | 默认值 | 说明 |
|---|---|---|---|
| WiFi Configuration | WiFi SSID | `TEST24` | 设备要连接的 WiFi 网络名 |
| | WiFi Password | （空） | 留空表示连接开放（无密码）网络 |
| | WiFi country code | `01` | 两字母监管国家代码（比如 `CN`/`US`），或者用 `01` 这个全球通用的安全默认值 |
| Server Configuration | Data server host | `192.168.2.71` | 第 2 步部署的后端的 IP/域名 |
| | Data server port | `8080` | 必须跟后端的 `UBIBOT_LISTEN_ADDR` 端口一致 |
| Device Identity | Device product ID | `ubibot-ws1b` | 同型号所有设备共用——整批产线设备保持默认值即可 |
| | Device serial number | `RV41554WS1B` | 每台物理设备必须唯一——但见下面的提示，不需要每次构建都改这个值来做到这一点 |

设置会保存到本地的 `sdkconfig` 文件（已加入 gitignore，不会被提交）；默认值定义在
`main/Kconfig.projbuild` 里。

> **这些只是出厂默认值**：本节的 menuconfig 设置在构建时固化进 `sdkconfig`，只是设备第一次
> 烧录时的 WiFi/服务器/序列号默认值。烧录之后，还可以在**运行时**通过串口，用
> [硬件通信协议](../protocol/hardware-communication-protocol.zh-CN.md)§1.2 的
> `SetupWifi`/`SetupServer`/`SetupDevice` 指令覆盖它们，不需要重新编译或烧录——见第 4 节。
> 特别是，`SetupDevice` 只设置序列号，这意味着你可以用**一份构建**烧录整批产线设备（共用同一个
> `pid` 和同一个默认 `sn`），之后再通过串口给每台物理设备设置它自己真正的 `sn`，而不用每台设备
> 都重新编译一次。蓝牙配网（协议 §1.1）依然明确不支持；串口配网目前是唯一的运行时配网方式。

### 3.3 构建并烧录

```bash
idf.py build
idf.py -p <port> flash monitor
```

`<port>` 比如 Windows 上的 `COM5`，Linux 上的 `/dev/ttyUSB0`，或者 macOS 上的
`/dev/cu.usbserial-xxxx`。`flash` 完成后会自动进入 `monitor`（串口日志查看），这样你就能实时看到
设备开机、连上 WiFi、上报数据的过程。按 `Ctrl+]` 退出 monitor。

## 4. 查看日志、调试，并通过串口给设备配网（ubibot-serial-sync）

用途：不用装 ESP-IDF 就能查看设备的串口日志、手动收发数据（对已经在产线上烧录完、不方便再接上
ESP-IDF 环境的设备特别有用）——一个跨平台的图形界面，用起来比 `idf.py monitor` 更顺手。协议
§1.2 的串口配网指令（`SetupWifi`/`SetupServer`）也是通过这个工具的手动发送框来发的。

### 4.1 获取它

推荐直接下载预编译包——不需要 Qt 或构建工具：

| 平台 | 下载 |
|---|---|
| Windows | [UbiBotSerialAssistant-windows-x64.zip](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest/download/UbiBotSerialAssistant-windows-x64.zip) |
| macOS | [UbiBotSerialAssistant-macos.dmg](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest/download/UbiBotSerialAssistant-macos.dmg) |
| Linux | [UbiBotSerialAssistant-linux-x86_64.AppImage](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest/download/UbiBotSerialAssistant-linux-x86_64.AppImage)（`chmod +x` 之后直接运行） |

第一次运行会被操作系统标记为"未知发布者"（Windows SmartScreen / macOS Gatekeeper）——点击继续
（macOS 上是右键 → 打开），跟其他未签名的开源软件一样，这是预期内的。

### 4.2 连接设备

1. 打开这个应用，在"串口连接"面板里选中你设备所在的端口，波特率要跟固件的 UART 日志输出一致
   （ESP-IDF 默认 115200）。
2. 打开端口——右侧的数据监控面板会实时显示带颜色区分的 TX/RX/SYS/ERR 日志，跟 `idf.py monitor`
   类似，但不需要装 ESP-IDF。
3. 要手动发点什么，用手动发送框（支持 ASCII/HEX），或者从"设备指令库"面板里按型号选一个预设
   指令。

> 设备指令库目前还没有内置 WS1B 的预设指令集（这个库纯粹是靠 JSON 配置驱动的，加一个型号就是
> 加一个 JSON 文件，不需要改代码）——配网指令目前也先用手动发送框来发。

### 4.3 通过串口配网（不需要重新烧录）

设备只在**刚上电**的时候打开配网窗口（收到最后一个字节之后静默几秒就会提前关闭，另外不管有没有
活动都有一个硬性总时长上限——精确时长见
[main/provisioning.c](https://github.com/ubibot-open/ubibot-ws1b/blob/main/main/provisioning.c)）；
普通的定时器唤醒不会重新打开这个窗口。所以要**在设备刚上电或者刚复位之后**立刻通过串口连接并
发送：

```jsonc
// 修改 WiFi
{"command":"SetupWifi","ssid":"MyHomeWiFi","password":"12345678","type":"WPA2"}
// 修改服务器地址
{"command":"SetupServer","host":"192.168.2.71","port":8080}
// 设置这台设备自己的序列号——对应 §3.2 里的批量烧录流程：
// 一份构建烧录所有设备，之后再逐台设置各自的 sn
{"command":"SetupDevice","sn":"RV41554WS1B"}
```

处理完每条指令之后，设备会在数据监控面板里写回一行 JSON 应答：成功是
`{"c":0,"msg":"wifi saved"}` 这种，校验失败是 `{"c":1,"msg":"<原因>"}`，保存失败是
`{"c":2,"msg":"<原因>"}`。保存成功之后**立刻生效**——不需要重启——并且会写入能扛住断电的存储里，
所以之后每次开机都会用最新的配置，直到再次被覆盖。窗口关闭之后再发指令，完全不会有任何响应
（没有应答，也没有效果）；给设备重新上电或者复位，再试一次就好。

## 5. 端到端验证：设备上线并上报

按顺序核对下面几项；哪一项对不上，就去对应的章节排查：

1. **设备侧**：`idf.py monitor`（或者 serial-sync 的日志面板）打印出类似"WiFi connected"、
   拿到了一个 IP 地址的内容。
   → 没看到：检查第 3.2 步里的 WiFi SSID/密码/国家代码是不是对的，或者路由器/接入点能不能连上。
2. **网络与协议**：日志显示一次 `POST /api/v1/data/report` 调用，返回了 `{"c":0,...}`。
   → 卡住没响应或者超时：确认第 2 步的后端在运行，`Data server host/port` 真的指向后端监听的
   地址，运行后端那台机器的防火墙放行了那个端口。
3. **仪表盘可见**：回到管理控制台（`http://<后端地址>:8080`）的设备列表，你配置的那个 SN 自动
   出现了——协议规定设备第一次成功上报就会自动注册，不需要提前在控制台里创建。它最新的读数在
   "数据仓库"页面也能看到。

如果这三项都核对通过，"设备上线 → 上报 → 仪表盘可见"这条链路就算端到端跑通了。

## 6. 不用真实硬件测试（可选）

[ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator) 是一个独立仓库里的
纯 C 设备模拟器，协议行为跟真实固件完全一致——手头没有硬件，或者只是想快速验证一个后端/仪表盘
改动的时候会有用：

```bash
git clone https://github.com/ubibot-open/ubibot-open-simulator.git
cd ubibot-open-simulator
cmake -S . -B build
cmake --build build
./build/ub_device_sim --host 127.0.0.1 --port 8080
```

默认值跟后端首次运行时预置的演示设备（`sn_ws1_20001_1`）一致；用 `--pid`/`--sn` 可以换成别的
设备身份——协议不要求提前注册设备，服务器会在第一次上报时自动注册。`--tick-ms` 调整主循环的心跳
间隔（默认 1000ms）；`Ctrl+C` 停止模拟器。

## 7. 故障排查

| 症状 | 错误码 | 原因与解决办法 |
|---|---|---|
| 上报被拒绝，`c=1002` | 时间戳超出了 ±5 分钟的窗口 | 设备的本地时钟漂移了；确认它在收到这个错误之后会调用 `POST /api/v1/auth/time` 重新校准 |
| 上报被拒绝，`c=1003` | 请求体格式不对 | 对照协议 §5 检查固件的 JSON 序列化逻辑 |
| 上报被拒绝，`c=1103` | 设备被管理员停用了 | 在管理控制台的设备详情页重新启用它 |
| 上报被拒绝，`c=1900` | 被限速了 | 上报太频繁了；等下一个周期，或者检查设备是不是在异常重试 |
| 设备的日志始终到不了后端 | 没有响应/超时 | 确认 `Data server host/port` 是对的、后端进程在运行、运行后端那台机器的防火墙放行了 `UBIBOT_LISTEN_ADDR` 对应的端口，并且设备和后端在网络上互相能访问 |
| 忘记管理控制台密码 | — | 停掉 `ubibot-server` 进程，用 `sqlite3` 打开数据库，执行 `DELETE FROM admin_users;` 清空管理员表，然后重新启动服务——这会重新走一遍 §2.3 里"首次运行"的逻辑，创建一个全新的 `admin` 账号（设备数据不受影响） |
| 通过串口发了 `SetupWifi`/`SetupServer`，什么反应都没有 | — | 配网窗口只在**刚上电**的时候打开，而且只开一小段时间（见 §4.3）；窗口关闭之后再发不会有任何应答——给设备重新上电或者复位，然后赶紧再发一次 |

---

参考资料：
[硬件通信协议](../protocol/hardware-communication-protocol.zh-CN.md) ·
[ubibot-open-server](https://github.com/ubibot-open/ubibot-open-server) ·
[ubibot-open-ws1b](https://github.com/ubibot-open/ubibot-ws1b) ·
[ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync)
