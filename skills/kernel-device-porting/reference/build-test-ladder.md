# Build-test ladder (no hardware needed)

Each rung proves something specific. Report which rungs passed; none of
them proves runtime behavior.

## 1. Kconfig resolves as intended

```
scripts/config --module CONFIG_FOO --enable CONFIG_BAR
make ARCH=arm64 olddefconfig
grep -E '^CONFIG_(FOO|BAR)=' .config     # verify, don't assume
```
`olddefconfig` never errors; it silently:
- caps a value by its dependencies (`depends on RFKILL || !RFKILL`
  with `RFKILL=m` forces `m`; `y` is impossible),
- drops options whose `depends on` is unmet, and anything only reachable
  via their `select`.
If an option vanishes, read its `Kconfig` entry for `depends on`.

## 2. Devicetree compiles

```
make ARCH=arm64 <vendor>/<board>.dtb     # path relative to arch/arm64/boot/dts/
```
Proves syntax and that every phandle/macro resolves.

## 3. Schema validation (what DT reviewers run)

Needs `dtschema` (`pip install dtschema`; if `pylibfdt` won't build,
install the distro's prebuilt libfdt python package and
`pip install --no-deps dtschema` plus `jsonschema ruamel.yaml rfc3987`).
```
make dt_binding_check DT_SCHEMA_FILES=<path/to/binding>.yaml
make CHECK_DTBS=y <vendor>/<board>.dtb
```
Proves the binding and its example are valid, and the board node
matches the binding (required properties present, no unknown ones).

## 4. Driver compiles

```
make ARCH=arm64 modules_prepare
make ARCH=arm64 W=1 M=drivers/<subsys>/<driver> modules
```
Read the `CC [M]` lines and any warnings. The final `modpost`
"Module.symvers is missing / undefined" errors are expected for an
isolated directory build (no full-kernel symbol table) and say nothing
about the code. A full kernel build removes them but is heavy; only do
it when you need an actual bootable image.

Disable unrelated heavy features for a test build only (e.g.
`CONFIG_DEBUG_INFO_BTF` pulls in a BPF host toolchain) — never carry
that into a real config change.

## 5. Hardware (the only rung that answers runtime questions)

Probe → `dmesg | grep -i <driver>`; bus presence → `i2cdetect -y <bus>`;
functional test of the actual feature; repeat across driver reload and
cold boot. Record the exact device variant. A good result on a series
under review is worth a `Tested-by` reply to the latest version.
