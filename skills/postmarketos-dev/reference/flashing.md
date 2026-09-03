# Getting postmarketOS onto the real device

Not doable from this Termux session — plan it, don't attempt it here.
`fastboot` requires the phone sitting in bootloader mode with no OS running,
so nothing running on-device (including this session) can drive that step.
This needs the user's separate PC over USB.

## Current pmOS Fairphone 6 support (verified 2026-09-03 against actual kernel source, not secondhand posts)

Earlier note in this file (and an earlier answer given to the user) claimed
wifi/cellular/camera were still broken — that was a Sept 2025 forum
snapshot, a full year stale, and was wrong by the time it was cited. Don't
repeat that mistake: check the pinned kernel source directly, it's cheap
and authoritative.

- Official port exists, published the same day the FP6 shipped (June 2025).
  Installs via the web flasher at flash.postmarketos.org.
- **Confirmed working** (checked against `v7.2.0-milos`, the exact tag
  `device/testing/linux-postmarketos-qcom-milos/APKBUILD` builds, plus
  Aug 2026 dev logs — linmob.net weekly updates, postmarketos.org blog):
  - WiFi + Bluetooth: DT nodes present (`milos-fairphone-fp6.dts` has
    `wcn6755-bt`, `&wifi`), `CONFIG_ATH11K=m`/`CONFIG_ATH11K_AHB=m`/
    `CONFIG_BT_QCA=m` enabled in the shipped defconfig. Upstream mainline
    Linux merged the same DT patches (authored by the FP6 porter, Luca
    Weiss) on 2026-08-31.
  - Display/GPU driver framework enabled (`CONFIG_DRM_MSM`, DPU, DSI) —
    GPU node lives in a shared SoC-level `.dtsi`, not the per-device `.dts`,
    didn't chase it further but the driver stack is wired in, not absent.
  - All three cameras (main/wide/telephoto), audio/mic, VoLTE (at least
    emergency-call tested end-to-end), fingerprint unlock, offline charging.
- **Still unverified** (not confirmed either way — check these first once
  actually booted): regular (non-emergency) mobile data and SMS, NFC,
  suspend/battery behavior day-to-day.
- Verification method that actually works when the wiki is unreachable
  (bot-walled): pull the exact tag/commit the pmaports package builds
  (`_tag="v$pkgver-milos"` in the kernel APKBUILD) straight from
  `github.com/milos-mainline/linux` via the GitHub API/raw content, and
  grep the real `.dts`/defconfig — this is ground truth, unlike blog posts
  or forum threads which go stale fast on a port this actively developed.

## The plan: A/B dual boot, not a replace

Fairphone 6 has A/B seamless-update slots. Documented community pattern on
Fairphone hardware: Android stays on slot A, postmarketOS goes on slot B.

1. **Back up first.** Unlocking the bootloader factory-resets the device.
2. **Get the unlock code**: Fairphone's own site, IMEI-based
   (fairphone.com bootloader-unlocking-code page / support article "How to
   unlock or lock your Fairphone's bootloader"). Some FP6 units have hit an
   "OEM unlocking blocked" snag — check Settings → Developer options →
   OEM unlocking is actually togglable before assuming this will be smooth;
   search the Fairphone forum thread on this if it's greyed out.
3. Unlock: `fastboot flashing unlock_critical` (from the PC, device in
   bootloader mode), confirm on-device.
4. Flash postmarketOS to the **inactive slot** (the web flasher or manual
   `fastboot flash` per-partition, targeting slot B) while Android stays on
   slot A untouched.
5. Switch slots with `fastboot set_active a` / `fastboot set_active b`.
   **Never use `qbootctl` to switch slots** — community reports of hard
   bricks requiring EDL-mode recovery. `fastboot set_active` only.

## The actual flash sequence (community-documented pattern, not FP6-verified firsthand)

The web flasher (flash.postmarketos.org) flashes to whatever slot is
*currently active* — it doesn't have its own slot picker. So targeting slot
B means switching first, then flashing:

```
fastboot getvar current-slot          # confirm which slot is active now
fastboot set_active b                 # switch to the inactive slot first
# then run the web flasher (or manual `fastboot flash boot_b <img>` etc.)
# targeting the now-active slot
fastboot set_active a                 # switch back to boot Android normally
```

To boot into postmarketOS later: `fastboot set_active b` then power on
normally (or reboot to bootloader and pick the slot there, device-menu
permitting). To get back to Android: `fastboot set_active a`.

This is the pattern documented across multiple Qualcomm A/B devices (XDA
dual-boot guides, Fairphone forum reports) — not something verified
firsthand against FP6 specifically, since that step needs the PC + physical
device. Confirm `fastboot getvar current-slot` actually reflects reality
before and after each switch — don't assume it worked.

## Still open

Exact wipe/flash partition list for FP6 slot B (some devices need
`vendor_boot_<slot>` and `dtbo_<slot>` explicitly wiped before flashing
`boot_<slot>` — seen in other A/B dual-boot guides, not yet confirmed
required for FP6). Check the web flasher's own log output when running it —
it should state exactly which partitions it touches.
