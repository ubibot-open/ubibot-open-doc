# 固件：新增串口配网指令

*[English](06-firmware-add-serial-command.md)*

> **状态：** 仅目录大纲，正文尚未撰写。

照着 provisioning.c 里 SetupWifi/SetupServer/SetupDevice 现有的模式，新增一个 SetupXxx 指令：解析、NVS 持久化，以及应答格式。

<!-- TODO: 撰写本章内容 -->
