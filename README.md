# Radxa Cubie A5E — ImmortalWrt

Independent ImmortalWrt bring-up and build research for the **Radxa Cubie A5E V1.20 (Allwinner A527, 1 GB RAM)**.

## Project handoff

Start with **[update.md](update.md)** for the latest CI results, IOMMU fix history, artifact inspection, 1 GB DRAM cautions, and next actions.

## Current phase

This repository currently has a successful **kernel/device-tree validation workflow**. It is not yet a complete firmware build and its artifacts are not flashable firmware.

The initial validation is intended to confirm that the pinned ImmortalWrt source can configure and compile the ARM64 sunxi kernel and produce the Cubie A5E device tree.

## Source

- Upstream: [immortalwrt/immortalwrt](https://github.com/immortalwrt/immortalwrt)
- Initial source revision: `4765b657625ec5bbda74b44400465fe043802b30`
- Target: `sunxi/cortexa53`
- Kernel series in the pinned source: Linux 6.18

The source revision is pinned for reproducibility. Update it deliberately after reviewing upstream changes.

## Hardware

- Board: Radxa Cubie A5E V1.20
- SoC: Allwinner A527
- CPU: ARM Cortex-A55, 64-bit
- RAM: 1 GB (user-confirmed)

## Safety

- Do not write boot0/SPL, TF-A, U-Boot, SPI flash, or DRAM timing data as part of early experiments.
- Do not treat a kernel-validation artifact as a complete image.
- Do not flash any image until its partition layout, boot script, FIT metadata, device tree, root arguments, and bootloader compatibility have been independently checked.
- Preserve the user's working ImmortalWrt/OpenWrt card and original image files.

## Planned stages

1. Validate kernel configuration, kernel compilation, and A5E DTB generation.
2. Inspect the official Radxa/Allwinner Tina SDK and existing A5E boot chain.
3. Compare the user's known-working ImmortalWrt image with the generated kernel and device tree.
4. Design a board-specific image profile and root filesystem.
5. Build and inspect a complete image before considering any test flash.

## Artifacts

Successful validation runs upload a short-lived GitHub Actions artifact named `cubie-a5e-kernel-validation` containing the Linux `Image`, A5E DTB, and checksums.
