# Overview

*[中文](00-overview.zh-CN.md)*

This handbook is a set of task-oriented walkthroughs for changing UbiBot Open's code — "I want to
add X, show me how" rather than "here's how the system is shaped". Read the
[Architecture Overview](../architecture/overview.md) first if you haven't; it explains the five
repositories, how they talk to each other, the request flow from device to dashboard, and the
design principles the codebase leans on. This handbook assumes that context.

If you want to *operate* the platform rather than modify it, see the [User Manual](../manual/README.md)
instead.

## The checklist every chapter here follows

Each worked example in this handbook — adding an API endpoint, a data model, an admin page, a
firmware command — follows the same shape, because that's how the existing code was actually
built and reviewed:

1. **Make the change** in the smallest unit that compiles/builds on its own (a store method, a
   handler, a page component, a firmware function).
2. **Wire it in** — register the route, the menu entry, the CMake source list, whatever makes the
   new code reachable.
3. **Update the relevant doc** if the change affects something documented elsewhere: a new
   `/api/admin/*` route belongs in [api/admin-api.md](../api/admin-api.md); a wire-format change
   belongs in [protocol/hardware-communication-protocol.md](../protocol/hardware-communication-protocol.md).
4. **Add or extend a test** that exercises the change through its real entry point (an HTTP
   request against the router, not just calling the Go function directly) wherever the existing
   code already has that kind of test coverage.
5. **Verify against a real toolchain** — `go build`/`go test`, `npm run build`/`npm run lint`,
   an actual `idf.py build`, or `ctest`, depending on which repo you're in. Don't consider a
   change done on the strength of it "looking right".

## A note on scope

This project deliberately keeps a narrow scope — see the standing decisions recorded in the
[Architecture Overview](../architecture/overview.md)'s design-principles section (minimal by
default, fire-and-forget over ack/retry, no second hardware model). If a chapter here shows you
how to extend something, that doesn't mean every extension is in scope for a PR back to this
project — when in doubt, open an issue describing what you want to add before investing time in
building it.

## Chapters

See the [table of contents](README.md) for the full list. Start with
[Environment Setup](01-environment-setup.md) if you haven't built any of these repos before.
