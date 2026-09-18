# 没有硬件？用模拟器

*[English](07-no-hardware-path.md)*

这一章会完全替代第 4-6 章。[ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator)
是一个体积很小的 C 程序，说的是跟真实 WS1B 设备完全一样的 HTTP 协议——不需要 WiFi，不需要串口，
不需要任何实体硬件——只用你的笔记本电脑就能达到同样的最终状态（设备上线并上报）。

## 1. 构建它

```bash
git clone https://github.com/ubibot-open/ubibot-open-simulator.git
cd ubibot-open-simulator
cmake -S . -B build
cmake --build build
```

Windows 上用 MinGW 的话，把上面的 `cmake -S . -B build` 换成
`cmake -S . -B build -G "MinGW Makefiles"`。

**截图占位符：** `07-no-hardware-path-01.png` —— 构建成功的终端输出。

## 2. 对着你的后台跑起来

保持[第 2 章](02-deploy-server.zh-CN.md)里的 `ubibot-server` 一直在运行：

```bash
./build/ub_device_sim --host 127.0.0.1 --port 8080
```

除了 `--host`/`--port` 不带其他参数的话，它会用 `sn_ws1_20001_1` 这个序列号来标识自己——跟后台
第一次运行时自动创建的那台演示设备（见[第 2 章](02-deploy-server.zh-CN.md)）序列号完全一样，所以
不用做任何配网就能"直接用"。之后它会一直循环：采样三个传感器（`field1` 温度、`field2` 湿度、
`field3` 光照——都是随机游走生成的假数据，不是真实读数）、上报、休眠、再来一遍。按
`Ctrl+C` 停止。

**截图占位符：** `07-no-hardware-path-02.png` —— 模拟器的控制台输出，显示成功上报。

> **注意：** 想自己亲眼看看设备自动注册是怎么发生的，而不是用预置好的那台演示设备？可以传一个
> 你自己的 `--sn something-else`——协议没有预先注册这一步，所以只要这台模拟设备第一次上报一到，
> 服务器就会当场建一条全新的设备记录。

## 3. 用同样的三个核对点验证

这一步就是[第 6 章](06-verify-device-online.zh-CN.md)那三个核对点的等价版，只是不用管 WiFi
那一步：

1. **模拟器自己的控制台**会显示它发了一次上报，收到了 `{"c":0,...}` 的响应——跟真实硬件不同，
   不需要另外开一个日志查看工具。
2. **后台的日志**（跑 `ubibot-server` 的那个终端）没有报什么异常——如果上报被拒绝了，错误会在
   这里出现。
3. **管理控制台的设备管理 → 设备**页面会显示 `sn_ws1_20001_1`（或者你传的那个 `--sn`）在线，
   **数据仓库 → 我的数据**里能看到一条新的读数。

**截图占位符：** `07-no-hardware-path-03.png` —— 设备列表显示这台模拟设备在线。

## 下一步

你现在已经走到了跟真实硬件一样的地方。继续进入 Part B：
[管理设备与产品](08-managing-devices-and-products.zh-CN.md)。
