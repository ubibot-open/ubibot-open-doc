# Overview

*[中文](00-overview.zh-CN.md)*

Welcome. This manual walks you from an empty folder to a UbiBot Open platform with a device
reporting data on a dashboard — every step spelled out, with a screenshot of what you should be
seeing at each point. It assumes no prior familiarity with this project, Go, ESP-IDF, or the admin
console.

## Who this is for, and what it isn't

If you already know your way around a shell, Git, and a build toolchain, and just want the exact
commands and flags, the [Deployment, Flashing & Bring-up Guide](../guides/deployment-flashing-guide.md)
covers the same ground more densely, as a reference checklist rather than a tutorial. This manual
and that guide describe the same underlying steps — pick whichever matches how you like to learn.
Many of this manual's command blocks are the same ones, just with more explanation around them.

This manual is about **operating** the platform: deploying it, connecting a device to it, and
using the admin console day to day. If what you actually want to do is **change the platform's
code** — add an API endpoint, a new admin page, or teach the firmware a new command — see the
[Developer Handbook](../dev-guide/README.md) instead.

## Two paths

UbiBot Open is a five-repository project (see the [Architecture Overview](../architecture/overview.md)
for the full picture), but you don't need all five to follow this manual:

- **With real WS1B hardware**: chapters 1 through 6 take you from a bare backend to a physical
  device reporting over WiFi, using [ubibot-open-ws1b](https://github.com/ubibot-open/ubibot-open-ws1b)
  firmware and the [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync) desktop
  tool to provision it.
- **No hardware on hand**: skip from chapter 3 straight to [chapter 7](07-no-hardware-path.md),
  which uses [ubibot-open-simulator](https://github.com/ubibot-open/ubibot-open-simulator) — a
  program that runs on your own machine and speaks the exact same protocol a real device would —
  to reach the same end state (a device online and reporting).

Either way, once a device is online, Part B (chapters 8 onward) is identical: managing devices,
sending commands, setting up alerts, and the rest of the admin console.

## How each chapter is laid out

Every chapter is a short, numbered sequence of actions. A screenshot follows any step where
something changes on your screen, so you can confirm you're in the same place before moving on.
A callout like this flags something easy to miss:

> **Note:** callouts like this one point out a common mistake, an optional detour, or something
> that behaves differently than you might expect.

## Next

Start with [Prerequisites](01-prerequisites.md) to see what to install before the first command.
