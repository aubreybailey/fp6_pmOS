# NFC enablement — draft, not hardware-tested

Two patches, two different upstream targets:

- `0001-arm64-dts-qcom-milos-fairphone-fp6-Add-NFC-node.patch` — against
  `arch/arm64/boot/dts/qcom/milos-fairphone-fp6.dts` in
  [milos-mainline/linux](https://github.com/milos-mainline/linux) (the
  kernel fork `linux-postmarketos-qcom-milos` in pmaports builds).
- `0001-linux-postmarketos-qcom-milos-enable-NFC-S3FWRN5-dri.patch` —
  against pmaports' kernel config fragment
  (`device/testing/linux-postmarketos-qcom-milos/config-postmarketos-qcom-milos.aarch64`).
  Enables `CONFIG_NFC`, `CONFIG_NFC_S3FWRN5`, `CONFIG_NFC_S3FWRN5_I2C`.

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

Drafted 2026-09-03. Not yet build-tested (needs the full kernel
toolchain), not yet flashed to real hardware, not yet submitted upstream
anywhere.
