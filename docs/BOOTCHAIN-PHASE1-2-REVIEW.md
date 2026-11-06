# Boot-chain Phase 1–2 review — 2026-11-06

**Repository scope:** `5H4K1L/cubie-a5e-immortalwrt` only.  
**Board:** Radxa Cubie A5E V1.20, owner-reported 1 GB DRAM.  
**Reference image:** `Immortal-1GB-DRAM-A5E-LuCi-XLR8.img`.  
**Status:** Phase 1 reference-file verification complete; Phase 2 static source review and assembly design complete. No Actions workflow was dispatched or modified. No image was flashed.

## Phase 1 — preserve and verify reference assets

The immutable source copies remain on the `reference-files` branch under `reference/`. The branch now includes `VERIFICATION-MANIFEST-2026-11-06.md`; its README points to the manifest.

| Asset | Size | SHA-256 | Result |
|---|---:|---|---|
| `reference/u-boot-sunxi-with-spl.bin` | 896,329 bytes | `2cc42f955c5d0c5042b307ff67b646aa8d49740e2b0680a411506738b03e73eb` | PASS — independently hashed and structure checked |
| `reference/BOOT.SCR` | 375 bytes | `6b6e846f0f386c9809f33c51d2219c6da281337bbcd6f805752806db9ca5f814` | PASS — independently hashed; legacy header and payload CRCs valid |
| `reference/uImage` | 6,439,800 bytes | `092b355715b481ecd4791eb923f4f2533ab0b4dfd4a39c62c54413e9acde3c5e` | PASS — user's local SHA-256 output matches the report and checksum list |

### Verified structural facts

- Integrated U-Boot/SPL begins with the eGON header, declares a 49,152-byte SPL, and contains a FIT immediately afterward with a reported/parsed total size of 847,177 bytes.
- The integrated binary is 896,329 bytes long.
- The extraction report places it at byte offset 8,192 (`0x2000`) in the reference image.
- The report's first partition begins at byte 1,048,576. The integrated payload ends at byte 904,521, leaving 144,055 bytes before that partition.
- BOOT.SCR selects partition 2 for root, resolves its PARTUUID, loads `uImage` from FAT and invokes `bootm`.

### Verification limits

The three standalone files match the supplied extraction digests. The original full image was not re-read and re-extracted in this follow-up pass, so the extraction report's provenance/offset relationship has not been independently re-established from the original disk image here. The uImage checksum was supplied by the user after local hashing; the assistant did not independently download and parse the full 6.4 MB file. Most importantly, none of these checks proves that the SPL is specific to the 1 GB board or that the DRAM timings are correct for this exact PCB/DRAM part.

## Phase 2 — static review of the pinned upstream image path

Reviewed upstream `immortalwrt/immortalwrt` commit:

`4765b657625ec5bbda74b44400465fe043802b30`

Relevant files:

- `target/linux/sunxi/image/Makefile`
- `target/linux/sunxi/image/cortexa53.mk`
- `target/linux/sunxi/image/gen_sunxi_sdcard_img.sh`
- `package/boot/uboot-sunxi/uEnv-a523.txt`

### Confirmed build behavior

1. `Device/radxa_cubie-a5e` inherits `Device/sun55i-a527`, which inherits the sun50i FIT/LZMA kernel recipe.
2. The `sunxi-sdcard` image step creates a FAT boot filesystem, copies the generated `boot.scr` and generated kernel FIT as `uImage`, then calls `gen_sunxi_sdcard_img.sh`.
3. The image generator creates the partition table and writes the newly built U-Boot/SPL at byte offset 8,192 using `dd bs=1024 ... seek=8 conv=notrunc`.
4. It then writes the FAT and rootfs partition images at their partition offsets.
5. The A523 boot script selects MMC partition 2 for root, resolves its PARTUUID, loads FAT `uImage`, and calls `bootm`.

**Consequence:** a normal successful full-image build replaces the known-working early boot payload with a newly built U-Boot/SPL at `0x2000`. CI success alone does not make that image safe for this board.

## Proposed preservation architecture — design only

Do not change the existing standard image recipe yet. The safer future implementation is a separate post-build assembly operation that creates a *new output file* and leaves the original CI image untouched:

1. Check out the verified reference asset from the `reference-files` branch at an immutable commit SHA, not a floating branch name. The branch contains the reference binary and its verification manifest.
2. Verify the integrated bootloader's exact expected byte length and SHA-256 before opening the image for modification. Fail closed on any mismatch.
3. Parse the output image's MBR and calculate the first partition's actual byte offset. Do not assume 1 MiB merely because that is the reference image's layout.
4. Require `offset + bootloader_size <= first_partition_offset`. Reject overlap; never truncate the reference binary.
5. Copy the built image to a distinct output path, then write the verified integrated bootloader at byte offset `0x2000` in that copy only.
6. Re-read the assembled image region and verify its SHA-256 and byte-for-byte equality with the approved reference binary.
7. Verify that bytes outside the replaced early-boot region remain unchanged. Retain the original standard-build image and both hashes as comparison artifacts.
8. Label the result `UNTESTED — DO NOT FLASH`. This is a candidate for later inspection, not a hardware-approved image.

### What should and should not be reused

- **Preserve as one unit:** the integrated `u-boot-sunxi-with-spl.bin`. Do not mix its SPL with a separately rebuilt U-Boot FIT or BL31.
- **Do not automatically restore the old `uImage`:** the old file contains Linux 6.18.34 and its DTB. The new image must carry the kernel/DTB built with its own root filesystem and matching modules.
- **Do not blindly restore the old `BOOT.SCR`:** compare the generated script's commands and environment assumptions with the verified reference script. The important behavior is FAT-loading the generated `uImage`, selecting root partition 2 by PARTUUID, and invoking `bootm`.
- Keep the generated MBR, partition table, FAT filesystem, root filesystem, kernel FIT and root PARTUUID from the same build unless a separate compatibility review justifies a change.

### Required validation before any hardware test

- Verify reference binary hash/length at build time and post-assembly.
- Verify MBR signature and partition boundaries; confirm the reference bootloader ends before the first partition.
- Verify generated `boot.scr` content and U-Boot command compatibility.
- Inspect generated `uImage` FIT configuration, kernel, DTB, compression, load/entry values and consistency with the rootfs kernel modules.
- Confirm root partition UUID in the generated boot arguments resolves to the actual root partition.
- Compare early boot bytes and report all changed image ranges.
- Establish a recovery route and test only on a separate microSD card. Do not touch SPI.

## Next implementation step

Prepare a small, independently tested host-side assembly helper and unit tests that implement the fail-closed checks above. Only after code review should the full-image workflow be changed to call it. Any workflow execution remains a separate decision and requires explicit authorization.

## Explicitly not done

- No changes to the full-image workflow or upstream source recipe.
- No workflow dispatch, restart, cancellation or duplicate build.
- No bootloader, SPI, eMMC or microSD write.
- No claim that this SPL is proven 1 GB-specific.
- No claim that any generated image is flashable.
- No work in the companion OpenWrt repository.
