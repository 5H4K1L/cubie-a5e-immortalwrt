# Phase 3 — host-side bootloader preservation helper

Branch: `feature/bootloader-preservation-helper`

## Implemented

- `scripts/preserve_bootloader.py`: standalone Python 3 helper, standard library only.
- `tests/test_preserve_bootloader.py`: synthetic-image unit tests.
- `tools/bootloader-preservation/README.md`: usage and safety notes.

## Fail-closed checks

1. Reference binary must be exactly 896,329 bytes and match SHA-256 `2cc42f955c5d0c5042b307ff67b646aa8d49740e2b0680a411506738b03e73eb`.
2. Input image must have a valid MBR signature and at least one well-formed MBR partition; protective GPT MBRs are rejected.
3. The earliest partition start is parsed from the actual MBR, not assumed. The replacement range `[0x2000, 0x2000 + size)` must end before it.
4. Reject extended MBR partitions, partition extents beyond the image, and overlapping primary partition entries. The helper accepts an uncompressed raw MBR `.img`, not a gzip archive.
5. Input image is copied to a temporary file in the output directory; only that copy is modified.
6. Read-back SHA-256, final image size and byte-for-byte identity outside the replacement range are checked.
7. Output is atomically published only if the destination does not already exist.

## Local validation

Executed with Python's standard `unittest` runner against synthetic 2 MiB MBR images: **9 tests passed**.

Covered: successful replacement and unchanged source, bad digest, partition overlap, invalid MBR signature, protective GPT MBR, existing-output refusal, wrong payload size, out-of-image partition extents and overlapping MBR partitions.

## Scope / limitations

- No real firmware image was assembled in this step.
- No workflow file was changed or dispatched.
- The helper does not prove DRAM compatibility, SPL runtime behavior, kernel/DTB/rootfs compatibility or flash safety.
- The generated file is explicitly marked `UNTESTED — DO NOT FLASH`.
- Before workflow integration, review the helper, add it to CI's unit-test stage, and run it against a disposable known-layout image fixture. Do not add an automatic hardware-flashing step.
