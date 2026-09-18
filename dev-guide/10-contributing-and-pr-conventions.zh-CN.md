# 贡献与 PR 规范

*[English](10-contributing-and-pr-conventions.md)*

完整版是组织级的
[CONTRIBUTING.md](https://github.com/ubibot-open/.github/blob/main/CONTRIBUTING.md)——
`ubibot-open` 下面的每个仓库都遵循它，除非自己有单独的一份（目前没有哪个仓库有）。这一章是
精简版，外加 CI 实际会跑什么，让你在开 PR 之前知道会遇到什么。

## 开始之前

- **Bug 修复或小改动**：直接开 PR。
- **新功能，或者任何跨仓库改变行为的东西**（协议改动、新的 API 端点、新的后台页面）：先开一个
  issue。这个项目刻意保持精简（见《[架构总览](../architecture/overview.md)》的设计原则，以及
  《[手册说明](00-overview.zh-CN.md)》关于范围的说明）——先确认方向能省得你重写一遍 PR。
- **纯文档改动**：直接对
  [ubibot-open-doc](https://github.com/ubibot-open/ubibot-open-doc) 开 PR。

## 每个仓库的 CI 实际检查什么

| 仓库 | 触发时机 | 做什么 |
|---|---|---|
| `ubibot-open-server` | push 到 `main`、每个 PR | 后端跑 `go build`/`go vet`/`go test`；管理控制台跑 `npm run lint`/`npm run build`——两个独立的 job |
| `ubibot-open-ws1b` | push 到 `main`、每个 PR | `espressif/esp-idf-ci-action@v1`，ESP-IDF v6.0.2，目标芯片 `esp32c5`——真的跑一次 `idf.py build` |
| `ubibot-open-simulator` | push 到 `main`、每个 PR | `cmake`/`cmake --build`/`ctest`，跑在 Ubuntu+Windows 的矩阵上 |
| `ubibot-serial-sync` | 推版本 tag，或手动触发——**不是**每个 PR | 给三个操作系统构建并发布 release 包；**这个仓库的普通 PR 没有 CI 构建检查**，所以在开 PR 之前照着 [BUILD.md](https://github.com/ubibot-open/ubibot-serial-sync/blob/main/BUILD.md) 真的在本地构建一遍，比其他仓库更重要 |

推代码之前先在本地跑一遍同样的命令——具体到你在改的那个仓库该跑什么命令，见
《[开发环境搭建](01-environment-setup.zh-CN.md)》。

## PR 的具体流程

1. Fork，从 `main` 上开一个分支（`fix/short-description` 这种命名）。
2. 跟周围代码的风格和注释密度保持一致——除此之外没有单独的风格指南；几条跟具体栈相关的补充说明
   （Go：`gofmt`/`go vet` 干净、能用表驱动测试就用；React/TS：函数组件+hooks，跟现有的
   `pages/`/`components/` 结构保持一致；C/ESP-IDF：跟现有的文件头/注释格式对齐，用已有的
   `osi_*`/`mem_*` 封装而不是直接调 FreeRTOS/libc 原语；模拟器的 C/CMake：把主机专用代码限制在
   已经标记为主机专用的那几个文件里，因为大部分代码本来就是要兼容 FreeRTOS 移植的）。
3. 一个 PR 只做一件逻辑上独立的事。
4. 描述改了什么、为什么改、怎么测的（有对应 issue 就带上链接）。
5. 授权协议跟着你贡献的那个仓库走——Apache 2.0（`ubibot-open-server`、`ubibot-open-doc`、
   `ubibot-open-simulator`）、MIT（`ubibot-open-ws1b`）、LGPLv3（`ubibot-serial-sync`）。不需要
   单独签 CLA。

## 目前这本手册就是这些了

这本手册[目录](README.zh-CN.md)里剩下的章节会逐步补上——具体哪些写完了、哪些还是占位大纲，去
那边看就知道。
