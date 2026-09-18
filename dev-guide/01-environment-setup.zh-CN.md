# 开发环境搭建

*[English](01-environment-setup.md)*

每个仓库一节——你实际要改哪个仓库，就搭哪个的环境。下面列的命令跟各仓库自己 CI 里跑的是同一套，
所以本地能跑通，CI 大概率也能过。

## ubibot-open-server（Go 后台 + React 管理控制台）

需要 **Go 1.23+** 和 **Node.js 18+**。

```bash
git clone https://github.com/ubibot-open/ubibot-open-server.git
cd ubibot-open-server
```

后台服务——构建、静态检查、测试：

```bash
cd server
go build ./...
go vet ./...
go test ./...
```

管理控制台——安装依赖、类型检查、构建（没有单独的 lint 工具，这里的 `npm run lint` 实际就是
`tsc --noEmit`）：

```bash
cd admin
npm install
npm run lint
npm run build
```

开发时让管理控制台对着一个正在运行的后台跑（热更新，不用每次改动都重新构建整个可执行文件）：

```bash
# 终端 1
cd server && go run ./cmd/server

# 终端 2
cd admin && npm run dev   # 会在 Vite 开发服务器端口打开，比如 http://localhost:5173
```

开发服务器会把 API 请求代理到后台；后台那边的 CORS 本来就配得比较宽松（见
`server/internal/api/router.go` 里的 `withCORS`），就是专门为了支持这种开发方式。

不想连真实设备测试后台的话，见下面的 [ubibot-open-simulator（设备模拟器）](#ubibot-open-simulator设备模拟器)。

## ubibot-open-ws1b（WS1B 固件）

需要 **ESP-IDF v6.0.2**，目标芯片是 **ESP32-C5**。按
[Espressif 官方指南](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c5/get-started/index.html)
针对你的操作系统装好，然后每开一个新终端都要先激活它的环境（`. ./export.sh`，Windows 上是
`export.ps1`），之后才能用 `idf.py` 命令。

```bash
git clone https://github.com/ubibot-open/ubibot-open-ws1b.git ubibot-open-ws1b
cd ubibot-open-ws1b
idf.py set-target esp32c5   # 只需要在第一次做
idf.py build
```

接上真实硬件之后，`idf.py -p <port> flash monitor` 会烧录完立刻进入串口日志查看——这是改固件时
最快的内循环方式。这个仓库没有能在主机上跑的单元测试（嵌入式 C 代码绑定了 ESP-IDF 的头文件）；
正确性靠在真实硬件上编译运行来验证，或者把你要测的逻辑挪到
[ubibot-open-simulator（设备模拟器）](#ubibot-open-simulator设备模拟器)里——那个是真的能在你自己
电脑上跑起来的。

## ubibot-serial-sync（桌面串口工具）

只有你要改这个工具本身的时候才需要从源码构建——单纯使用的话，下载对应操作系统的
[预编译版本](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest)就够了。要构建的话：

需要 **Qt 6.7+**（本项目实测跑在 6.11 上），带 **Quick**、**QuickControls2**、**SerialPort** 模块
以及 Qt Linguist 工具，**CMake 3.21+**，以及一个 **C++17** 编译器。

```bash
git clone https://github.com/ubibot-open/ubibot-serial-sync.git
cd ubibot-serial-sync
cmake -S . -B build -DCMAKE_PREFIX_PATH=/path/to/Qt/6.11.0/<你的套件>
cmake --build build
```

如果是要打包一份能直接交给别人用的构建产物，而不只是本地跑跑，具体的平台打包方式
（`windeployqt`/`macdeployqt`/`linuxdeployqt`）见该仓库自己的
[BUILD.md](https://github.com/ubibot-open/ubibot-serial-sync/blob/main/BUILD.md)。

## ubibot-open-simulator（设备模拟器）

需要一个 **C 编译器**（gcc/clang/MSVC）和 **CMake 3.10+**。不需要 Qt，不需要 ESP-IDF，不需要 Go。

```bash
git clone https://github.com/ubibot-open/ubibot-open-simulator.git
cd ubibot-open-simulator
cmake -S . -B build
cmake --build build
ctest --test-dir build --output-on-failure
```

如果你的改动其实是关于协议或者后台逻辑的，这是迭代速度最快的方式——见
《[用模拟器迭代后端](08-testing-with-the-simulator.zh-CN.md)》。

## 下一步

[新增一个后台 API 端点](02-backend-add-admin-endpoint.zh-CN.md)。
