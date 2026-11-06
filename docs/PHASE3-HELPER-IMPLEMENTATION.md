# Phase 3 — host-side bootloader preservation helper

Branch: `feature/bootloader-preservation-helper`

## Implemented

- `scripts/preserve_bootloader.py`: standalone Python 3 helper, standard library only.
- `tests/test_preserve_bootloader.py`: synthetic-image unit tests.
- `tools/bootloader-preservation/README.md`: usage and safety notes.
- `docs/PHASE3-HELPER-IMPLEMENTATION.md`: implementation record.

## Fail-closed checks

1. Reference binary must be exactly 896,329 bytes and match SHA-256 `2cc42f955c5d0c5042b307ff67b646aa8d49740e2b0680a411506738b03e73eb`.
2. Input image must have a valid MBR signature and at least one well-formed MBR partition; protective GPT MBRs are rejected.
3. The earliest partition start is parsed from the actual MBR, not assumed. The replacement range `[0x2000, 0x2000 + size)` must end before it.
4. Reject extended MBR partitions, invalid partition boot flags, partition extents beyond the image, and overlapping primary partition entries. The helper accepts an uncompressed raw MBR `.img`, not a gzip archive.
5. Input image is copied to a temporary file in the output directory; only that copy is modified.
6. Read-back SHA-256, final image size and byte-for-byte identity outside the replacement range are checked.
7. Output is published without clobbering an existing destination.

## Local validation history

- Initial helper validation: 9 synthetic-image unit tests passed locally, with `py_compile` also passing. The local test harness used an equivalent implementation and a patched small test payload; it was not a GitHub Actions run.
- Two additional regression tests have now been added for an invalid MBR partition boot flag and refusing an output path equal to the source image. **The expanded 11-test suite has not yet been executed.**

## Remaining validation

- Run the exact committed test suite and `py_compile` against the branch contents.
- Run the helper against a disposable known-layout image fixture using the actual reference bootloader binary, if the binary can be made available to the test environment.
- The real full firmware image has not been assembled.
- No workflow file has been changed or dispatched.
- No image or hardware has been modified.

## Scope / limitations

- The helper does not prove DRAM compatibility, SPL runtime behavior, kernel/DTB compatibility or root filesystem correctness.
- The generated file is explicitly marked `UNTESTED — DO NOT FLASH`.
- No automatic hardware-flashing step is appropriate.


### Test fixture review note

A static review caught over-escaped byte literals in the expanded MBR tests; these were corrected so the synthetic fixture uses actual `55 aa` MBR signature bytes and `00 00` for the invalid-signature case. The 11-test suite remains pending execution against the committed branch files.


## Validation results (2026-11-06)

- Exact branch helper and expanded 11-test suite were reconstructed from committed GitHub file contents in an isolated local test directory.
- `python3 -m py_compile`: passed.
- `python3 -m unittest discover -s tests -v`: **11 tests passed**.
- Additional sparse 324 MiB image fixture: passed using documented MBR partition starts (LBA 2048 and 45056) and a generated 896,329-byte test payload. It verified the actual MBR-derived first partition offset (1,048,576 bytes), replacement end (904,521 bytes), read-back digest, source preservation, image-size preservation, and byte identity outside the replacement range.
- The extra fixture did **not** use the actual reference U-Boot/SPL binary; actual-binary assembly remains outstanding. The test payload's length matched the approved binary but its content/hash was generated and the helper's expected digest was patched for this isolated fixture.
- This does not validate DRAM initialization, SPL execution, kernel boot, device tree, root filesystem, or hardware. Candidate outputs remain `UNTESTED — DO NOT FLASH`.
- No workflow was modified or dispatched, and no firmware image or hardware was touched.


## Isolated CI workflow update — 2026-11-06

- Added `.github/workflows/bootloader-preservation-tests.yml` to run Python syntax compilation, the 11 helper unit tests, and `git diff --check`, separately from firmware builds.
- Its automatic push-triggered run #1 (ID `42208328230`) completed with `failure` after roughly four seconds. The job endpoint returned no step details and the log endpoint returned `BlobNotFound`; the cause is unknown and cannot be attributed to the helper tests.
- No rerun was started. The independently executed local validation remains 11/11 tests passing plus successful `py_compile`.
- The actual reference binary has not yet been used in a helper assembly fixture in the local runtime. Do not treat the generated-payload fixture as actual-binary validation.
- Existing firmware/kernel workflows were not modified or dispatched. No firmware image was built, flashed, or written to hardware.
- Next gate: establish the CI failure cause before any rerun or expansion to a CI integration fixture using the `reference-files` branch binary.


### CI job-start evidence

The failed run's only job reports `runner_id=0`, no runner name, and an empty steps array; it completed within three seconds. The job did not reach checkout, Python setup, or tests. The failure therefore appears to occur before step execution; the exact GitHub-side cause is unavailable. No retry was made.
