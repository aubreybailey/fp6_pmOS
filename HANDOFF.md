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

## Rule zero: search for existing work before starting anything

Before drafting any patch, search lore/netdev/linux-arm-msm (ratatoskr.run
indexes these), the `milos-mainline/linux` branches, pmaports MRs, and the
porters' blogs (catcrafts.net, linmob.net weekly updates) for the
subsystem. The first NFC patch here was drafted without doing this and
turned out to duplicate — and be worse than — a series already at v5
upstream. The useful contributions from this repo are more likely to be
hardware testing (`Tested-by` on a Gen 6), integration of list-only work
into what pmOS actually ships, and verifying the undocumented areas.

## Priority order

1. **Confirm basic connectivity first.** Before anything else: does WiFi
   actually come up? (`docs/status.md` says yes at the kernel/DT level,
   confirmed by reading the pinned kernel source directly — but that's
   never been verified against a real boot.) This gates almost everything
   else, including whether you can `apk add`/`git pull`/report back at
   all without a USB-ethernet/SSH fallback.

2. **NFC** — `patches/nfc/`. Read the 2026-09-26 update at the top of
   `NOTES.md` first: our S3FWRN5-based DTS patch is superseded. The chip
   is an S3NRN4V, and an upstream series in review (Jorijn van der Graaf,
   v5 as of Aug 2026) adds driver support; tag reading works with it,
   card emulation doesn't exist anywhere yet. So the job is now:
   - DONE 2026-09-26: v5 fetched (patchwork), applied to `v7.2.0-milos`
     with two prerequisite backports, and passes every non-hardware check
     (Kconfig, DTB, driver W=1, dt_binding_check, CHECK_DTBS). Exact
     recipe and results: `patches/nfc/upstream-s3nrn4v-v5/README.md`.
     What's left is the hardware test, which is on you.
   - On hardware, sanity check with `i2cdetect -y 1` (anything at
     `0x27`?) and `dmesg | grep -iE "s3nrn|s3fwrn|nfc|i2c"`, then try an
     NDEF tag read. Report probe failure modes precisely.
   - Note the series' DTS uses PM7550 LDO20 for `pvdd-supply` and
     `RPMH_LN_BB_CLK2` as the clock; check the mapping against this
     tree's actual regulator/clock labels rather than assuming.
   - Gen 6 vs Gen 6+: one reviewer reporting NFC dead was on a 6+.
     This phone is a Gen 6; record which you're testing on.
   - Integration gap (checked 2026-09-26): neither `milos-mainline/linux`
     (`milos-7.2.y`, `v7.2.0-milos`: FP6 DTS still has only the
     `/* Samsung NFC @ 0x27 */` comment, no `s3nrn4v` in the driver) nor
     pmaports `main` (`# CONFIG_NFC is not set`) carries the series. So
     nobody running pmOS on an FP6 gets tag reading today. Once tested on
     hardware, the useful output is: a `Tested-by` reply on the series,
     and an offer to the fork/pmaports maintainer (Luca Weiss) to carry
     it plus the Kconfig change (`NFC=m`, `NFC_NCI=m`, the driver as `m`;
     `NFC=y` is impossible here, see `patches/nfc/NOTES.md`) until it
     lands upstream. Ask them first; they may already have plans.

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
