# UbiBot Open Doc

UbiBot Open 项目（`ubibot-open` 组织）的配置与技术文档仓库，收录跨仓库、不适合放在单个代码仓库里的文档：设备通信协议、系统部署/烧录/联调指南等。

## 目录

- [硬件通信协议](protocol/UbiBot开放平台硬件通信协议.md) — 设备↔服务端 HTTP 协议的权威定义，含蓝牙配网、时间同步、数据上传、错误码等章节。
- [系统部署烧录联调指南](guides/系统部署烧录联调指南.md) — 从零跑通"后端部署 → 固件烧录 → 串口调试 → 设备联网上报 → 后台可见"全链路的操作手册。

## 相关仓库

| 仓库 | 说明 |
|---|---|
| [ubibot-open-server](https://github.com/ubibot-open/ubibot-open-server) | 设备接入后端 + 管理后台（Go + React），单一可执行文件部署 |
| [ubibot-open-ws1b](https://github.com/ubibot-open/ubibot-ws1b) | WS1B 设备开源参考固件（ESP-IDF，ESP32-C5） |
| [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync) | 跨平台桌面端串口调试工具 |


## 快速开始

跟着下面 4 步，从零跑通"服务器起来 → 硬件烧录 → 联网上报 → 后台看到数据"的完整链路。
每一步的详细说明、可选参数和踩坑排查见 [系统部署烧录联调指南](guides/系统部署烧录联调指南.md)；
手头没有真实设备也可以先跳到第 4 步前的"没有硬件？"小节，用内置仿真器走通 1、4 两步。

### 1. 部署服务器

```bash
git clone https://github.com/ubibot-open/ubibot-open-server.git ubibot-open-server
cd ubibot-open-server
./build.sh        # Windows 用 .\build.ps1；需要 Go 1.23+ 和 Node.js/npm
./ubibot-server    # 默认监听 :8080
```

浏览器打开 `http://localhost:8080`。首次启动会在日志里打印一次性生成的 `admin` 密码
（例如 `no admin account found — created "admin" with a generated password: ...`），
记得当场记下来，只显示这一次。详见指南 §2。

### 2. 编译烧录硬件

```bash
git clone https://github.com/ubibot-open/ubibot-ws1b.git ubibot-open-ws1b
cd ubibot-open-ws1b
idf.py set-target esp32c5
idf.py menuconfig      # 见下一步"配置硬件"，改完保存退出再继续
idf.py build
idf.py -p <串口号> flash monitor
```

需要预先装好 **ESP-IDF v6.0.2**（目标芯片 **ESP32-C5**）并激活好环境变量
（`export.sh`/`export.ps1`）。`<串口号>` 例如 Windows 的 `COM5`、Linux 的 `/dev/ttyUSB0`。

### 3. 配置硬件

上一步 `idf.py menuconfig` 里进入 `UbiBot WS1B Configuration` 菜单，至少要改这 3 项再保存：

| 配置项 | 改成什么 |
|---|---|
| WiFi SSID / Password | 现场实际要连的 WiFi |
| Data server host / port | 第 1 步部署的服务器地址，默认端口 `8080` |
| Device serial number (SN) | 每台设备唯一，不要和其他设备重复 |

完整配置项（含国家码、产品型号等）见指南 §3.2。改完回到第 2 步继续 `build`/`flash`。

### 4. 查看数据

设备联网上报成功后，回到第 1 步打开的管理后台，设备列表里会自动出现刚才配置的 SN——
不需要在后台预先创建设备；点进去能看到最新一条数据，"数据仓库"页可以看所有设备的最新记录。

没看到设备？按指南的[端到端验证](guides/系统部署烧录联调指南.md#5-端到端验证设备联网并上报数据)
三步排查（WiFi 是否连上 → 上报是否成功 → 后台是否可见）。

> **没有硬件？** `ubibot-open-server/simulation` 目录自带一个纯 C 编写的设备仿真程序，
> 协议行为和真实固件完全一致，可以跳过第 2、3 步直接验证服务器和后台，见指南 §6。

