# HKMC CarPlay Auto-Display — NEMESIS v1.0 (dAudio Gen5W)

Brings **Apple CarPlay to the foreground as soon as it connects — wired or
wireless** — using the unit's own iAP2/CarPlay connect event. Built on the
NEMESIS package model (`mode=restart` service, state dir, root diagnostics).
Auto-starts on boot.

**Tested on:** 2025 Kia EV6 GT-Line (US).

**Package id:** `hkmc.carplay-auto-display`
**Service:** `hkmc.carplay-auto-display/apply`

> Test changes while parked.

## How it works

A root service polls `logcat -d` (dump mode) every `poll_seconds` (default 3)
and watches for the unit's CarPlay-connect log line:

```
I/CarPlay_: [ TimeStampManager ] setCarPlayConnected() isConnected = true
```

Lines are read with timestamps (`logcat -d -v time`), and the service fires
only when a **new** connect line appears (timestamp-diffed) — so it triggers
once per real connect and never on stale/repeat lines. On a connect it runs
`am start -n com.daudio.app.dipo/.ui.DipoActivity`, which foregrounds CarPlay.

This one event fires for **both wired and wireless** CarPlay, so there's no
transport-specific logic.

### Notes on this unit (so the detector isn't naively changed)

- The iPhone never appears in `dumpsys usb` during CarPlay (iAP2 stack), so a
  USB vendor id is not usable.
- Every *level* signal is sticky: `sys.usb.state=carplay_connect` is wired-only;
  Wi-Fi `curState=TetheredState`, `sys.launcher3.pp.dipo`, the DiPO
  `CarPlayService` binding, and the SoftAP ARP entry all remain set after
  disconnect. Detection therefore keys on the logcat **event**, not a state.
- logcat **follow** mode (`logcat -T`) does not work in the service context,
  but logcat **dump** mode (`logcat -d`) does — which is what this uses.

## Install

Two supported methods. Use whichever is handy; both end at the same installed
package.

### A) ADB over USB

With the head unit connected to a computer by USB and USB debugging enabled:

```sh
adb push hkmc.carplay-auto-display-v1.0.nemesis.zip /data/local/tmp/
adb shell nemesis install --update /data/local/tmp/hkmc.carplay-auto-display-v1.0.nemesis.zip
adb shell nemesis service status hkmc.carplay-auto-display/apply
```

The last line should report the service as running (v1.0).

### B) USB flash drive

1. Extract `hkmc.carplay-auto-display-v1.0.USB.zip` to the **root** of a
   FAT32/exFAT USB flash drive. It contains `NEMESIS.data` and
   `packages/hkmc.carplay-auto-display-v1.0.nemesis.zip`.
2. Insert the drive into the **parked** vehicle and approve the install.

## Verify

```sh
adb shell nemesis log service hkmc.carplay-auto-display/apply | grep -E 'v1.0|connect line|am |probe'
```

Expect at start `probe: logcat readable (dump mode)` and a `baseline connect
line`, then on each connect: `new connect line: ...` → `am start -n
com.daudio.app.dipo/.ui.DipoActivity` → `am exit=0`. (Use `grep`, never `tail` —
this unit's shell has no `tail`.)

## Settings

One-value files in `/data/nemesis/state/hkmc.carplay-auto-display/`:

```sh
S=/data/nemesis/state/hkmc.carplay-auto-display
nemesis root -c "echo 3 > $S/poll_seconds"
nemesis root -c "echo 2 > $S/connect_grace_seconds"   # delay before switching
nemesis root -c "echo 8 > $S/cooldown_seconds"
# The log line matched (grep -E; escape the parens). Change only if the firmware wording differs:
nemesis root -c 'echo "setCarPlayConnected\(\) isConnected = true" > $S/connect_mark'
# Overrides (normally unneeded):
nemesis root -c "echo com.daudio.app.dipo/.ui.DipoActivity > $S/carplay_component"
nemesis root -c "printf 'am start -n com.daudio.app.dipo/.ui.DipoActivity' > $S/switch_cmd"
```

Settings are read at service start; after a change:

```sh
nemesis service stop  hkmc.carplay-auto-display/apply
nemesis service start hkmc.carplay-auto-display/apply
```

## Diagnostics / uninstall

```sh
nemesis pkg-launch hkmc.carplay-auto-display      # read-only discovery dump
nemesis uninstall  hkmc.carplay-auto-display      # stop auto-display
```
