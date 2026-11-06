# Reference boot-file verification manifest — 2026-11-06

## Scope

This manifest records checksum and structural verification of the three standalone boot files stored in this branch. The source archives and original known-working disk image are retained unchanged.

Board reported by owner: Radxa Cubie A5E V1.20, 1 GB DRAM.
Reference image named in the extraction report: `Immortal-1GB-DRAM-A5E-LuCi-XLR8.img`.

## Verification results

| File | Size (bytes) | SHA-256 | Verification |
|---|---:|---|---|
| `u-boot-sunxi-with-spl.bin` | 896329 | `2cc42f955c5d0c5042b307ff67b646aa8d49740e2b0680a411506738b03e73eb` | PASS — binary bytes independently hashed; eGON header, SPL length and FIT container boundaries inspected |
| `BOOT.SCR` | 375 | `6b6e846f0f386c9809f33c51d2219c6da281337bbcd6f805752806db9ca5f814` | PASS — binary bytes independently hashed; U-Boot legacy image header CRC and payload CRC both valid |
| `uImage` | 6439800 | `092b355715b481ecd4791eb923f4f2533ab0b4dfd4a39c62c54413e9acde3c5e` | PASS — user ran SHA-256 on the downloaded file and supplied this result; it matches the extraction report and `SHA256SUMS` |

## Structural findings

### Integrated U-Boot/SPL

- eGON signature: `eGON.BT0` at relative offset 4.
- SPL length field: 49,152 bytes.
- FIT begins at relative offset 49,152.
- FIT totalsize: 847,177 bytes.
- Integrated file length: 896,329 bytes.
- Reported source-image placement: byte offset 8,192 (`0x2000`).
- Reported first partition start: byte offset 1,048,576 (1 MiB).
- Reported end of integrated payload: byte offset 904,521, leaving 144,055 bytes before the first partition.

### BOOT.SCR

- Legacy U-Boot image magic: `0x27051956`.
- Header CRC: valid.
- Payload CRC: valid.
- Script selects MMC partition 2 as root, resolves its PARTUUID, loads `uImage` from FAT, and invokes `bootm`.

### uImage

The supplied extraction report describes this file as a FIT containing an LZMA-compressed Linux 6.18.34 kernel and an A5E device tree. The checksum match verifies the downloaded file against the report's listed digest; the assistant did not independently retrieve and parse the full 6.4 MB file from GitHub in this session.

## Important limits

1. The checksum checks establish that the uploaded standalone files match the extraction report's expected digests. The original full disk image was not re-read in this verification pass, so the reported source-image offset and provenance have not been independently re-extracted byte-for-byte here.
2. The extraction report explicitly says it has not established whether this SPL is exclusive to the 1 GB variant. Exact DRAM part, geometry and timing values remain unknown.
3. Structural validity and checksum agreement do not constitute a hardware boot test.
4. Do not write these files to SPI, eMMC or microSD as standalone operations. Do not flash a generated image until the complete image, recovery route and hardware test plan are reviewed.

## Decision

The three files are accepted as **verified reference assets for controlled build-time comparison and image assembly design only**. The integrated U-Boot/SPL binary may be considered as the preservation candidate, subject to the fail-closed checks in the project's bootloader-preservation plan. It is not a flash approval.
