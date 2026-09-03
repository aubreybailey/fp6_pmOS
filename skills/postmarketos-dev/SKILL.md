---
name: postmarketos-dev
description: Build and edit postmarketOS (pmaports) packages natively on this Fairphone 6, and know the real path to a bootable on-device install. Use whenever the user asks about postmarketOS, pmaports, pmbootstrap, abuild, dual-booting/A/B slots with Android, or porting/developing for this phone's Linux side.
---

# postmarketOS development (this device)

Goal: contribute to/extend postmarketOS for this Fairphone 6, which already
has an official (early, WIP) pmOS port. Two separate tracks, don't conflate
them:

1. **Package dev — works today, entirely in Termux.** Build/edit individual
   pmaports packages natively (this device is aarch64, the target is
   aarch64 — no cross-compilation, unlike doing this on an x86 laptop).
2. **Real device flashing — needs a separate PC over USB.** Bootloader
   unlock and `fastboot` slot flashing cannot run from Termux on-device:
   fastboot mode has no OS running, so nothing here can drive it. This step
   happens on the user's PC, not in this session.

## 1. Check the Alpine build container exists

```
proot-distro list
```

Shows `alpine` if already set up. If not: **read `reference/setup.md`**.

## 2. Build a package

```
proot-distro login alpine -- sh -c 'cd <pkgdir> && abuild -F -r'
```

`-F` (force-root) is required here, not optional — see
**`reference/workflow.md`** for why, and for the full command set (fetching
pmaports, testing, output paths, what doesn't work and why).

## 3. Getting it onto real hardware

Don't attempt bootloader unlock or fastboot from here. **Read
`reference/flashing.md`** for the dual-boot plan to hand to the PC side of
this work.
