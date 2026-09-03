# Reconnecting

## adb: already paired before, just disconnected (most common case)

The pairing is remembered by Android across adb server restarts and most
reboots; only the TCP listener needs to be back up. Ask the user to open
**Settings → System → Developer options → Wireless debugging** (they may need
to toggle it on) and read you the IP:port shown on that main screen — this is
the *connect* port, different from the one-time pairing port.

```
adb connect <ip>:<port>
adb devices -l
```

That's usually enough. If it still shows nothing or `failed to connect`, the
pairing itself is gone (e.g. Termux data was cleared, or Wi-Fi network
changed and pairing got invalidated) — do a full re-pair below.

## adb: full re-pair (fresh Termux install, or pairing lost)

1. `pkg install -y android-tools` if `adb` isn't found.
2. Ask the user to open **Settings → System → Developer options → Wireless
   debugging → Pair device with pairing code**. This screen shows a
   *different* IP:port than the main screen, plus a 6-digit code.
3. Get the IP:port and code from the user **and pair immediately** — the
   code expires in roughly a minute and is single-use. Don't do other work
   in between asking and running this:
   ```
   adb pair <pair-ip>:<pair-port> <6-digit-code>
   ```
4. Then connect using the address from the *main* Wireless debugging screen
   (not the pairing one):
   ```
   adb connect <connect-ip>:<connect-port>
   adb devices -l
   ```
5. If `adb pair` returns `protocol fault (couldn't read status message):
   Success` — the code already expired. Ask for a freshly generated one and
   retry right away.

Developer options must exist first: **Settings → About phone → tap Build
number 7×**.

## Termux:API

Two separate pieces are required — the Termux:API **app** (installed via
F-Droid, not Play, since this device is degoogled) and the `termux-api`
**Termux package**:

```
pkg install -y termux-api
termux-battery-status   # smoke test
```

If the app isn't installed, ask the user to grab it from F-Droid (same repo
family as Termux itself — same-signature requirement means it must come from
the same source Termux did). Individual `termux-*` calls (camera, sms,
location, contacts, notifications) will pop a runtime permission dialog the
*first* time each is used — tell the user to expect and grant it; the call
will hang/timeout until they respond.
