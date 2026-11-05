# Full-image build and inspection gate

**Status:** workflow added; manual execution required. No firmware image from this workflow has been built or approved yet.

## Workflow

Run **A5E ImmortalWrt full-image inspection (UNTESTED)** from the Actions tab:

- Workflow: `.github/workflows/full-image-inspection.yml`
- Trigger: `workflow_dispatch` only. It does not run on push.
- Upstream source: `immortalwrt/immortalwrt` at `4765b657625ec5bbda74b44400465fe043802b30`.
- Device profile: `sunxi/cortexa53`, `radxa_cubie-a5e`.
- Build: complete ImmortalWrt device image, including the standard sunxi SD-card image recipe.
- Artifact: `cubie-a5e-UNTESTED-full-image-inspection`, seven-day retention.

The workflow applies the existing Linux 6.18 IOMMU callback compatibility patch and checks the expected PCIe MSI-X support. It then builds the selected board profile, validates the compressed SD-card image, records the partition table, and uploads the image plus boot-region/U-Boot/boot-script inspection files when found.

## Critical bootloader warning

The pinned upstream sunxi image generator writes the board-specific `u-boot-with-spl.bin` at byte offset `0x2000` (8 KiB). Therefore the generated SD-card image contains a newly built early boot chain. The pinned U-Boot package is version 2026.04 and its A5E target depends on the A523 TF-A build.

The user's board has 1 GB RAM. The upstream A5E U-Boot DRAM settings have not been validated against the exact DRAM part and PCB revision. The standard generated image is consequently **for CI inspection only**.

**Do not flash this artifact.** Before any hardware test:
1. Inspect the user's known-working image and identify its partition layout and early boot payload.
2. Compare the generated U-Boot/SPL/TF-A bytes with the known-working 1 GB image.
3. Establish whether the board boots early stages from microSD, SPI, or both.
4. Decide whether to reuse the verified early boot payload unchanged.
5. Review the generated FIT, boot script, partition UUIDs, rootfs, and kernel/DTB compatibility.
6. Test only on a separate microSD card with a recovery route.

## Known limitation

The current model's container/file-processing runtime has returned errors while attempting to inspect the user's uploaded image and archive. No content from those uploads has been extracted or copied into this repository. The workflow does not substitute for that comparison.

## Expected result

A successful GitHub Actions run proves that the pinned source can build the selected device image and that its output can be structurally inspected. It does **not** prove that the image boots or that DRAM, Ethernet, Wi-Fi, PCIe, USB, storage, or IOMMU works on the physical board.
