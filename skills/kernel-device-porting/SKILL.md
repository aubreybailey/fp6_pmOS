---
name: kernel-device-porting
description: Workflow patterns for enabling hardware on a mainline/downstream Linux kernel for a specific device (phones, SBCs) - finding existing work, sourcing real hardware wiring, fetching and backporting patch series, and build-testing devicetree/Kconfig/driver changes without the hardware. Use when asked to enable a peripheral, write or test a DTS/Kconfig/driver patch, apply a mailing-list series, or figure out why a device feature doesn't work under Linux.
---

# Kernel device porting

Device-agnostic method. Environment-specific toolchain notes live
elsewhere (e.g. `postmarketos-dev` for Termux/proot/Alpine).

## The order that works

1. **Search for existing work before writing anything.** Mailing lists,
   the device's kernel fork branches, distro packaging MRs, porter blogs.
   Hardware that's "obviously easy" is often already done or in review.
   → `reference/research.md`
2. **Establish the real hardware facts from primary sources.** Vendor
   GPL devicetree/driver source for wiring; the running stock OS for
   "is this hardware present and working". Never guess GPIOs or
   compatibles; treat blog posts and summaries as leads, not facts.
   → `reference/research.md`
3. **Prefer testing and integrating others' series over writing your
   own.** Fetch the real patches, find their base, backport
   prerequisites, apply. → `reference/applying-series.md`
4. **Build-test up the ladder**, checking the actual result of each
   rung rather than trusting the diff. → `reference/build-test-ladder.md`
5. **Only hardware settles runtime questions.** Record exactly what
   was and wasn't verified, and on which device variant.

## Traps that cost real time

- Assuming a newer chip is compatible with the older driver because
  names look similar. Vendor file names (`rn4v`) were the answer, not
  a hint.
- Trusting a flat `.config` edit: `olddefconfig` silently downgrades
  impossible values and drops options with unmet dependencies.
- Treating a year-old status post as current. Check dates; verify
  against the source tree the distro actually builds.
- Reporting a build/toolchain failure as a patch problem (or the
  reverse). Isolate with a minimal repro before attributing blame.
