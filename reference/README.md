# Radxa Cubie A5E reference files

This directory contains reference material extracted from the known-working
`Immortal-1GB-DRAM-A5E-LuCi-XLR8.img` image.

## Contents

- `Cubie-A5E-1GB-Bootloader-Extraction.zip` — extracted boot-chain package.
- `uboot_files.zip` — additional U-Boot reference files.
- `EXTRACTION_REPORT.md` — extraction findings, image layout, boot flow, hashes and caveats.
- `SHA256SUMS` — supplied checksum list.
- `BOOTCHAIN-INTEGRATION-DECISION.md` — project integration decision and safety checklist.
- `VERIFICATION-MANIFEST-2026-11-06.md` — results of the follow-up checks on the standalone U-Boot/SPL, BOOT.SCR and uImage files.

## Verification status

The three standalone files now match the extraction report's expected SHA-256 digests.
The integrated U-Boot/SPL binary and BOOT.SCR were independently hashed and structurally
checked. The uImage digest was checked against the user's locally calculated SHA-256
result. See the verification manifest for exact sizes, hashes and limitations.

## Important

Checksum agreement does not prove that the SPL is exclusive to the 1 GB board variant,
and the exact DRAM part/timings remain unknown. The original full disk image was not
re-extracted during the follow-up verification pass. No file here is approved for
direct flashing. Keep these files as immutable reference assets and use them only in
a fail-closed build-time preservation process after reviewing the generated kernel,
boot script, partition layout and recovery plan.
