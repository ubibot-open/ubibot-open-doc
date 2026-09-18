# Prerequisites

*[中文](01-prerequisites.zh-CN.md)*

## For everyone

You need these regardless of which path (with hardware or without) you're following, since
they're what builds and runs the backend and admin console:

| Tool | Version | Used for |
|---|---|---|
| [Go](https://go.dev/dl/) | 1.23 or newer | Building the backend (`ubibot-open-server`) |
| [Node.js](https://nodejs.org/) | 18 or newer (any current LTS), with npm | Building the admin console's frontend |
| [Git](https://git-scm.com/) | any recent version | Cloning the repositories |

> **Note:** the backend compiles to a single, self-contained binary with the admin console's
> built frontend embedded inside it (see [build.sh](https://github.com/ubibot-open/ubibot-open-server/blob/main/build.sh)).
> You only need Go and Node.js on the machine where you *build* it — running the resulting binary
> needs neither.

## If you have real WS1B hardware

You'll additionally need:

| Tool / item | Notes |
|---|---|
| A UbiBot WS1B device | Or any board built on the ESP32-C5, running this project's reference firmware |
| A USB cable | Data-capable, not charge-only — connects the device to your computer for flashing and serial provisioning |
| [ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c5/get-started/index.html) v6.0.2 | Espressif's toolchain for building and flashing the firmware. Follow their official installation guide for your OS — it also installs the `idf.py` command-line tool this manual uses |
| A USB-to-serial driver | Most WS1B boards use a CH340 or CP210x USB-serial bridge chip; Windows/macOS may need a driver installed the first time one is plugged in. [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync)'s own first-run check will tell you if one is missing |
| [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest) | The desktop tool used to view logs and provision the device over serial. Download the prebuilt package for your OS — no build required |

Chapters [4](04-flash-firmware.md) and [5](05-provision-device-over-serial.md) cover installing
and using these.

## If you don't have hardware

Skip the row above — instead, [chapter 7](07-no-hardware-path.md) only needs a C compiler
(gcc/clang/MSVC — whatever you'd normally have available) and [CMake](https://cmake.org/download/)
3.10+, to build [ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator).

## Next

[Deploy the Backend Server](02-deploy-server.md).
