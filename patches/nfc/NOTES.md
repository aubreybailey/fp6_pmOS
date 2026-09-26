# NFC enablement — build-tested, not hardware-tested

## UPDATE 2026-09-26 — the patches below are superseded; the chip is NOT an S3FWRN5

Found via a user-supplied writeup plus the LKML threads (details from
fetched summaries, so verify against the actual patches before relying on
any specific value): the FP6 chip is a **Samsung S3NRN4V** — the "rn4v" in
Fairphone's vendor filenames was the answer, not a hint. Jorijn van der
Graaf (Catcrafts) has a series in review (v5 as of Aug 2026,
[thread](https://ratatoskr.run/oe-linux-nfc/2026/08/17399045/t)) adding a
`samsung,s3nrn4v` variant to the s3fwrn5 driver. Tag *reading* (reader
mode, ISO 14443-4) works on a Gen 6 with it; card emulation (HCE/SE) does
not, and payments will never come (bank-provisioned secure element +
Android/iOS-only wallets). Not merged; maintainers requested changes and
patch 6 (the FP6 DTS) waits on the binding landing via the qcom tree.

What this means for our drafts:
- **Our DTS node is wrong as written**: `samsung,s3fwrn5-i2c` won't drive
  this chip. The S3NRN4V differs: it ships working firmware behind a
  bootloader protocol the old driver doesn't implement, uses a different
  RF-calibration command (`DUAL_OPTION`), needs a `FW_CFG` byte at init
  for the 19.2 MHz reference, and needs its reference clock serviced.
- **Our hardware sourcing was right**: i2c1@0x27, IRQ gpio31 (rising
  edge), en gpio56, wake gpio7 all match the upstream series. That
  validates pulling wiring from Fairphone's GPL devicetree.
- **What we were missing** (per the series): `pvdd-supply` = PM7550
  LDO20 (the `vdd_ldo20` we'd noted but never mapped), and a clock,
  `RPMH_LN_BB_CLK2` from the PMK7635. Our unwired `NFC_CLK_REQ` question
  is answered: the clock matters. v4 reportedly dropped clock-gating as
  non-functional — check the final series for what it actually does.
- The Kconfig fix (NFC=m, NFC_NCI=m dependency) is still valid and
  needed; the RFKILL cap finding stands.

**Don't send our DTS patch upstream. Test the upstream series instead.**
Next step is applying Jorijn's driver + binding + DTS patches to
`v7.2.0-milos` and build-testing those (our toolchain notes in
`skills/postmarketos-dev/reference/setup.md` apply as-is). Get the series
from lore/ratatoskr, not from a summary. Our patch files stay here as
history and as the record of the Kconfig findings.

Two patches, two different upstream targets:

- `0001-arm64-dts-qcom-milos-fairphone-fp6-Add-NFC-node.patch` — against
  `arch/arm64/boot/dts/qcom/milos-fairphone-fp6.dts` in
  [milos-mainline/linux](https://github.com/milos-mainline/linux) (the
  kernel fork `linux-postmarketos-qcom-milos` in pmaports builds).
- `0001-linux-postmarketos-qcom-milos-enable-NFC-S3FWRN5-dri.patch` —
  against pmaports' kernel config fragment
  (`device/testing/linux-postmarketos-qcom-milos/config-postmarketos-qcom-milos.aarch64`).
  Enables `CONFIG_NFC`, `CONFIG_NFC_NCI`, `CONFIG_NFC_S3FWRN5`,
  `CONFIG_NFC_S3FWRN5_I2C` (all `=m`).

## Build-test results (2026-09-03)

Actually built against the real pinned kernel source
(`github.com/milos-mainline/linux` tag `v7.2.0-milos`, the exact tarball
`linux-postmarketos-qcom-milos`'s APKBUILD fetches), inside the Termux
`proot-distro` Alpine container. Not a paper review — ran the real
toolchain and caught two real bugs the diff alone didn't show:

- **`CONFIG_NFC=y` is impossible on this defconfig.** `net/nfc/Kconfig`
  has `depends on RFKILL || !RFKILL`, and this defconfig has `RFKILL=m`
  elsewhere, which caps NFC at `=m`. `olddefconfig` doesn't error on an
  impossible value — it silently downgrades it. Only shows up by checking
  the resulting `.config`, not by reading the patch.
- **`CONFIG_NFC_S3FWRN5_I2C` depends on `NFC_NCI && I2C`.** `NFC_NCI`
  wasn't enabled, so the whole option was invisible and got silently
  dropped, taking `CONFIG_NFC_S3FWRN5` down with it (only reachable via
  `S3FWRN5_I2C`'s `select`). Fixed by adding `CONFIG_NFC_NCI=m`.

Both are now fixed in the patch. After the fix, `olddefconfig` resolves
cleanly to exactly the four options intended, no drops, no downgrades.

**Device tree**: compiles clean — `make ARCH=arm64 ... qcom/milos-fairphone-fp6.dtb`
produces a real `.dtb`, zero warnings or errors. Confirms every phandle
reference in the patch (`&tlmm`, `&i2c1`, `IRQ_TYPE_EDGE_RISING`,
`GPIO_ACTIVE_HIGH`, the two new pinctrl states) resolves correctly.

**Driver module compile**: completed. `core.c`, `firmware.c`, `nci.c`,
`phy_common.c`, `i2c.c` all compile clean (`CC [M]`, no errors) against
this kernel version. Getting there required three unrelated
build-environment fixes in the Alpine container/kernel source (missing
`tools/` uapi header, missing versioned LLVM binutils symlinks, BTF
pulling in an unrelated BPF toolchain) — all now documented in the
`postmarketos-dev` skill's `setup.md` so they don't need rediscovering.
Final step (`modpost`) reports undefined symbols and a missing
`Module.symvers` — expected and not a real problem: that file only gets
built by a full `vmlinux`/`modules` pass over the entire kernel, which a
targeted single-directory build deliberately skips (avoided here given
this device's real memory pressure during builds). The compile step is
the meaningful signal, and it's clean.

**Toolchain note for next time**: this container's default `clang22`
package can't link the kernel's own Kconfig host tools under Alpine's musl
(`undefined symbol: __errno`, `__assert2` — looks like an Alpine clang22
packaging issue, not a kernel bug). Fix: installed `clang21`/`lld21`
instead and built with `LLVM=-21 LD=ld.lld`. Worth checking if this is
fixed in a later Alpine clang22 point release before repeating the swap.

## Where the hardware wiring came from

Fairphone publishes kernel devicetree sources for GPL compliance on
`gerrit-public.fairphone.software`. The GPIO numbers in the DTS patch (i2c
address 0x27, en-gpio 56, wake/firm-gpio 7, irq-gpio 31, clk-req-gpio 6)
are copied directly from their `nfc-devicetree` repo,
`samsung/volcano-nfc-samsung-sec-slsi.dtsi` — **not guessed**. "Volcano" is
Qualcomm's internal board codename for this SoC; it's the same hardware
pmOS calls "milos" in its own kernel fork.

Clone it yourself with:
```
git clone -b odm/rc/target/15/fp6 https://gerrit-public.fairphone.software/platform/vendor/qcom/proprietary/nfc-devicetree
```
Full repo list for other subsystems (camera, bt, display, etc — same
pattern) is at
https://code.fairphone.com/projects/fairphone-gen-6/kernel.html

## Open risk — read before trusting this patch

The DTS patch uses `compatible = "samsung,s3fwrn5-i2c"`, mainline's
existing driver for Samsung's S3FWRN5 NFC chip. But Fairphone's own vendor
driver source (`vendor/samsung_slsi/nfc`, same gerrit host) has config
filenames referencing **"rn4v"** — which reads as a newer Samsung NFC chip
generation than S3FWRN5. The transport-level NCI protocol tends to stay
similar across generations, so this patch might well work as-is, but that
is genuinely unconfirmed. **First thing to check once this boots on real
hardware**: does the chip survive probe/`CORE_RESET` under the S3FWRN5
driver, or does `dmesg` show a mismatch that means a different/newer
driver is needed.

Secondary unknowns, lower stakes:
- IRQ trigger type (`IRQ_TYPE_EDGE_RISING`) is a datasheet-typical
  assumption, not confirmed for this specific unit.
- `NFC_CLK_REQ` (gpio6) has no equivalent property in the mainline
  binding — pinctrl state is defined but unused by the driver. Might not
  matter; might mean something's missing.

## Status

Drafted 2026-09-03, build-tested same day (DTS compile clean, Kconfig
chain resolves cleanly after fixing the two bugs above). Not yet flashed
to real hardware, not yet submitted upstream anywhere.
