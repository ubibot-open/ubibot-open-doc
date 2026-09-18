# 验证设备上线

*[English](06-verify-device-online.md)*

三个核对点，按顺序来——哪一步不对，先解决它，再往下走。

## 1. 设备侧：连上了 WiFi

在还开着的 `idf.py monitor`（从[第 4 章](04-flash-firmware.zh-CN.md)延续下来）或者
`ubibot-serial-sync` 的日志面板里，找一行类似"WiFi connected"、后面跟着设备拿到了一个 IP 地址的
日志。

**截图占位符：** `06-verify-device-online-01.png` —— 串口日志显示 WiFi 连接成功。

**没看到？** 先检查你在[第 5 章](05-provision-device-over-serial.zh-CN.md)里配置的 WiFi
SSID/密码/国家代码是不是对的，以及路由器/接入点是不是真的能连上、正在广播。还是不行的话看
[第 13 章](13-troubleshooting.zh-CN.md)。

## 2. 网络与协议：上报被接受

还是同一份日志，找一次 `POST /api/v1/data/report` 调用，后面跟着一个 `{"c":0,...}` 的响应。

**截图占位符：** `06-verify-device-online-02.png` —— 日志里显示上报请求和它的
`{"c":0,...}` 响应。

**卡住没响应或者超时？** 确认[第 2 章](02-deploy-server.zh-CN.md)里的后台服务还在跑，你配置的
服务器地址/端口真的指向它运行的那台机器（局域网 IP，设备能访问到——不是 `localhost`），并且那台
机器上的防火墙没有挡住这个端口。

## 3. 在仪表盘上能看到

回到管理控制台（`http://<后台地址>:8080`），打开**设备管理 → 设备**。你配置的那个序列号应该已经
出现在列表里了——协议规定设备第一次成功上报的时候就会自动注册，不需要在控制台里预先创建。

**截图占位符：** `06-verify-device-online-03.png` —— 设备列表，新设备出现在里面，标记为在线。

点进去看看，或者去**数据仓库 → 我的数据**——它最新的读数应该也能在那里看到。

**截图占位符：** `06-verify-device-online-04.png` —— 设备的详情页，或者"我的数据"里它那一行，
显示着一个真实的读数。

如果这三点都核对通过了，整条链路——设备上线→上报→仪表盘可见——就算端到端跑通了。接下来进入
Part B，从《[管理设备与产品](08-managing-devices-and-products.zh-CN.md)》开始了解控制台剩下的
功能。

## 下一步

[管理设备与产品](08-managing-devices-and-products.zh-CN.md)。
