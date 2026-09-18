# Environment Setup

*[中文](01-environment-setup.zh-CN.md)*

One section per repository — set up whichever ones you're actually going to touch. Each shows the
commands that also run in that repo's own CI, so if these pass locally, CI will too.

## ubibot-open-server (Go backend + React admin console)

Requires **Go 1.23+** and **Node.js 18+**.

```bash
git clone https://github.com/ubibot-open/ubibot-open-server.git
cd ubibot-open-server
```

Backend — build, vet, and test:

```bash
cd server
go build ./...
go vet ./...
go test ./...
```

Admin console — install, type-check, and build (there's no separate lint tool; `npm run lint` here
is `tsc --noEmit`):

```bash
cd admin
npm install
npm run lint
npm run build
```

Run the admin console against a live backend during development (hot reload, instead of rebuilding
the whole binary on every change):

```bash
# terminal 1
cd server && go run ./cmd/server

# terminal 2
cd admin && npm run dev   # opens on a Vite dev server port, e.g. http://localhost:5173
```

The dev server proxies API calls to the backend, and CORS is already permissive on the backend
side (see `withCORS` in `server/internal/api/router.go`) specifically to support this.

See [ubibot-open-simulator](#ubibot-open-simulator-device-simulator) below for testing the backend
without any real device.

## ubibot-open-ws1b (WS1B firmware)

Requires **ESP-IDF v6.0.2**, targeting **ESP32-C5**. Install it per
[Espressif's official guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c5/get-started/index.html)
for your OS, then activate its environment in every new terminal (`. ./export.sh`, or `export.ps1`
on Windows) before `idf.py` is available.

```bash
git clone https://github.com/ubibot-open/ubibot-open-ws1b.git ubibot-open-ws1b
cd ubibot-open-ws1b
idf.py set-target esp32c5   # first time only
idf.py build
```

With real hardware connected, `idf.py -p <port> flash monitor` flashes and immediately drops into
the serial log viewer — the fastest inner loop for firmware changes. There's no host-run unit test
suite for this repo (embedded C tied to ESP-IDF headers); correctness is verified by building and
running on real hardware, or by porting the logic you're testing into
[ubibot-open-simulator](#ubibot-open-simulator-device-simulator) instead, which *does* run on your
own machine.

## ubibot-serial-sync (desktop serial tool)

You only need to build this from source if you're changing the tool itself — using it just needs
the [prebuilt release](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest) for your
OS. To build:

Requires **Qt 6.7+** (validated against 6.11) with the **Quick**, **QuickControls2**, and
**SerialPort** modules plus the Qt Linguist tools, **CMake 3.21+**, and a **C++17** compiler.

```bash
git clone https://github.com/ubibot-open/ubibot-serial-sync.git
cd ubibot-serial-sync
cmake -S . -B build -DCMAKE_PREFIX_PATH=/path/to/Qt/6.11.0/<your-kit>
cmake --build build
```

See that repo's own [BUILD.md](https://github.com/ubibot-open/ubibot-serial-sync/blob/main/BUILD.md)
for platform-specific packaging (`windeployqt`/`macdeployqt`/`linuxdeployqt`) if you're producing a
build to hand to someone else, not just running it locally.

## ubibot-open-simulator (device simulator)

Requires a **C compiler** (gcc/clang/MSVC) and **CMake 3.10+**. No Qt, no ESP-IDF, no Go.

```bash
git clone https://github.com/ubibot-open/ubibot-open-simulator.git
cd ubibot-open-simulator
cmake -S . -B build
cmake --build build
ctest --test-dir build --output-on-failure
```

This is the fastest thing to iterate against when your change is really about the protocol or the
backend — see [Iterate on the Backend with the Simulator](08-testing-with-the-simulator.md).

## Next

[Add a New Admin API Endpoint](02-backend-add-admin-endpoint.md).
