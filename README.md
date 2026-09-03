# UbiBot Open Doc

UbiBot Open 项目（`ubibot-open` 组织）的配置与技术文档仓库，收录跨仓库、不适合放在单个代码仓库里的文档：设备通信协议、系统部署/烧录/联调指南等。

## 目录

- [硬件通信协议](protocol/UbiBot开放平台硬件通信协议.md) — 设备↔服务端 HTTP 协议的权威定义，含蓝牙配网、时间同步、数据上传、错误码等章节。
- [系统部署烧录联调指南](guides/系统部署烧录联调指南.md) — 从零跑通"后端部署 → 固件烧录 → 串口调试 → 设备联网上报 → 后台可见"全链路的操作手册。

## 相关仓库

| 仓库 | 说明 |
|---|---|
| [ubibot-open-server](https://github.com/ubibot-open/ubibot-platform-open) | 设备接入后端 + 管理后台（Go + React），单一可执行文件部署 |
| [ubibot-open-ws1b](https://github.com/ubibot-open/ubibot-ws1b) | WS1B 设备开源参考固件（ESP-IDF，ESP32-C5） |
| [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync) | 跨平台桌面端串口调试工具 |


## 快速开始

1. 部署服务器
2. 编译烧录硬件
3. 配置硬件
4. 查看数据

