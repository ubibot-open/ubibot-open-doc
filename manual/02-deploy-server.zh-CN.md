# 部署后台服务

*[English](02-deploy-server.md)*

这一章结束后，你会把 `ubibot-open-server` 构建并运行起来，并且第一次登录进管理控制台。

## 1. 拉取代码

```bash
git clone https://github.com/ubibot-open/ubibot-open-server.git
cd ubibot-open-server
```

## 2. 构建

```bash
./build.sh        # Linux/macOS
# .\build.ps1      # Windows
```

这一步会先构建管理控制台的前端，然后编译 Go 后台服务，把前端内嵌进去——最终只有一个可执行文件
`ubibot-server`（Windows 上是 `ubibot-server.exe`），API 和网页界面都在这一个文件里。第一次运行会
花一两分钟（要下载 Go 模块和 npm 依赖包）；之后再构建会快很多。

**截图占位符：** `02-deploy-server-01.png` —— 构建成功的终端输出，以 `==> Done: ubibot-server`
结尾。

## 3. 运行

```bash
./ubibot-server
```

观察日志输出。如果是全新的数据库，你会看到类似下面这两行：

```
no admin account found — created "admin" with a generated password: 3f9a1c...
this password is only shown once; set UBIBOT_ADMIN_PASSWORD to control it on next first run
```

> **注意：** 现在就把这个密码记下来。它是随机生成的，只会打印这一次——没有"忘记密码"这种功能。
> 如果弄丢了，看常见问题排查那一章，里面有怎么重置的方法。

服务默认监听 8080 端口（`http://localhost:8080`——API 和管理控制台都从这同一个地址提供服务；
可以通过设置 `UBIBOT_LISTEN_ADDR` 环境变量来改端口）。

**截图占位符：** `02-deploy-server-02.png` —— 终端显示生成的密码那两行日志，以及
`ubibot API listening on :8080` 那一行。

## 4. 登录

在浏览器里打开 `http://localhost:8080`，应该会看到登录页。

**截图占位符：** `02-deploy-server-03.png` —— 登录页。

用下面的信息登录：

- **用户名：** `admin`
- **密码：** 第 3 步日志里打印出来的那个

**截图占位符：** `02-deploy-server-04.png` —— 第一次登录成功后的仪表盘（这时候基本是空的——还
没有设备）。

## 下一步

[首次登录与导航](03-first-login-and-tour.zh-CN.md)。
