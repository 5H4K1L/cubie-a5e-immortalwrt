# Project update / AI handoff

**Last reviewed:** 2026-11-05 (UTC)  
**Board:** Radxa Cubie A5E V1.20, Allwinner A527 / sun55i, 1 GB RAM  
**Related project:** [OpenWrt companion repository](https://github.com/5H4K1L/cubie-a5e-openwrt)

> Start here when resuming. This repository currently validates kernel/device-tree compilation only. It does not yet create a complete firmware image.

## Goal and safety constraints

Validate and develop ImmortalWrt support for the Radxa Cubie A5E 1 GB board, ultimately producing a reproducible, board-specific microSD image.

- Preserve the working ImmortalWrt microSD and original image files.
- Do not write SPI, Boot0/SPL, TF-A, U-Boot, or DRAM timing data.
- Do not apply 2 GB/4 GB DRAM settings to this 1 GB board.
- Do not flash current CI artifacts: they are kernel/DTB validation outputs, not firmware.
- Treat all undocumented boot offsets, DRAM clocks/geometry, and FIT load/entry addresses as unknown until verified.

## Hardware / baseline

- Board: Radxa Cubie A5E V1.20; SoC Allwinner A527.
- Device tree compatible strings: radxa,cubie-a5e and allwinner,sun55i-a527.
- RAM: 1 GB (user-confirmed); eight Cortex-A55 cores.
- Current working installation: ImmortalWrt sunxi/cortexa53, Linux 6.18.34 AArch64.
- Boot partition: FAT /dev/mmcblk0p1 (20 MB); root: ext4 /dev/mmcblk0p2.
- U-Boot version string: 2026.01_armbian-2026.01-S127a-Pd40e-Hc6a9-V2b7c-Bd0d2-R448a.
- Working boot script loads FAT uImage, obtains root PARTUUID with part uuid, and invokes bootm.
- Working FIT uses config-1, LZMA kernel and uncompressed A5E DTB. Numeric load/entry addresses remain unknown.

## Source and current workflow

- Default branch: main.
- Workflow: .github/workflows/kernel-validation.yml.
- Pinned upstream: immortalwrt/immortalwrt commit 4765b657625ec5bbda74b44400465fe043802b30.
- Target: sunxi/cortexa53; build log reports Linux kernel source version 6.18.60.
- Kernel config fragment: build/cubie-a5e-kernel.config.
- Workflow appends the fragment to target/linux/sunxi/cortexa53/config-6.18, runs make defconfig, builds host tools/toolchain, then make -j2 target/linux/compile V=s.
- Intended artifact after successful compile: Linux Image and sun55i-a527-cubie-a5e.dtb, uploaded as cubie-a5e-kernel-validation.

## Latest CI result and exact blocker

[Workflow run #1](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41977839204) on 2026-11-05 failed.

Completed successfully:
- Checkout of this repository and pinned upstream source.
- Dependency installation.
- A5E kernel configuration and make defconfig.
- Host tools and ARM64 toolchain installation.

Failed:
- Step: Compile ImmortalWrt Linux target.
- First concrete compiler error in the job log:
  drivers/iommu/sun55i-iommu.c:1492:43: error:
  initialization of 'int (*)(struct iommu_domain *, struct device *, struct iommu_domain *)'
  from incompatible pointer type
  'int (*)(struct iommu_domain *, struct device *)'
  [-Wincompatible-pointer-types]
- The build then fails in drivers/iommu/sun55i-iommu.o, propagating to target/linux/compile.
- This is a C API/signature mismatch in the Sun55i IOMMU driver callback versus the iommu_ops definition in the pinned Linux source. It is not an apt/dependency or host-toolchain setup failure.
- Do not blindly cast the function pointer or disable IOMMU. Inspect the callback implementation, the struct iommu_ops definition and the upstream/vendor patch history; implement the correct signature and semantics for this exact kernel revision.

## Next steps

1. Inspect the exact source for drivers/iommu/sun55i-iommu.c in the pinned build tree and the corresponding struct iommu_ops callback declaration in that Linux 6.18.60 source.
2. Identify whether the driver patch is stale/backported incorrectly or whether the source tree has a partial API transition. Compare with upstream Linux/ImmortalWrt changes around this driver.
3. Make a minimal source patch in the project (prefer a reproducible patch file applied by the workflow, not an in-place sed edit). Preserve callback behavior and correct function signature/arguments; do not use unsafe casts.
4. Re-run the workflow and verify kernel compilation and DTB collection both pass. Then review the artifact and hashes.
5. Continue full-image and boot-chain research only after kernel/DTB validation passes.

## Working OpenWrt comparison

The companion OpenWrt repository's latest observed run [#3](https://github.com/5H4K1L/cubie-a5e-openwrt/actions/runs/41975643597) passed kernel compilation and artifact collection on branch build/initial-a5e-workflow. Its pinned OpenWrt source is b455c5b8514f2a6de0e3c68276771f953bddb046. Compare its kernel config and source patches, but do not assume the two source trees have identical IOMMU APIs.

## Important: not flashable

Neither the successful OpenWrt validation artifact nor a future ImmortalWrt kernel/DTB artifact is a complete image. Do not flash until the FIT, boot script, rootfs, partition layout, kernel load/entry addresses, A5E DTB, root arguments and bootloader compatibility are verified on a separate microSD card.
