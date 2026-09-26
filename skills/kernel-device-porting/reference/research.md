# Research: existing work and hardware facts

## Where to look for existing work (in this order)

1. **The distro's pinned kernel.** Find the exact tag/branch the
   packaging builds (e.g. pmaports `APKBUILD` `_tag=`), then read the
   device DTS and defconfig *at that ref*. Placeholder comments like
   `/* Samsung NFC @ 0x27 */` mean someone identified the part and
   stopped.
2. **Other branches of the device kernel fork** (`gh api
   repos/<org>/<repo>/branches`, then read files with
   `contents/<path>?ref=<branch>`). Newer branches may already carry it.
3. **Mailing lists.** Search by chip name, compatible string, board
   name. lore.kernel.org's search, feeds and partial-message-id pages
   are behind a bot check; direct `/all/<msgid>/t.mbox.gz` works.
   Use instead:
   - **patchwork REST API** (not bot-walled, has search, versions,
     states, mbox):
     `https://patchwork.kernel.org/api/series/?q=<term>` → each series
     has `version`, `cover_letter.msgid`, `mbox`; patch states via
     `/api/patches/?series=<id>`.
   - ratatoskr.run mirrors lists with readable threads, but its diffs
     are split across HTML blocks: good for reading, bad for applying.
4. **Distro packaging MRs and porter blogs / weekly roundups** for
   runtime status. These go stale; always note their date.

States worth knowing: netdev patchwork `awaiting-upstream` means netdev
expects another tree (subsystem or SoC tree) to take it — not merged.

## Hardware facts from primary sources

- **Vendor GPL source.** Many OEMs publish kernel + devicetree repos
  (for Fairphone: `code.fairphone.com/projects/<device>/kernel.html`
  lists repos on `gerrit-public.fairphone.software`, directly
  `git clone`-able per repo). Qualcomm vendor trees use internal board
  codenames (FP6's SoC "milos" upstream = "volcano" in vendor trees);
  find the file by peripheral + vendor name, not by device name.
- **Vendor node → mainline node translation.** Vendor properties
  (`sec-nfc,ven-gpio`, `pmic-ldo = "vdd_ldo20"`) map to mainline
  binding properties (`en-gpios`, `pvdd-supply = <&vreg_l20b>`).
  Don't drop a vendor property just because the mainline binding lacks
  an obvious equivalent — the unwired clock-request line here turned
  out to be load-bearing.
- **Vendor userspace config names identify chips.** HAL config
  filenames and product enums often carry the exact part (`rn4v` →
  S3NRN4V). Check them before picking a compatible.
- **The running stock OS proves presence.** e.g. `adb shell dumpsys
  nfc`, `getprop`. `/proc/device-tree` below the root is usually
  SELinux-blocked without root.

## Verifying claims

Secondhand summaries (including a fetch tool's summary of a page) can
omit or distort specifics. Before relying on a register value, GPIO or
property, read it in the actual patch/source.
