# Fairphone 6 postmarketOS status

Last verified: 2026-09-03, against the actual kernel source pmOS builds
(`v7.2.0-milos` tag) and recent dev logs — not secondhand blog posts, which
go stale fast on a port this actively developed. Re-verify the same way
before trusting this if it's been more than a few weeks.

## Confirmed working

Checked directly in the pinned kernel tag's device tree/defconfig, and/or
reported in Aug 2026 dev logs (linmob.net weekly updates, postmarketos.org
blog):

- WiFi + Bluetooth (WCN6755, `ath11k`) — DT nodes present, driver enabled
  in defconfig, matching upstream mainline patches merged 2026-08-31.
- Display/GPU driver framework enabled (`CONFIG_DRM_MSM`, DPU, DSI).
- All three cameras (main/wide/telephoto).
- Audio/mic.
- VoLTE (at least emergency calling, tested end-to-end).
- Fingerprint unlock (implementation currently bit-banged i2c rather than
  routed through the SLPI coprocessor — flagged by a dev as suboptimal,
  works but not the "right" long-term implementation).
- Offline charging mode.

## Confirmed present at the kernel-config level (likely working, not
independently confirmed in a dev log)

- Mobile data / SMS infra: `CONFIG_QRTR`, `CONFIG_RMNET`,
  `CONFIG_QCOM_IPA`, `CONFIG_QCOM_Q6V5_PAS` (modem remoteproc loader) all
  enabled, `remoteproc_mpss` has the FP6-specific modem firmware path
  wired in.

## Confirmed NOT working / not implemented

- **NFC**: `CONFIG_NFC` disabled entirely, no device node (just a location
  comment in the DTS). Real hardware confirmed present and functional on
  the Android side (`adb shell dumpsys nfc` shows live firmware data).
  Draft patch in `../patches/nfc/` — not yet hardware-tested.

## Genuinely unverified (not checked either way)

- Non-emergency mobile data / SMS in practice (infra looks present, not
  independently confirmed working)
- General suspend/battery behavior day-to-day

## How to re-check this yourself instead of trusting old text

The pmOS wiki (`wiki.postmarketos.org`) is bot-walled (Anubis) and
unreliable to fetch. What actually works:

1. Find the exact kernel tag pmaports builds:
   `pmaports/device/testing/linux-postmarketos-qcom-milos/APKBUILD`, the
   `_tag="v$pkgver-milos"` line.
2. Pull the real DTS/defconfig straight from
   `github.com/milos-mainline/linux` at that tag (GitHub API or raw
   content) and grep for the subsystem in question.
3. Cross-reference Fairphone's own published GPL sources at
   `gerrit-public.fairphone.software` (see
   `code.fairphone.com/projects/fairphone-gen-6/kernel.html` for the repo
   list) for real hardware wiring — GPIOs, compatible strings, chip
   identity — instead of guessing.
4. `linmob.net/weekly-update-*` and `postmarketos.org/blog` for recent dev
   activity, checked against the current date.
