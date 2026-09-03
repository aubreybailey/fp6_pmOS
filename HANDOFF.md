# Handoff

Living document — whoever's picking up this project (a different Claude
Code instance, likely running on-device under booted postmarketOS itself
rather than under Android/Termux) starts here. Update this file before you
stop, so the next session (wherever it runs) picks up cleanly instead of
re-deriving state. Keep it current, not historical — move finished items
out, don't just append.

If you're the pmOS-side instance: you have real root and no proot
limitations, which the Android/Termux-side instance doesn't have. Use
that — things documented as "couldn't verify" or "worked around" in the
skills/patches here were often constrained by that, not by anything
fundamental.

## Priority order

1. **Confirm basic connectivity first.** Before anything else: does WiFi
   actually come up? (`docs/status.md` says yes at the kernel/DT level,
   confirmed by reading the pinned kernel source directly — but that's
   never been verified against a real boot.) This gates almost everything
   else, including whether you can `apk add`/`git pull`/report back at
   all without a USB-ethernet/SSH fallback.

2. **NFC** — the actual open item, `patches/nfc/`. The patches there are
   build-tested clean (DTS compiles, Kconfig resolves, driver source
   compiles) but never run on hardware. Specific things to check, in
   order:
   - `i2cdetect -y 1` (confirm bus number — `i2c1` in the mainline DTS,
     double check against real hardware) — does *anything* answer at
     `0x27`? This isolates "chip not wired right" from "chip wired right,
     wrong driver."
   - `dmesg | grep -iE "s3fwrn|nfc|i2c"` after boot — does the
     `samsung,s3fwrn5-i2c` driver even attempt to probe? What's the
     failure mode if it does and fails (timeout vs. protocol
     rejection vs. clean bind)?
   - The real open question from `patches/nfc/NOTES.md`: Fairphone's own
     vendor driver config references a newer chip generation ("rn4v")
     than S3FWRN5. If the S3FWRN5 driver doesn't bind, that's the likely
     reason — check `vendor/samsung_slsi/nfc` (cloneable from
     `gerrit-public.fairphone.software`, see `patches/nfc/NOTES.md` for
     the exact path) for what chip family "rn4v" actually corresponds to,
     and whether a newer mainline driver (there may not be one yet)
     matches better.

3. **Re-verify the rest of `docs/status.md` against real hardware**, not
   just kernel source — it's currently a "should work based on config"
   assessment for camera/audio/VoLTE/fingerprint (those came from dev
   logs, at least secondhand-confirmed) vs. genuinely unverified for
   mobile data/SMS/GPU-in-practice/suspend-battery. Update the doc with
   what you actually observe, and note the date you checked.

## Questions worth answering and writing back, even if the answer is
"still don't know"

- Does GPU acceleration actually work in practice (not just
  "driver framework enabled")? Anything glxinfo/wayland-compositor-level
  you can check.
- Real mobile data (not just emergency VoLTE) and SMS — do they work?
- Suspend/wake and battery drain over a normal period of non-use — this
  is the "sounds fine in config, often isn't" category on immature ports.
- If NFC ends up needing a different driver than S3FWRN5: is there
  already a mainline driver for whatever "rn4v" actually is, or is this a
  bigger porting job than the current patch anticipated?

## Where things live

- `docs/status.md` — hardware support status, keep it current.
- `patches/<subsystem>/` — one directory per subsystem, each with its own
  `NOTES.md` for sourcing/risk/status. Follow that pattern for new work.
- `skills/` — the actual Claude Code skills (symlinked into
  `~/.claude/skills/` on the Android/Termux side; on pmOS you'd want them
  symlinked the same way if this repo gets cloned into a Claude Code
  session there — check whether `~/.claude/skills/` exists as the
  install location on that side too, don't assume).

## Note on going back the other direction

If you're on the Android/Termux side reading this after the pmOS side did
work: check `docs/status.md`'s "last verified" date and git log for what
changed, don't assume anything here is still what it was — that's the
whole point of this file.
