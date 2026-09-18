# 准备工作

*[English](01-prerequisites.md)*

## 所有人都需要的

不管你走"有硬件"还是"没硬件"哪条路径，下面这些都需要装，因为它们是用来构建和运行后台服务和
管理控制台的：

| 工具 | 版本 | 用途 |
|---|---|---|
| [Go](https://go.dev/dl/) | 1.23 或更新 | 构建后台服务（`ubibot-open-server`） |
| [Node.js](https://nodejs.org/) | 18 或更新（任意当前 LTS 版本），自带 npm | 构建管理控制台前端 |
| [Git](https://git-scm.com/) | 任意较新版本 | 拉取仓库代码 |

> **注意：** 后台服务会编译成一个单独的、自包含的可执行文件，管理控制台构建好的前端会内嵌在
> 这个文件里面（见 [build.sh](https://github.com/ubibot-open/ubibot-open-server/blob/main/build.sh)）。
> 只有在你**构建**它的那台机器上才需要装 Go 和 Node.js——运行构建好的可执行文件两者都不需要。

## 如果你有真实的 WS1B 硬件

你还需要额外准备：

| 工具/物品 | 说明 |
|---|---|
| 一台 UbiBot WS1B 设备 | 或任何基于 ESP32-C5、运行本项目参考固件的板子 |
| 一根 USB 数据线 | 要能传输数据，不能是只能充电的线——用来把设备连到电脑上做烧录和串口配网 |
| [ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c5/get-started/index.html) v6.0.2 | Espressif 官方的固件构建/烧录工具链。按官方针对你操作系统的安装指南装好即可——它同时会装好本手册要用的 `idf.py` 命令行工具 |
| USB 转串口驱动 | 大多数 WS1B 板子用的是 CH340 或 CP210x 这类 USB 转串口桥接芯片；Windows/macOS 第一次插上设备时可能需要装一下驱动。[ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync) 自带的首次运行检测会告诉你是否缺驱动 |
| [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest) | 用来查看日志、通过串口给设备配网的桌面工具。下载对应你操作系统的预编译包即可——不需要自己构建 |

[第 4 章](04-flash-firmware.zh-CN.md)和[第 5 章](05-provision-device-over-serial.zh-CN.md)会讲怎么
安装和使用这些工具。

## 如果你没有硬件

上面那一段可以跳过——[第 7 章](07-no-hardware-path.zh-CN.md)只需要一个 C 编译器
（gcc/clang/MSVC，你电脑上通常已经有的那种）和 [CMake](https://cmake.org/download/) 3.10+，用来
构建 [ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator)。

## 下一步

[部署后台服务](02-deploy-server.zh-CN.md)。
