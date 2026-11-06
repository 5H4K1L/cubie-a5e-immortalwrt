# Radxa Cubie A5E reference files

This directory contains reference material extracted from the known-working
`Immortal-1GB-DRAM-A5E-LuCi-XLR8.img` image.

## Contents

- `Cubie-A5E-1GB-Bootloader-Extraction.zip` — extracted boot-chain package.
- `uboot_files.zip` — additional U-Boot reference files.
- `EXTRACTION_REPORT.md` — extraction findings, image layout, boot flow, hashes and caveats.
- `SHA256SUMS` — supplied checksum list.
- `BOOTCHAIN-INTEGRATION-DECISION.md` — project integration decision and safety checklist.

## Important

The report recommends preserving the integrated SPL + U-Boot FIT bootloader as a
single unit. It does not prove that the SPL is specific to the 1 GB board variant
or recover the exact DRAM timing parameters. No file here is approved for flashing
without independent binary verification and hardware-specific validation.

The ZIP archives are uploaded, but binary contents still need to be independently
inspected and re-hashed before they are used by the build.
