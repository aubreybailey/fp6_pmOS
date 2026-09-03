# Package build workflow

## The core command

```
proot-distro login alpine -- sh -c 'cd <pkgdir> && abuild -F -r'
```

- `-F` forces running as root — required, see `setup.md` for why the normal
  builder-user path is broken in this environment. Not a hack around a
  working system; it's the correct workaround for a real proot limitation.
- `-r` auto-installs/removes makedepends via the system's own `apk` (works
  fine as root — no elevation needed).

Verified working end-to-end on a real test package: dependency install,
build, strip, package, checksum, sign, index, and `apk add` + run the
result. This is the real abuild pipeline, not a simulation.

## Gotcha: output path depends on $HOME at build time

Building as root (not as the `builder` user) puts output under
`/root/packages/<pkgname-or-repo>/aarch64/`, not
`/home/builder/packages/...` — even if `PACKAGER_PRIVKEY` points at a key
under `/home/builder/.abuild/`. Check both if a built `.apk` seems to have
vanished; easiest is `find / -name '*.apk' -newer APKBUILD 2>/dev/null`.

## Getting pmaports itself

```
git clone https://gitlab.postmarketos.org/postmarketOS/pmaports.git /root/pmaports
```

**Not** `gitlab.com/postmarketOS/pmaports` — that one looks identical and
clones fine, but it's a frozen archive pointing at a Nov 2024 commit
(postmarketOS migrated their canonical repo off gitlab.com then; the old
repo's last commit is literally about that migration). Wasted a clone
finding this out — always confirm you're at
`gitlab.postmarketos.org`, and sanity check with `git log -1 --format=%ci`
after cloning to make sure it isn't a year stale.

Already cloned to `/root/pmaports` in the `alpine` container as of
2026-09-02 (`git pull` there to update, don't reclone).

Individual package directories each have their own `APKBUILD` and build the
same way as the test package above. For this phone specifically:

- `device/testing/device-fairphone-fp6/` — the device package itself.
- `device/testing/firmware-fairphone-fp6/` — firmware blobs.
- Kernel is **not** a per-device package here — FP6 depends on
  `linux-postmarketos-qcom-milos`, a shared kernel package for the
  Qualcomm "milos" SoC family (find it under `device/` or `main/` by
  searching for that name — it's the one to patch for kernel-level work,
  not a `linux-fairphone-fp6` package, which doesn't exist).

## What does NOT work here, and don't waste time retrying it

- **`pmbootstrap`** — postmarketOS's real build orchestrator. It shells out
  to genuine `chroot`, bind-mounts, and sometimes loop devices to assemble
  bootable rootfs/boot images. None of that is available without real
  kernel namespaces/root. Don't try to install/run it in Termux.
- Anything needing `unshare`, real `mount --bind`, or loop devices
  (`losetup`) — confirmed broken (`unshare: Invalid argument`, no
  `unprivileged_userns_clone`).

## Recommended split with the PC side

- **Here (phone, Termux)**: write/edit APKBUILDs, patch kernel configs,
  build and sanity-test individual packages natively (aarch64-on-aarch64,
  no cross-compile/QEMU needed — an actual advantage over an x86 dev
  machine). Push commits/patches to a branch or MR.
- **PC (with pmbootstrap installed normally)**: pull those changes, use
  `pmbootstrap` to assemble the actual bootable image, and `fastboot` to
  flash it — see `flashing.md`.

## In-progress work

`~/pmos-fp6-nfc/` — first real porting attempt, started 2026-09-03: an NFC
device-tree patch for `milos-fairphone-fp6.dts` (not yet upstream-submitted)
plus the matching kernel config commit already applied in the local
`/root/pmaports` clone (inside the alpine container). Contains:
- `0001-*.patch` — the DTS patch itself.
- `milos-fairphone-fp6.dts.patched` — full file with the patch applied, for
  reference/rebasing.
- `reference-fairphone-nfc-devicetree/`, `reference-fairphone-nfc-driver/` —
  Fairphone's own published GPL sources this patch's GPIO wiring was taken
  from (not guessed) — see `code.fairphone.com/projects/fairphone-gen-6/`
  for how to find these repos on `gerrit-public.fairphone.software` for
  other subsystems too (camera-devicetree, bt-devicetree, etc. — same
  pattern, real hardware wiring straight from the source rather than
  reverse-engineering blind).
- Known open risk: vendor driver config filenames reference "rn4v", which
  reads as a newer Samsung NFC chip generation than S3FWRN5 (the mainline
  driver target used in the patch). Needs real-hardware testing to confirm
  protocol compatibility — this is the first thing to check once it boots.

## Resource notes

At last check: 157GB free disk (plenty for multiple package builds and a
pmaports clone). RAM is 7.2GB total but often under real pressure from
Android itself (as little as 625MB free / swap near full even at idle) —
watch memory during large builds (e.g. anything pulling in llvm/clang as a
dependency); consider `free -h` before kicking off a big one.
