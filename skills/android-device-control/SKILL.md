---
name: android-device-control
description: Reconnect to and control this Fairphone 6 (LineageOS, degoogled) from Termux via adb over Wi-Fi and Termux:API. Use whenever the user asks to see/screenshot/tap/control the phone screen, reconnect adb, check adb devices, or use termux-api (battery, clipboard, notifications, sensors, etc.) and it isn't already working.
---

# Android device control (this device)

This device is self-controlled: Termux (this shell) drives the same physical
Android device it runs on, via `adb` over Wi-Fi wireless debugging, plus
`termux-api` for OS features adb can't reach.

## 1. Check if already connected

```
adb devices -l
```

If it lists a `device` (authorized) entry, you're in — skip to capabilities.
If it says `unauthorized`, `offline`, or lists nothing, reconnect:
**→ read `reference/reconnect.md`**.

## 2. Check Termux:API is live

```
termux-battery-status
```

JSON back = working. Hangs, errors, or `command not found` — see
**`reference/reconnect.md`** (Termux:API section).

## 3. Do the thing

For the adb and termux-api command cheat sheet (screenshots, tap/swipe/text,
input, clipboard, notifications, sensors, etc.) and this device's known specs
(screen size, model, quirks) — **read `reference/capabilities.md`**.

Screenshots: capture with adb into the session scratchpad dir, then view with
the Read tool (Read renders images directly) — never try to view a PNG via
Bash/cat.
