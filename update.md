# Project update / AI handoff

**Last reviewed:** 2026-11-05 (UTC)  
**Board:** Radxa Cubie A5E V1.20, Allwinner A527 / sun55i, 1 GB RAM  
**Default branch:** `main`  
**Related project:** [OpenWrt companion repository](https://github.com/5H4K1L/cubie-a5e-openwrt)

> Start here when resuming. The Linux 6.18 Sun55i IOMMU callback compile blocker has been corrected, and the latest ImmortalWrt kernel-validation workflow (#8) passed through kernel compilation, DTB collection, and artifact upload. The project has not yet built a complete firmware image.

## Goal and safety constraints

Develop reproducible ImmortalWrt support for the Radxa Cubie A5E 1 GB board, ultimately producing a board-specific microSD image.

- Preserve the working ImmortalWrt microSD and original image files.
- Do not write SPI, Boot0/SPL, TF-A, U-Boot, or DRAM timing data.
- Never apply 2 GB/4 GB DRAM settings to this 1 GB board.
- Do not flash current CI artifacts: they are kernel/DTB validation outputs, not firmware.
- Treat undocumented boot offsets, DRAM clocks/geometry, memory ranges, and FIT load/entry addresses as unknown until verified.

## Hardware / working baseline

- Board: Radxa Cubie A5E V1.20; SoC Allwinner A527.
- Compatible strings: `radxa,cubie-a5e`, `allwinner,sun55i-a527`.
- RAM: 1 GB (owner-confirmed); eight Cortex-A55 cores.
- Working installation: ImmortalWrt `sunxi/cortexa53`, Linux 6.18.34 AArch64.
- Boot partition: FAT `/dev/mmcblk0p1` (20 MB); root: ext4 `/dev/mmcblk0p2`.
- U-Boot version string: `2026.01_armbian-2026.01-S127a-Pd40e-Hc6a9-V2b7c-Bd0d2-R448a`.
- Working boot script loads FAT `uImage`, obtains root PARTUUID with `part uuid`, and invokes `bootm`.
- Working FIT uses `config-1`, LZMA kernel and uncompressed A5E DTB. Numeric load/entry addresses remain unknown.

## Source and current workflow

- Repository: [5H4K1L/cubie-a5e-immortalwrt](https://github.com/5H4K1L/cubie-a5e-immortalwrt), default branch `main`.
- Workflow: `.github/workflows/kernel-validation.yml`.
- Pinned upstream: `immortalwrt/immortalwrt` commit `4765b657625ec5bbda74b44400465fe043802b30`.
- Target: `sunxi/cortexa53`; CI kernel source version: Linux 6.18.60.
- Kernel config fragment: `build/cubie-a5e-kernel.config`.
- IOMMU patch: `build/patches/950-sun55i-iommu-attach-dev-linux-6.18.patch`.
- Workflow applies the patch to the checked-out source, builds host tools/toolchain, compiles `target/linux`, then collects the raw kernel `Image` and `sun55i-a527-cubie-a5e.dtb`.

## IOMMU compile blocker — resolved for compilation

### Original error

The first build failed because `sun55i_iommu_attach_dev()` had the old two-argument callback signature while the Linux 6.18 `iommu_domain_ops.attach_dev` function pointer expected three arguments:

```text
initialization of 'int (*)(struct iommu_domain *, struct device *,
struct iommu_domain *)'
from incompatible pointer type
'int (*)(struct iommu_domain *, struct device *)'
[-Wincompatible-pointer-types]
```

### Correction now in the repository

The pinned source's driver is located at:

`target/linux/sunxi/files/drivers/iommu/sun55i-iommu.c`

The compatibility patch changes the callback to:

```c
static int sun55i_iommu_attach_dev(struct iommu_domain *domain,
                                   struct device *dev,
                                   struct iommu_domain *old)
```

and marks `old` unused with `(void)old;`. The driver already serializes domain changes with `domain_lock`, tracks its hardware domain in `active_domain`, and uses that state to restore the previous translation-table base if hardware initialization fails. The patch preserves that existing logic; it does not cast the function pointer or disable IOMMU.

The workflow's explicit patch-application step now succeeds against the pinned source, and the subsequent Linux target compilation succeeds. This confirms the callback type mismatch no longer blocks this pinned Linux 6.18.60 build.

**Runtime caveat:** compilation does not prove that the driver's domain-switching behavior is correct at runtime on A527 hardware. The new `old` API argument is intentionally unused because this driver's existing `active_domain` state is used for rollback. Runtime IOMMU behavior still needs hardware testing if the device tree enables this IOMMU/client path.

## CI history — 2026-11-05

| Run | Commit / branch | Result | Finding |
|---|---|---|---|
| [#1](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41977839204) | `07f36a04`, main | Failure | Original attach_dev callback type mismatch |
| [#2](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41977944553) | `07f36a04`, main | Failure | Original validation attempt also failed |
| [#3](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41989178917) | `871b72cc`, fix branch | Failure | Patch hunk did not match source |
| [#4](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41989205076) | `d5dc11b2`, fix branch | Failure | Intermediate patch attempt |
| [#5](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41989214709) | `354c3e20`, fix branch | Success | Kernel validation passed on fix branch |
| [#6](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41993789031) | `0549f011`, main | Failure | Intermediate callback patch attempt |
| [#7](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41993795193) | `7320be21`, main | Failure | Malformed patch: hunk line count / tab formatting |
| **[#8](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41994608070)** | **`181496d1`, main** | **Success** | **Corrected patch, kernel compile, DTB collection, artifact upload all passed** |

### Latest successful run #8

- Run ID: `41994608070`.
- Commit: `181496d13895d517a8bc13abbd85c62eeafc74ea`.
- Branch: `main`.
- Result: completed / success.
- Patch step: passed.
- Host tools and ARM64 toolchain: passed.
- Linux target compilation: passed.
- Kernel and A5E DTB collection: passed.
- Artifact upload: passed.
- Artifact: `cubie-a5e-kernel-validation`, available and not expired when checked.
- GitHub-reported artifact archive size: 7,021,334 bytes.

The artifact contains the raw kernel `Image` and `sun55i-a527-cubie-a5e.dtb` collected by the workflow. No artifact contents or SHA-256 values have yet been independently inspected outside CI.

## What this result proves — and does not prove

It proves the corrected patch applies to the pinned ImmortalWrt source and that the Linux target compiles far enough for CI to collect and upload the expected kernel and DTB.

It does **not** prove:
- that the kernel boots on the physical A5E;
- that the IOMMU driver is functionally correct at runtime;
- that the generated DTB has the correct memory map after bootloader fixups;
- that a complete FIT, boot script, root filesystem, partitioned disk image, or flashable firmware has been produced;
- that Ethernet, storage, USB, Wi-Fi, thermal, reboot, or other peripherals work.

Do not flash the current artifact.

## Next steps — in order

1. Download the #8 `cubie-a5e-kernel-validation` artifact and inspect the kernel `Image` and A5E DTB. Record sizes, kernel version/config, DTB compatible strings, and SHA-256.
2. Compare the generated DTB against the known-working FIT's embedded DTB and inspect the runtime FDT/memory fixups. Do not invent memory nodes or RAM ranges.
3. Review the complete patch against the pinned driver's full callback implementation and Linux 6.18 API. Keep the minimal patch unless a semantic issue is demonstrated.
4. Compare the ImmortalWrt kernel/DTB/config with the successful OpenWrt validation artifact. Do not assume the two source trees are identical.
5. Inspect ImmortalWrt's sunxi image-generation pipeline and plan a board-specific FIT/rootfs/partition recipe matching the observed `bootm` flow.
6. Build a complete image in CI, inspect its partition table, boot files, FIT metadata, rootfs, PARTUUID behavior, and checksums.
7. Test only from a separate microSD card after static review. Keep the known-working card and SPI bootloader untouched.

## Companion OpenWrt status

[OpenWrt companion repository](https://github.com/5H4K1L/cubie-a5e-openwrt) passed kernel validation in [run #3](https://github.com/5H4K1L/cubie-a5e-openwrt/actions/runs/41975643597), branch `build/initial-a5e-workflow`, commit `ce996798ae6db98c8899c2958cdb8dcbbbe5d6be`. Its artifact is also kernel/DTB only, not a complete image.

## Important: not flashable

Neither repository has yet produced a complete, hardware-validated A5E firmware image. Do not flash a raw kernel/DTB artifact. Before any test boot, verify the FIT format, boot script, rootfs, partition layout, load/entry addresses, A5E DTB, root arguments, bootloader compatibility, and recovery plan. Test from a separate microSD card only.
