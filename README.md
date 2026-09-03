# fp6_pmOS

Working notes, patches, and Claude Code skills for developing postmarketOS
for the Fairphone 6, from directly on the device (Termux + proot Alpine,
native aarch64 — no cross-compilation needed).

**Picking this up fresh (including on a different machine/OS boot, e.g. a
Claude Code instance running under booted pmOS itself)? Read
[`HANDOFF.md`](HANDOFF.md) first** — current priorities and open
questions, kept up to date rather than historical.

## Layout

- `skills/` — Claude Code skills (symlinked into `~/.claude/skills/` on
  this device, so this repo is the actual source of truth for them):
  - `android-device-control/` — reconnecting adb/Termux:API to control this
    phone (screen, input, sensors) from Termux.
  - `postmarketos-dev/` — the build environment (proot Alpine + abuild),
    known proot limitations and workarounds, and the real dual-boot plan
    for flashing to slot B.
- `patches/` — actual kernel/pmaports patches, organized by subsystem.
  Each subdirectory has its own `NOTES.md` with sourcing and known risks —
  read those before trusting a patch is correct.
- `docs/status.md` — current hardware-support status, verified against
  real kernel source rather than secondhand blog posts, with the method
  documented so it can be re-checked instead of re-guessed.

## Approach

This device is aarch64 and so is the target — package/kernel work happens
natively in a Termux `proot-distro` Alpine container, no emulation. Real
device flashing (bootloader unlock, `fastboot`) can't happen from Termux
itself (no OS running in fastboot mode) and is planned/documented here for
execution on a separate PC over USB.

Fairphone publishes kernel/devicetree source under GPL on
`gerrit-public.fairphone.software` — real hardware wiring (GPIOs,
compatible strings) comes from there wherever possible, not guessed.
