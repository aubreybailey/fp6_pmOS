# Capabilities cheat sheet

## This device

- Fairphone 6, LineageOS (degoogled, F-Droid-only), no root.
- Screen: 1116x2484 physical.
- Termux user is a sandboxed app UID (`u0_a231`), not root; `su` is Termux's
  stub and finds no real root binary.
- adb connects over Wi-Fi wireless debugging to itself (shows up as
  `emulator-5554` in `adb devices` — that's just adb's naming for a
  loopback/local transport, it's not an actual emulator).

## adb: screen + input

```
adb exec-out screencap -p > <scratchpad>/shot.png   # then Read tool to view
adb shell input tap <x> <y>
adb shell input swipe <x1> <y1> <x2> <y2> [duration_ms]
adb shell input text "hello"                         # no spaces-as-%s needed, quote it
adb shell input keyevent KEYCODE_HOME
adb shell input keyevent KEYCODE_BACK
adb shell wm size                                    # confirm resolution before tapping
```

Coordinates are in the device's physical pixel space reported by `wm size`,
matching the raw screencap PNG dimensions — no scaling needed between the two.
Eyeballing icon centers from a downscaled screenshot view is imprecise
(easy to be off by a whole icon); after any tap meant to hit something
specific, re-screencap immediately to confirm what actually got hit before
proceeding.

## adb: apps

```
adb shell pm list packages | command grep -i <name>
adb shell monkey -p <package> -c android.intent.category.LAUNCHER 1   # launch
adb shell am force-stop <package>
adb install <path/to.apk>
adb uninstall <package>
adb shell dumpsys window | command grep mCurrentFocus   # what's on screen now
```

## adb: files / logs / settings

```
adb pull <device-path> <local-path>
adb push <local-path> <device-path>
adb logcat -d | tail -100
adb shell settings get system screen_off_timeout
adb shell settings put system screen_off_timeout <ms>
```

## termux-api: device features

Each of these is a real Android permission the Termux:API app must be
granted — first call per feature triggers the OS permission prompt.

```
termux-battery-status
termux-clipboard-get / termux-clipboard-set "text"
termux-notification-list
termux-notification --title T --content C
termux-vibrate
termux-torch on|off
termux-tts-speak "text"
termux-location                 # slow, GPS fix takes several seconds
termux-sensor -l                # list sensors
termux-camera-photo -c 0 <file.jpg>
termux-sms-list / termux-sms-send -n <number> "text"
termux-telephony-call <number>
termux-wifi-connectioninfo
termux-share -a send <file>
```

## APK inspection

`aapt`, `jadx`, `apktool`, `unzip` are installed (pulled in OpenJDK 21).
`jadx`/`apktool` need `JAVA_HOME` — set persistently in `~/.bashrc`
(`/data/data/com.termux/files/usr/lib/jvm/java-21-openjdk`), so a normal
interactive shell has it, but a fresh non-interactive `Bash` tool call may
not source `.bashrc` — prefix with `JAVA_HOME=... jadx ...` if `jadx` errors
about JAVA_HOME.

```
adb pull "<device-path-from-pm-list-packages--f>" <local>.apk
aapt dump badging <apk>          # package/version/sdk/labels
aapt dump permissions <apk>      # uses-permission list
unzip -l <apk>                   # raw file listing (dex, libs, assets)
jadx -d <outdir> <apk>           # decompile to readable Java in <outdir>/sources
apktool d <apk> -o <outdir>      # decode resources/manifest to readable XML + smali
```

Get a package's install path with:
`adb shell pm list packages -f | command grep <name>`

## Known quirks in this shell

- `grep` and `find` are shadowed by Claude-Code-installed shell **functions**
  that shell out to `ugrep`/`ufind`-wrapped binaries; both are currently
  broken here (`-G`/`-S`: error while loading shared libraries). Use
  `command grep` / `command find` to bypass them, not `\grep` (backslash
  doesn't skip functions, only aliases).
- `which`, `file`, `ip` are not installed in this minimal Termux base;
  use `command -v`, and `ifconfig`/`getprop`/`termux-wifi-connectioninfo`
  for network info instead of `ip addr`.
- To view a screenshot, always use the Read tool on the saved PNG path —
  never try to cat/print binary image data via Bash.
