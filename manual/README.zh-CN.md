# UbiBot Open — 用户手册

*[English](README.md)*

从零开始的教程式指南，每一步都配截图。如果你已经熟悉命令行、只想要命令本身，《[部署、烧录与上电指南](../guides/deployment-flashing-guide.md)》是覆盖同样内容的更快、更精炼的速查文档。这份手册是给第一次接触的人看的。

> **状态：** 所有章节都已经写完。其余章节暂时还是占位内容——正在按章节逐个撰写，具体进度见各章节页面。

## Part A · 从零搭建

按顺序阅读。手头没有 WS1B 硬件？可以从第 3 章直接跳到第 7 章——用模拟器可以达到同样的最终状态。

1. [手册说明](00-overview.zh-CN.md)
2. [准备工作](01-prerequisites.zh-CN.md)
3. [部署后台服务](02-deploy-server.zh-CN.md)
4. [首次登录与导航](03-first-login-and-tour.zh-CN.md)
5. [烧录 WS1B 参考固件](04-flash-firmware.zh-CN.md) —— 有真实硬件
6. [通过串口配置设备](05-provision-device-over-serial.zh-CN.md) —— 有真实硬件
7. [验证设备上线](06-verify-device-online.zh-CN.md) —— 有真实硬件
8. [没有硬件？用模拟器](07-no-hardware-path.zh-CN.md) —— 没有硬件

## Part B · 日常使用

平台搭起来、至少有一台设备（真实或模拟）在上报之后的日常操作。

9. [管理设备与产品](08-managing-devices-and-products.zh-CN.md)
10. [给设备下发指令](09-sending-commands.zh-CN.md)
11. [告警](10-alerts.zh-CN.md)
12. [用户、角色与 API 密钥](11-users-roles-and-api-keys.zh-CN.md)
13. [系统设置与监控](12-system-settings-and-monitor.zh-CN.md)
14. [常见问题排查](13-troubleshooting.zh-CN.md)

## 截图

每一章的图片都放在 [`assets/`](assets/) 目录下，命名为 `<章节slug>-01.png`、`<章节slug>-02.png`……按页面出现顺序编号。除非截图本身就是在展示界面文字，否则中英文两个版本共用同一批截图。

## 另见

- [开发者手册](../dev-guide/README.zh-CN.md) —— 面向修改平台代码，而不只是使用平台。
- [架构总览](../architecture/overview.md)、[硬件通信协议](../protocol/hardware-communication-protocol.md)、[管理后台 API 参考](../api/admin-api.md)、[开放 API 参考](../api/open-api.md)（暂无中文版）。
