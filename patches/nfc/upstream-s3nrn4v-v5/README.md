# Upstream S3NRN4V series v5, build-tested on `v7.2.0-milos`

Series: "[PATCH v5 0/6] nfc: s3fwrn5: support the S3NRN4V variant",
Jorijn van der Graaf, 2026-08-11,
`<20260811220135.472380-1-jorijnvdgraaf@catcrafts.net>`.
Patchwork series 1144349 (linux-arm-msm) / 1144348 (netdevbpf, all six
marked `awaiting-upstream`). Not merged anywhere as of 2026-09-26.

- `series-v5.mbox` — the six patches, from patchwork (apply source).
- `thread-v5.mbox` — full lore thread with review replies.
- `pre1-777fd2dab446.patch`, `pre2-ce2d85e3d293.patch` — mainline
  commits (both 2026-08-11) the series is based on and `v7.2.0-milos`
  lacks. Only their `drivers/nfc/s3fwrn5/*` hunks are needed.

## How to apply to `v7.2.0-milos` (verified 2026-09-26)

```
# split series-v5.mbox into 0001..0006 (one message each), then in the tree:
git apply 0001.patch 0002.patch 0003.patch 0004.patch
git apply -C1 --include='drivers/nfc/s3fwrn5/*' pre1-777fd2dab446.patch
git apply -C1 --include='drivers/nfc/s3fwrn5/*' pre2-ce2d85e3d293.patch
git apply 0005.patch
git apply -C1 0006.patch   # milos tree has extra &tlmm nodes; only
                           # trailing context differs, block is identical
```
Config: `NFC=m NFC_NCI=m NFC_S3FWRN5=m NFC_S3FWRN5_I2C=m` via
`scripts/config` (NFC can't be `y` here, RFKILL=m caps it).

## Results

| check | result |
|---|---|
| Kconfig (`olddefconfig`) | all four options survive as `m` |
| `qcom/milos-fairphone-fp6.dtb` | compiles clean |
| `M=drivers/nfc/s3fwrn5 W=1` | all `.c` compile, zero warnings (modpost tail expected: no full-kernel `Module.symvers`) |
| `dt_binding_check` on `samsung,s3fwrn5.yaml` | passes |
| `CHECK_DTBS=y` on FP6 DTB | zero warnings |
| on hardware | **not yet** |

The DTS references (`vreg_l20b` = PM7550 LDO20, `rpmhcc`
`RPMH_LN_BB_CLK2`) resolve in this tree.

## Open review items (not verified by us)

From the automated Sashiko review of v5, unanswered in-thread:
- 5/6: `GET_VER` return value ignored (possible response desync on
  timeout); `DUAL_OPTION` errors checked as `< 0`, missing positive NCI
  status codes.
- 6/6: no pinctrl state for the wake GPIO (gpio7). Low severity.

Worth checking against real behavior during hardware testing rather than
taking the bot's word either way.

## Also needed at runtime

Calibration blobs at `samsung/s3nrn4v/hwreg.bin` and `swreg.bin` in the
firmware path (the cover letter says reads still work without them,
using the chip's stored calibration). Source is the vendor stack; the
linux-firmware question is open in the thread.

## For a Tested-by

Run on this Gen 6 under pmOS with this tree: reader-mode tag read, from
cold boot and across `rmmod`/`modprobe` of `s3fwrn5_i2c`, with and
without the calibration files. Reply to the v5 thread (or v6 if one
appears; check patchwork first) with results and which variant (Gen 6).
