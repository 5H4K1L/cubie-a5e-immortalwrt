# Known-working boot-chain reference and integration decision

**Scope:** Radxa Cubie A5E V1.20, owner-confirmed 1 GB DRAM  
**Reference source image:** `Immortal-1GB-DRAM-A5E-LuCi-XLR8.img`  
**Evidence source:** `EXTRACTION_REPORT.md` uploaded with this branch  
**Status:** analysis only; no bootloader has been installed into the build and no firmware has been approved for flashing.

## Decision

Preserve the complete integrated bootloader from the known-working image as the reference candidate. Do not rebuild, split, or mix its SPL, U-Boot FIT, TF-A, or U-Boot DTB until the exact DRAM part and board-specific compatibility are established.

The extraction report recommends the file:

`bootloader/integrated/u-boot-sunxi-with-spl.bin`

Reported SHA-256:

`2cc42f955c5d0c5042b307ff67b646aa8d49740e2b0680a411506738b03e73eb`

Reported location in the source image: byte offset 8192 (0x2000), length 896329 bytes. The report describes a 49152-byte Allwinner eGON SPL immediately followed by an 847177-byte U-Boot FIT containing U-Boot proper, TF-A BL31 and the U-Boot DTB.

**Verification qualification:** these values and hashes are transcribed from the uploaded extraction report. The binary ZIP archives are present in this repository, but their binary contents have not yet been independently opened and re-hashed through the current GitHub connector. Treat the report as strong reference evidence, not an independent binary verification by this assistant.

## Reported image layout

- MBR partition table.
- Early boot payload starts at byte 8192.
- Partition 1 starts at byte 1048576 (sector 2048), active FAT16, 20 MiB.
- Partition 2 starts at byte 23068672 (sector 45056), ext4 rootfs, 300 MiB.
- FAT partition contains `BOOT.SCR` and `uImage`.
- Boot script selects root partition 2 by PARTUUID, loads `uImage` from FAT, and invokes `bootm`.
- The reported integrated bootloader ends at byte 904521, leaving 144055 bytes before the first partition begins.

These layout details must be compared against the generated image before any image recipe is changed.

## DRAM and boot safety

The extraction report finds DRAM initialization code inside the SPL and no separate DRAM-parameter blob or SCP image. It does **not** recover the actual DRAM timing values and does **not** prove that this SPL is exclusive to the 1 GB variant.

Therefore:

1. Do not import 2 GB/4 GB U-Boot settings or guessed DRAM timings.
2. Do not replace the working SPL/TF-A/U-Boot chain with a newly built one for a physical test.
3. Do not write to SPI NOR or the installed device's boot media as part of CI.
4. Keep the working SD card untouched and use a separate test card when hardware testing eventually becomes appropriate.
5. Preserve the reported boot script, FIT/kernel and partition/PARTUUID relationship unless a generated-image comparison proves a safe compatible alternative.

## Planned integration sequence

1. Obtain direct access to the extracted integrated binary and boot script as individual repository files (or another binary-readable transfer path).
2. Independently verify the integrated binary's SHA-256, size, SPL checksum, FIT structure and component boundaries.
3. Inspect the current pinned ImmortalWrt image-generation recipe and identify exactly where it writes `u-boot-with-spl.bin` at offset 0x2000.
4. Build an isolated image-generation path that preserves the verified boot payload and records its provenance; do not alter the standard recipe until the comparison is complete.
5. Compare generated partition layout, FAT boot files, root PARTUUID, kernel FIT and DTB.
6. Produce a clearly labelled inspection artifact and manifest. A successful CI build is not proof of physical boot.
7. Only after independent review and a recovery plan, consider testing on a separate microSD card. Never treat the output as flash-ready merely because CI passes.

## Current project boundary

This reference belongs to `5H4K1L/cubie-a5e-immortalwrt`. It does not authorize changes to the separate OpenWrt repository. No workflow is dispatched by this documentation commit.
