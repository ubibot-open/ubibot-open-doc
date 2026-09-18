# Provision the Device Over Serial

*[中文](05-provision-device-over-serial.zh-CN.md)*

The `menuconfig` values you set in [chapter 4](04-flash-firmware.md) are only factory defaults.
This chapter overrides them at runtime, over the same serial port, without rebuilding or
reflashing — the WiFi network, the server address, and (if you flashed a whole batch from one
build) this unit's own serial number.

## 1. Open ubibot-serial-sync and connect

Launch [ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync/releases/latest).
In the serial connection panel, pick the device's port and set the baud rate to **115200**
(matching the firmware's log/console UART) — the port picker flags likely candidates (CH340/
CP210x USB-serial chips) as recommended. Open the port.

**Screenshot placeholder:** `05-provision-device-over-serial-01.png` — the serial connection panel
with the right port selected and connected, boot log already scrolling in the data monitor pane.

## 2. Re-open the provisioning window

The device only listens for provisioning commands in a short window **right after power-on** — it
closes automatically after 5 seconds of serial silence, or 60 seconds total, whichever comes
first, and does not reopen on a routine wake-up. If you already missed it (e.g. you were reading
this chapter instead of typing), just power-cycle the device — unplug and replug the USB cable, or
press its reset button — to open a fresh window.

## 3. Set the WiFi network

In the manual-send box (right-hand data monitor pane, always visible regardless of which panel is
active), type:

```json
{"command":"SetupWifi","ssid":"YourNetworkName","password":"YourPassword","type":"WPA2"}
```

and send it. For an open (no-password) network, omit `password` or leave it empty.

> **Note:** `type` is required by the protocol, but the current firmware only stores it — it
> doesn't yet use it to select a specific authentication mode when connecting (the underlying
> WiFi driver negotiates that on its own). Send a value anyway (`WPA2` for a password-protected
> network, `OPEN` for one without) for forward compatibility.

Watch for the acknowledgment in the log:

```json
{"c":0,"msg":"wifi saved"}
```

**Screenshot placeholder:** `05-provision-device-over-serial-02.png` — the manual-send box with
the `SetupWifi` command typed, and the `{"c":0,...}` ack visible in the log above it.

`c: 0` means success. `c: 1` means something was missing or invalid (fix and resend — you still
have time left in the window, since each byte received resets the idle countdown). `c: 2` means
validation passed but the value failed to actually save — rare, but worth a retry.

## 4. Set the server address

```json
{"command":"SetupServer","host":"192.168.1.100","port":8080}
```

Use the LAN IP address of the machine running `ubibot-server` from [chapter 2](02-deploy-server.md)
— not `127.0.0.1`/`localhost`, which would point the device at itself. `port` should match
whatever `UBIBOT_LISTEN_ADDR` is set to (`8080` by default).

## 5. (Batch-flashed devices only) Set this unit's serial number

Skip this if you flashed one device with its own unique serial number already baked in via
`menuconfig`. If you flashed a whole batch from a single build (same `pid`, same default `sn`),
give this physical unit its own:

```json
{"command":"SetupDevice","sn":"a-unique-value-for-this-unit"}
```

There's no equivalent command for `pid` — every unit from the same firmware build is expected to
share it.

## 6. Confirm it took effect

All three settings apply **immediately**, within the device's current run — no reboot needed. The
serial log should show it connecting to the new WiFi network and reporting to the new server
address within moments. They're also saved to non-volatile storage, so they survive power loss and
deep-sleep wake-ups until you provision the device again.

**Screenshot placeholder:** `05-provision-device-over-serial-03.png` — the serial log showing the
device connecting to WiFi and successfully posting a report, right after provisioning.

## Next

[Verify the Device Is Online](06-verify-device-online.md).
