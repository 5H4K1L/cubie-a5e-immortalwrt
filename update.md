# Project update / AI handoff

**Last reviewed:** 2026-11-05 (UTC)  
**Board:** Radxa Cubie A5E V1.20, Allwinner A527 / sun55i, 1 GB RAM  
**Related project:** [OpenWrt companion repository](https://github.com/5H4K1L/cubie-a5e-openwrt)

> Start here when resuming. Highest-priority active task: resolve the Linux 6.18 Sun55i IOMMU callback mismatch and obtain a passing kernel/DTB validation run. This repository currently validates kernel/device-tree compilation only; it does not yet create a complete firmware image.

## Goal and safety constraints

Validate and develop ImmortalWrt support for the Radxa Cubie A5E 1 GB board, ultimately producing a reproducible, board-specific microSD image.

- Preserve the working ImmortalWrt microSD and original image files.
- Do not write SPI, Boot0/SPL, TF-A, U-Boot, or DRAM timing data.
- Do not apply 2 GB/4 GB DRAM settings to this 1 GB board.
- Do not flash current CI artifacts: they are kernel/DTB validation outputs, not firmware.
- Treat all undocumented boot offsets, DRAM clocks/geometry, and FIT load/entry addresses as unknown until verified.

## Hardware / baseline

- Board: Radxa Cubie A5E V1.20; SoC Allwinner A527.
- Device tree compatible strings: `radxa,cubie-a5e` and `allwinner,sun55i-a527`.
- RAM: 1 GB (user-confirmed); eight Cortex-A55 cores.
- Current working installation: ImmortalWrt sunxi/cortexa53, Linux 6.18.34 AArch64.
- Boot partition: FAT `/dev/mmcblk0p1` (20 MB); root: ext4 `/dev/mmcblk0p2`.
- U-Boot version string: `2026.01_armbian-2026.01-S127a-Pd40e-Hc6a9-V2b7c-Bd0d2-R448a`.
- Working boot script loads FAT uImage, obtains root PARTUUID with `part uuid`, and invokes `bootm`.
- Working FIT uses `config-1`, LZMA kernel and uncompressed A5E DTB. Numeric load/entry addresses remain unknown.

## Source and current workflow

- Repository: [5H4K1L/cubie-a5e-immortalwrt](https://github.com/5H4K1L/cubie-a5e-immortalwrt), default branch `main`.
- Workflow: `.github/workflows/kernel-validation.yml`.
- Pinned upstream: `immortalwrt/immortalwrt` commit `4765b657625ec5bbda74b44400465fe043802b30`.
- Target: `sunxi/cortexa53`; kernel source built by CI is Linux 6.18.60.
- Kernel config fragment: `build/cubie-a5e-kernel.config`.
- Workflow appends the fragment to `target/linux/sunxi/cortexa53/config-6.18`, runs `make defconfig`, builds host tools/toolchain, then `make -j2 target/linux/compile V=s`.
- IOMMU compatibility patch: `build/patches/950-sun55i-iommu-attach-dev-linux-6.18.patch`.
- Intended successful artifact: Linux `Image` and `sun55i-a527-cubie-a5e.dtb`, uploaded as `cubie-a5e-kernel-validation`.

## IOMMU issue: root cause and intended correction

The original kernel compilation error was:

```text
drivers/iommu/sun55i-iommu.c:1492:43: error:
initialization of 'int (*)(struct iommu_domain *, struct device *,
struct iommu_domain *)'
from incompatible pointer type
'int (*)(struct iommu_domain *, struct device *)'
[-Wincompatible-pointer-types]
```

The pinned ImmortalWrt source contains the driver at:
`target/linux/sunxi/files/drivers/iommu/sun55i-iommu.c`.

Its callback currently starts as:

```c
static int sun55i_iommu_attach_dev(struct iommu_domain *domain,
                                   struct device *dev)
```

Linux 6.18 changed `iommu_domain_ops.attach_dev` to accept the previous domain as a third argument. The callback must therefore accept:

```c
static int sun55i_iommu_attach_dev(struct iommu_domain *domain,
                                   struct device *dev,
                                   struct iommu_domain *old)
```

The existing driver tracks its prior hardware domain through `active_domain` while holding `domain_lock`, and uses that state for rollback if hardware initialization fails. The minimal compatibility patch preserves that existing logic and explicitly marks the new `old` parameter unused. Do not cast the callback pointer, disable IOMMU, or discard the existing rollback logic without a reasoned driver-level redesign.

Upstream API-change references:
- [Linux IOMMU attach_dev old-domain change](https://lore-kernel.gnuweeb.org/all/20261013112930.896254953%40linuxfoundation.org/)
- [Patch discussion and API rationale](https://lists.openwall.net/linux-kernel/2025/10/13/1)

## CI history / errors observed (2026-11-05)

### Run #1 — original kernel failure

[Workflow run #1](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41977839204)

- Checkout, dependency installation, A5E configuration, host tools, and ARM64 toolchain all completed successfully.
- Kernel compilation failed at `drivers/iommu/sun55i-iommu.c:1492` with the callback type mismatch shown above.
- Build then failed in `drivers/iommu/sun55i-iommu.o`, propagating to `target/linux/compile`.
- Run #2 was a repeat of the same initial workflow revision and also failed; no distinct root cause was established for it.
- Some GCC/toolchain make messages in the log are marked `Error 1 (ignored)`; they were not the terminal build failure. The concrete fatal blocker was the IOMMU callback type mismatch.

### Run #3 — first patch application attempt

[Workflow run #3](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41989178917)

- Configuration completed.
- Patch application failed: `Hunk #1 FAILED at 759`.
- `1 out of 1 hunk FAILED -- saving rejects to file target/linux/sunxi/files/drivers/iommu/sun55i-iommu.c.rej`.
- Cause: patch context/whitespace did not match the exact source text.

### Runs #4 and #5 — intermediate patch attempts

- [Run #4](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41989205076): patch-application step passed at the time it was inspected; full kernel result was not yet confirmed in the retrieved status.
- [Run #5](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41989214709): patch-application step passed at the time it was inspected; full kernel result was not yet confirmed in the retrieved status.
- Do not treat either as a successful kernel build without checking the final workflow conclusion.

### Run #6 — later patch attempt

[Workflow run #6](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41993789031)

- Patch-application step was reported successful in the latest inspected job snapshot.
- Full build status had not yet been confirmed at that snapshot.

### Run #7 — malformed patch file

[Workflow run #7](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41993795193)

- Checkout, dependency installation, and A5E configuration succeeded.
- Patch step failed before compilation with:
  `patch: **** malformed patch at line 19`.
- The committed patch was malformed: its hunk line counts did not match the body, and the tab indentation had been written as literal backslash-t text in the patch.
- Host tools, toolchain, kernel compile, artifact collection, and upload were skipped because patch application failed.

## Latest correction applied

The patch file was corrected on `main` in commit:

`181496d13895d517a8bc13abbd85c62eeafc74ea`

Changes:
- Replaced the malformed hunk with a correctly counted unified diff.
- Used actual tab characters in the patch context.
- Updated the callback to accept `struct iommu_domain *old`.
- Added `(void)old;` after the local declarations, preserving the existing active-domain/rollback implementation and avoiding a declaration-after-statement style issue.
- The corrected patch was dry-run tested against a local fixture containing the exact source function prefix: `patch --dry-run -p1` passed and the patch applied successfully.

**Important:** this local fixture test is not a substitute for applying the patch to the complete pinned upstream source in CI. The next required verification is a new GitHub Actions run: confirm the patch step passes, host tools/toolchain pass, and the Linux target compiles through DTB collection. Record the resulting run URL and any new first fatal error here.

## Next steps

1. Check the new workflow run triggered by commit `181496d13895d517a8bc13abbd85c62eeafc74ea`.
2. If the patch step fails, inspect the exact patch error and source context before changing the patch again.
3. If patch application passes, inspect the complete kernel build log for the next fatal compiler/linker error. Do not assume the IOMMU issue is solved until the kernel compile step succeeds.
4. After successful kernel compilation, verify that both `Image` and `sun55i-a527-cubie-a5e.dtb` are collected and checksummed.
5. Continue full-image and boot-chain research only after kernel/DTB validation passes.

## Working OpenWrt comparison

The companion OpenWrt repository is [cubie-a5e-openwrt](https://github.com/5H4K1L/cubie-a5e-openwrt). Its previously observed kernel-validation run #3 passed kernel compilation and artifact collection on branch `build/initial-a5e-workflow`, with pinned source revision `b455c5b8514f2a6de0e3c68276771f953bddb046`. Compare its kernel configuration and source patches when useful, but do not assume OpenWrt and ImmortalWrt have identical IOMMU APIs or patch stacks.

## Important: not flashable

Neither the successful OpenWrt validation artifact nor a future ImmortalWrt kernel/DTB artifact is a complete image. Do not flash until the FIT, boot script, rootfs, partition layout, kernel load/entry addresses, A5E DTB, root arguments, and bootloader compatibility are verified on a separate microSD card.
