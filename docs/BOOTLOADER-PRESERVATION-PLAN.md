# Bootloader-preservation plan — 2026-11-06

## Scope and safety

This note records a static review of the pinned upstream ImmortalWrt source at commit `4765b657625ec5bbda74b44400465fe043802b30`. It does not approve any image for flashing.

- Target: Radxa Cubie A5E V1.20, owner-reported 1 GB DRAM.
- Known-working reference: `Immortal-1GB-DRAM-A5E-LuCi-XLR8.img`.
- Keep the known-working SD card and any SPI contents unchanged.
- Do not write an unverified image, U-Boot, SPL, or SPI payload to the board.
- The bootloader ZIPs currently in `reference-files` are documented by an extraction report; their bytes and hashes have not yet been independently verified.

## Confirmed upstream image-writing behavior

At the pinned upstream commit:

1. `target/linux/sunxi/image/Makefile` uses the `sunxi-sdcard` image step and passes the device-specific `*-u-boot-with-spl.bin` file to `gen_sunxi_sdcard_img.sh`.
2. `target/linux/sunxi/image/gen_sunxi_sdcard_img.sh` writes that U-Boot file directly into the output image using:

   ```sh
   dd bs=1024 if="$UBOOT" of="$OUTPUT" seek=8 conv=notrunc
   ```

   This places the newly built U-Boot/SPL payload at byte offset 8192 (0x2000).
3. The same script then writes the FAT boot partition and ext4 root partition at the partition offsets calculated by `ptgen`.
4. `target/linux/sunxi/image/cortexa53.mk` defines the Cubie A5E profile as `Device/sun55i-a527`, which inherits the sun50i FIT kernel definition.
5. `package/boot/uboot-sunxi/Makefile` defines `radxa-cubie-a5e` for Cortex-A53, depends on `trusted-firmware-a-sunxi-a523`, sets `UENV:=a523` and `ATF:=a523`, and installs the generated `u-boot-sunxi-with-spl.bin` as `radxa_cubie-a5e-u-boot-with-spl.bin`.
6. The A5E U-Boot environment script (`uEnv-a523.txt`) loads `uImage` from the FAT partition, uses the UUID of MMC partition 2 as root, and boots with `bootm`.

Relevant pinned source files:
- `target/linux/sunxi/image/Makefile`
- `target/linux/sunxi/image/gen_sunxi_sdcard_img.sh`
- `target/linux/sunxi/image/cortexa53.mk`
- `package/boot/uboot-sunxi/Makefile`
- `package/boot/uboot-sunxi/uEnv-a523.txt`
- A5E-specific U-Boot patches under `package/boot/uboot-sunxi/patches/`

## Consequence for this project

The current full-image workflow uses the standard upstream SD-card recipe. Therefore a successful build would contain a newly built U-Boot/SPL payload at 0x2000. The workflow's current artifact inspection does not preserve or replace that region; it only extracts it for inspection.

**A successful build is not automatically safe to flash.** The bootloader is a separate hardware compatibility risk from the Linux kernel and root filesystem.

## Recommended implementation sequence

### Phase 1 — verify reference bytes (waiting for user upload)

When the user supplies the extracted files or makes them accessible:

1. Independently calculate SHA-256 and byte length for the known-working integrated U-Boot/SPL binary.
2. Confirm the binary really came from the known-working image at offset 0x2000, not merely from a filename or an unverified report.
3. Compare the candidate against the complete original early boot region and identify padding/reserved bytes before the first partition.
4. Inspect the known-working FAT boot partition, especially `boot.scr`, `uImage`, and partition identifiers.
5. Record the exact source image hash, extracted-file hashes, offsets, and lengths in a machine-readable manifest.

No report-derived hash should be treated as verified until the actual bytes are available.

### Phase 2 — build and inspect without changing the boot chain

1. Keep the existing full-image workflow as an inspection-only build until runner allocation is resolved.
2. Inspect the newly built kernel, DTB, root filesystem, partition table, generated `boot.scr`, and generated U-Boot/SPL independently.
3. Compare the generated boot script and partition layout with the known-working image.
4. Do not modify the board or flash any artifact during this phase.

### Phase 3 — design a controlled bootloader-preserving image

Only after Phase 1 and Phase 2 pass:

1. Add a dedicated, explicit image assembly step that starts from the built image but restores the verified known-working U-Boot/SPL bytes at exactly 0x2000.
2. Make the step fail closed unless the reference binary's expected SHA-256 and byte length match an approved manifest.
3. Check that the reference payload ends before the first partition; reject overlap rather than truncate.
4. Verify after assembly that the bytes at 0x2000 exactly match the approved reference binary.
5. Keep the generated FAT boot partition separate from the U-Boot/SPL payload. Do not assume the boot script is interchangeable; validate its commands and root PARTUUID behavior.
6. Emit a separate artifact clearly labeled as a bootloader-preserving candidate, not a tested or flash-approved image.

Do not implement this assembly step until the real reference bytes are independently verified.

### Phase 4 — hardware validation

Before any production use, establish a recovery route and test on a spare SD card. Keep SPI untouched. First validate SD boot, serial console, storage, Ethernet, and reboot/power-cycle behavior. Only consider any SPI operation if the board-specific recovery procedure and a verified backup exist.

## Current blockers

- User upload of the extracted bootloader files is pending.
- The extraction report's hash, size, and offset claims remain unverified against raw bytes.
- Full-image workflow run #4 failed before runner assignment (no steps); exact infrastructure cause remains unknown.
- The runner disk-cleanup mitigation has not yet been exercised.
- No hardware boot test has been performed on an image produced by this repository.

## Decision

**Proceed with static review and reference-byte verification only. Do not flash. Do not dispatch another full-image workflow until the runner-allocation issue is understood or the user explicitly approves a controlled retry.**
