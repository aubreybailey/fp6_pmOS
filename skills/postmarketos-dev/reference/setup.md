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
