# Setting up the Alpine build container

```
pkg install -y proot-distro
proot-distro install alpine
proot-distro login alpine
```

That's it for the container. Inside it, install real build tooling:

```
apk update
apk add alpine-sdk abuild git sudo
```

## Don't bother with the builder-user dance

The normal Alpine convention is: create an unprivileged `builder` user, add
it to the `abuild` group, and let `abuild -r` self-elevate via
`sudo`/`abuild-apk` to install build dependencies. **This fails here**:

```
abuild-apk: User builder is not a member of group abuild
```

even though `/etc/group` correctly lists it. Root cause (verified, not
guessed): this kernel has no user namespaces (`unshare --user` fails, no
`/proc/sys/kernel/unprivileged_userns_clone`), and proot only fakes uid/gid
*display* via passwd/group file lookups — it does not give the underlying
process (still the real, sandboxed Termux app UID) real kernel-level
supplementary groups. `su`'s `initgroups()` is effectively a no-op, so
`getgroups()` inside the container still returns the Android app's real
groups, not `abuild`. `abuild-apk` is a compiled helper that checks real
`getgroups()`, so no config file or env var fixes this.

**Just skip straight to building as fake-root** — see `workflow.md`. There's
no security boundary being protected here anyway (proot's fake-root already
has full access to the container).

## Signing key

Generate one once per container (as whichever user you'll build as — root
is fine given the above):

```
abuild-keygen -a -i -n
```

`-i` installs the pubkey into `/etc/apk/keys` so locally-built packages are
trusted for install without `--allow-untrusted`.

## Building an actual kernel (not just abuild-packaged userspace)

For build-testing a kernel patch directly against the real source (not
through abuild's full package flow), the default `clang22` in this
container's Alpine base **cannot link the kernel's own Kconfig host
tools**:

```
ld.lld: error: undefined symbol: __errno
ld.lld: error: undefined symbol: __assert2
```

This happens even for a from-scratch build with no patches applied — it's
an Alpine clang22-vs-musl packaging issue on this container, not anything
in pmOS's kernel source. Confirmed by isolating it: a trivial C program
using `assert()`/`errno` compiles fine with the same clang, so it's
specific to how the kernel's `scripts/kconfig/*.c` gets built.

**Fix**: swap to `clang21` (they can't coexist with `clang22` due to
`ld.lld` binary conflicts, so remove clang22 first):

```
apk del clang22 clang22-libs clang22-headers lld
apk add clang21 lld21 llvm21
```

Then build with `LLVM=-21 LD=ld.lld` instead of `LLVM=1`
(`LLVM=-21` selects the versioned `clang-21`; the plain `LLVM=1` picks
whatever `clang`/`ld.lld` are unversioned-default, which after this swap
is 21, but being explicit avoids resurrecting the same bug if clang22 ever
gets reinstalled as a dependency of something else).

Worth checking whether a later Alpine clang22 point release fixes this
before repeating the swap in a fresh container.

## Testing a single .dts file without a full kernel build

Don't run a full `make` for validating a devicetree patch — compiling one
target is fast and sufficient:

```
# needs a .config present first (any valid one; copy the device's
# defconfig in), then:
make ARCH=arm64 LLVM=-21 LD=ld.lld <subdir>/<name>.dtb
```

The target path is relative to `arch/arm64/boot/dts/` — e.g. for
`arch/arm64/boot/dts/qcom/milos-fairphone-fp6.dts`, the target is
`qcom/milos-fairphone-fp6.dtb`, **not** the full path (Kbuild will
double-prefix it) and not just the bare filename (no rule for that).

A config change gets properly validated with `scripts/config` (not by
hand-editing `.config` and hoping `olddefconfig` respects it silently —
see the RFKILL/NFC_NCI story in `../../patches/nfc/NOTES.md` for why
hand-set values can get silently downgraded or dropped without any error):

```
scripts/config --enable CONFIG_FOO --module CONFIG_BAR
make ARCH=arm64 LLVM=-21 LD=ld.lld olddefconfig
grep CONFIG_FOO .config   # confirm it survived, don't assume it did
```
