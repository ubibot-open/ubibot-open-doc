# 烧录 WS1B 参考固件

*[English](04-flash-firmware.md)*

如果你没有真实的 WS1B 设备，可以跳过这一章（以及第 5 章）——直接去看
《[没有硬件？用模拟器](07-no-hardware-path.zh-CN.md)》。

## 1. 拉取代码

```bash
git clone https://github.com/ubibot-open/ubibot-ws1b.git ubibot-open-ws1b
cd ubibot-open-ws1b
```

如果还没装 ESP-IDF v6.0.2，见《[准备工作](01-prerequisites.zh-CN.md)》——按 Espressif 官方文档
装好，然后每开一个新终端都要先激活它的环境（`. ./export.sh`，Windows 上是 `export.ps1`），之后
才能用 `idf.py` 命令。

## 2. 设置目标芯片

```bash
idf.py set-target esp32c5
```

只需要在第一次构建时做（如果 `build/` 目录已经存在，说明之前构建过，可以跳过）。

## 3. 配置设备参数

```bash
idf.py menuconfig
```

**截图占位符：** `04-flash-firmware-01.png` —— `idf.py menuconfig` 的终端界面，打开着
`UbiBot WS1B Configuration` 这个菜单。

进入 `UbiBot WS1B Configuration` 菜单，设置：

| 子菜单 | 设置项 | 默认值 | 这里该填什么 |
|---|---|---|---|
| WiFi Configuration | WiFi SSID | `TEST24` | 设备要连接的 WiFi 网络名 |
| | WiFi Password | （空） | 连开放网络（无密码）留空即可 |
| | WiFi country code | `01` | 你所在地区的两字母监管国家代码（比如 `US`/`CN`），或者用 `01` 这个全球通用的安全默认值 |
| Server Configuration | Data server host | `192.168.2.71` | [第 2 章](02-deploy-server.zh-CN.md)里后台服务的地址——是 WS1B 所在局域网能访问到的地址，不是 `localhost` |
| | Data server port | `8080` | 如果你改过 `UBIBOT_LISTEN_ADDR` 的端口，这里要跟它一致 |
| Device Identity | Device product ID | `ubibot-ws1b` | 同一型号的所有设备共用，保持默认即可 |
| | Device serial number | `RV41554WS1B` | 不需要每次构建都改成唯一值——见下面的提示 |

> **注意：** 这些 menuconfig 的值只是设置了**出厂默认值**——在构建时固化进固件，设备第一次开机
> 用的就是这些。之后所有这些值都可以在不重新构建/烧录的情况下，通过串口在运行时修改——这就是
> [第 5 章](05-provision-device-over-serial.zh-CN.md)的内容。特别是，因为序列号可以之后逐台通过
> 串口单独设置，你可以用同一份构建烧录整批设备，之后再给每一台设置它自己真正的序列号。

设置会保存到本地的 `sdkconfig` 文件（不会被提交到 git）。

## 4. 构建并烧录

```bash
idf.py build
idf.py -p <port> flash monitor
```

`<port>` 是设备的串口——比如 Windows 上的 `COM5`，Linux 上的 `/dev/ttyUSB0`，或者 macOS 上的
`/dev/cu.usbserial-xxxx`。`flash` 会写入固件，然后自动进入 `monitor`（实时串口日志查看），这样
你就能实时看到设备开机、连上 WiFi、开始上报数据的过程。

**截图占位符：** `04-flash-firmware-02.png` —— `idf.py flash` 终端输出成功结束，进入
`monitor`。

**截图占位符：** `04-flash-firmware-03.png` —— `monitor` 里的开机日志，显示设备连上 WiFi 并发出
第一次上报。

看完之后按 `Ctrl+]` 退出 `monitor`。

## 下一步

[通过串口配置设备](05-provision-device-over-serial.zh-CN.md)。
