# Linux 6.18 sun55i IOMMU build failure — 2026-11-05

## Original failure

The initial kernel-validation run [#2](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41977944553) completed host-tools and ARM64 toolchain installation, then failed while compiling the Linux 6.18.60 kernel.

The first fatal compiler diagnostic was:

```text
drivers/iommu/sun55i-iommu.c:1492:43: error:
initialization of 'int (*)(struct iommu_domain *, struct device *,
struct iommu_domain *)' from incompatible pointer type
'int (*)(struct iommu_domain *, struct device *)'
```

The failure is a Linux IOMMU callback API mismatch in the downstream Allwinner Sun55i IOMMU driver included by the pinned ImmortalWrt source. It is not evidence of an A5E DRAM, board-revision, or hardware problem.

## Compatibility change

The build-only patch in [`build/patches/0001-sun55i-iommu-linux-6.18-attach-dev-api.patch`](../build/patches/0001-sun55i-iommu-linux-6.18-attach-dev-api.patch) updates `sun55i_iommu_attach_dev()` to accept the third `old_domain` callback argument required by the Linux 6.18 API.

The driver already tracks its active hardware domain internally and uses that state for rollback if hardware initialization fails. The new callback argument is therefore accepted but intentionally unused in this minimal compatibility change; the existing rollback behavior is retained.

The workflow applies this patch to the pinned upstream source before building. The patch-application step succeeded in the corrected rerun.

## Corrected validation run

- Branch: `fix/linux-6.18-sun55i-iommu-api`
- Corrected run: [GitHub Actions run #41989214709](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41989214709)
- Patch application: passed
- Host tools / toolchain: running at the time this note was written
- Kernel compile and artifact upload: pending

A successful compile is still only kernel/DTB validation. It does not create a flashable firmware image. Do not flash or modify the working card, SPI bootloader, or DRAM settings based on this build alone.
