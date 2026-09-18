# Contributing and PR Conventions

*[中文](10-contributing-and-pr-conventions.zh-CN.md)*

The full guide is the org-wide
[CONTRIBUTING.md](https://github.com/ubibot-open/.github/blob/main/CONTRIBUTING.md) — every repo
under `ubibot-open` defers to it unless it has its own (none currently do). This chapter is the
short version plus what actually runs in CI, so you know what to expect before opening a PR.

## Before you start

- **Bug fix or small change**: open a PR directly.
- **New feature, or anything that changes behavior across repos** (a protocol change, a new API
  endpoint, a new admin-console page): open an issue first. This project stays deliberately
  minimal (see the [Architecture Overview](../architecture/overview.md)'s design principles, and
  [Overview](00-overview.md)'s note on scope) — confirming direction first saves a rewritten PR.
- **Docs-only change**: straight to a PR against
  [ubibot-open-doc](https://github.com/ubibot-open/ubibot-open-doc).

## What CI actually checks, per repo

| Repo | Runs on | What it does |
|---|---|---|
| `ubibot-open-server` | push to `main`, every PR | `go build`/`go vet`/`go test` for the backend; `npm run lint`/`npm run build` for the admin console — two separate jobs |
| `ubibot-open-ws1b` | push to `main`, every PR | `espressif/esp-idf-ci-action@v1`, ESP-IDF v6.0.2, target `esp32c5` — a real `idf.py build` |
| `ubibot-open-simulator` | push to `main`, every PR | `cmake`/`cmake --build`/`ctest`, on an Ubuntu+Windows matrix |
| `ubibot-serial-sync` | a version tag push, or manual dispatch — **not** every PR | Builds and publishes release packages for all three OSes; **there's no CI build check on ordinary PRs** for this repo, so actually running the build locally per [BUILD.md](https://github.com/ubibot-open/ubibot-serial-sync/blob/main/BUILD.md) before opening one matters more here than elsewhere |

Run the same commands locally before pushing — see [Environment Setup](01-environment-setup.md)
for the exact command for whichever repo you're touching.

## PR mechanics

1. Fork, branch off `main` (`fix/short-description` or similar).
2. Match the surrounding code's style and comment density — there's no separate style guide
   beyond that; a few stack-specific notes (Go: `gofmt`/`go vet` clean, table-driven tests where
   practical; React/TS: functional components + hooks matching the existing `pages/`/`components/`
   layout; C/ESP-IDF: match the existing header/comment format and the `osi_*`/`mem_*` wrappers
   rather than calling FreeRTOS/libc directly; C/CMake for the simulator: keep host-only code
   confined to the files already marked as such, since most of it doubles as portable
   FreeRTOS-ready source).
3. One logical change per PR.
4. Describe what changed, why, and how you tested it (link the issue, if any).
5. Licensing follows whichever repo you're contributing to — Apache 2.0 (`ubibot-open-server`,
   `ubibot-open-doc`, `ubibot-open-simulator`), MIT (`ubibot-open-ws1b`), LGPLv3
   (`ubibot-serial-sync`). No separate CLA.

## That's the whole handbook, for now

The remaining chapters in this handbook's [table of contents](README.md) are filled in as they're
written — check there for what's covered and what's still an outline-only stub.
