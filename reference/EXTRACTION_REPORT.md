# EXTRACTION REPORT: boot chain of `Immortal-1GB-DRAM-A5E-LuCi-XLR8.img` (Radxa Cubie A5E, 1 GB)

Extraction and documentation only. The source image was opened read-only, nothing was flashed or written to any block device, and the GitHub repository was not touched.
Labels used below: **[CONFIRMED]** = read directly from bytes in the image; **[KNOWLEDGE]** = relies on documented behaviour of Allwinner BROM / mainline U-Boot that is not visible in the image itself; **[INFERENCE]** = my reasoning, not proven; **[UNVERIFIED]** = could not be established.

## 1. Bottom line

* The boot chain is a single contiguous raw blob at byte offset **8192**: Allwinner `eGON.BT0` SPL (49152 bytes) immediately followed by a U-Boot **FIT** (`u-boot.itb`, 847177 bytes) holding U-Boot proper, TF-A **BL31** and the board DTB. Total **896329 bytes**, ending at byte 904521 (below the 1 MiB start of partition 1). [CONFIRMED]
* U-Boot identifies itself as `2026.01_armbian-2026.01-S127a-Pd40e-Hc6a9-V2b7c-Bd0d2-R448a`, built Jun 22 2026 14:26:24 +0800 (SPL and U-Boot proper both carry this string), and BL31 as `v2.13.0(debug):armbian`. [CONFIRMED]
* **Recommendation: option A. Reuse the complete integrated binary `bootloader/integrated/u-boot-sunxi-with-spl.bin` unmodified** (SHA-256 `2cc42f955c5d0c5042b307ff67b646aa8d49740e2b0680a411506738b03e73eb`). Reasoning in section 12.

## 2. Source image [CONFIRMED]

| Item | Value |
|---|---|
| File | `Immortal-1GB-DRAM-A5E-LuCi-XLR8.img` |
| Size | 337641746 bytes (= 659456 sectors + 274 trailing bytes) |
| SHA-256 | `b06846f3779c76ce18c7d9bf41560f2abef2e94e4cae4c78c7e6c129e12c3019` (re-checked after extraction; unchanged) |
| Partition table | MBR, disk signature `0x5452574f` (bytes "OWRT"); boot-code area (bytes 0 to 439) is all zero; no GPT |
| Distro | ImmortalWrt SNAPSHOT r39779-cb7c288288, target `sunxi/cortexa53`, board `radxa_cubie-a5e`, `supported_devices: radxa,cubie-a5e` |

| # | Start sector (byte offset) | Sectors / size | MBR type | Active | Filesystem | Label / UUID |
|---|---|---|---|---|---|---|
| 1 | 2048 (1048576) | 40960 / 20 MiB | 0x0c | yes | FAT16 (mkfs.fat; note the MBR type says FAT32-LBA while the filesystem is FAT16) | no label, volume serial 6A28-DBA2 |
| 2 | 45056 (23068672) | 614400 / 300 MiB | 0x83 | no | ext4 (has_journal, extents) | label `rootfs`, UUID `ff313567-e9f1-5a5d-9895-3ba130b4a864` |

Entries 3 and 4 are empty. PARTUUIDs derived from the MBR signature are `5452574f-01` and `5452574f-02` [KNOWLEDGE: Linux/U-Boot PARTUUID format for MBR].
The image is **not a pristine build output**: the ext4 superblock shows mount count 5, last mounted 2026-06-28, and files under `/etc` were modified after the 2026-06-10 build, so it was booted and configured at least once. [CONFIRMED]

## 3. Unpartitioned areas [CONFIRMED]

| Region | Content |
|---|---|
| Sector 0 | MBR (partition table only) |
| Bytes 512 to 8191 | all zero |
| Bytes 8192 to 904521 | SPL + u-boot.itb (last non-zero byte is 903521; the remaining 1000 bytes up to the FIT end are zero padding inside the FIT) |
| Bytes 904522 to 1048575 | all zero |
| Gap between partition 1 and 2 (1 MiB) | all zero |
| After partition 2 | 274 bytes: OpenWrt `fwtool` metadata trailer (JSON, then a short binary footer) |

## 4. Whole-image signature scan [CONFIRMED]

* `eGON.BT0`: 3 hits. 0x2004 is the real header. 0xB823 is a string inside the SPL and 0x9870D is a string inside U-Boot proper (both within the already-identified blobs). The same two strings account for the `TOC0.GLH` hits.
* `sunxi-package` / `BOOT_PACKAGE` (Allwinner vendor boot_package): 0 hits. No vendor boot0 / boot_package is present.
* FDT magic `d00dfeed`: 4 hits, all accounted for: 0xE000 (u-boot.itb), 0xD5710 (U-Boot DTB inside the FIT), 0x10F000 (`uImage` FIT in partition 1), 0x72C11C (kernel DTB inside `uImage`).
* `EFI PART`: 3 hits, all deep inside the ext4 rootfs (binary strings), none at LBA 1; there is no GPT.

## 5. Boot-chain component inventory

| File (in package) | Offset in image | Length | SHA-256 | Detection method | Standalone / integrated | Confidence |
|---|---|---|---|---|---|---|
| `bootloader/integrated/u-boot-sunxi-with-spl.bin` | 8192 (0x2000) | 896329 | `2cc42f955c5d0c5042b307ff67b646aa8d49740e2b0680a411506738b03e73eb` | eGON.BT0 header at 0x2004 + FDT magic at 0xE000 + FIT totalsize; end = SPL end + FIT totalsize | integrated (SPL + u-boot.itb, contiguous) | High (every boundary derived from headers; SPL checksum verified) |
| `bootloader/spl/sunxi-spl.bin` | 8192 (0x2000) | 49152 | `90766957a3107c173a0d11d6d9d0c1369dc2c6720aa52c395406c4a74cfc7352` | eGON.BT0 magic; header length field (0xC000); checksum recomputed and matches (0xee809e49) | component of the integrated binary (standalone-bootable by BROM) | High |
| `bootloader/uboot/u-boot.itb` | 57344 (0xe000) | 847177 | `73acf675878c7341c6028299dbef135799e580feead45a2489107e3d71ccfbce` | FDT magic d00dfeed directly after SPL; FDT totalsize field = 847177 | component (FIT container: U-Boot + BL31 + DTB) | High |
| `bootloader/uboot/u-boot-proper.bin` | 57652 (0xe134) | 775128 | `a146ffaec2b769df75aa14ead78ffce62e7d6d50119136e961eaaf7b43b816a3` | FIT image node "uboot" (type standalone, arm64, load 0x4a000000), inline data property | component inside the FIT (not independently bootable) | High for boundaries (FIT-defined); not independently run |
| `bootloader/trusted-firmware/bl31.bin` | 832972 (0xcb5cc) | 41124 | `f3ec81ed6c20e7c3f85741e10f745e4d71a18476813ec9ee476cd573e6681913` | FIT image node "atf" (type firmware, load/entry 0x54000); strings identify TF-A "v2.13.0(debug):armbian", Allwinner platform | component inside the FIT | High for boundaries; version string read from the binary |
| `device-tree/u-boot-allwinner_sun55i-a527-cubie-a5e.dtb` | 874256 (0xd5710) | 28952 | `95dac9300f75c9fc11d05d7ae789639497e386cffcb2f451d3839677980b389e` | FIT image node "fdt-1" (flat_dt), description allwinner/sun55i-a527-cubie-a5e | component inside the FIT | High |

Details of the integrated binary:
* SPL header: `eGON.BT0` at 0x2004, length field 49152, stored checksum 0xee809e49, recomputed 0xee809e49 (match: True); mainline-style `SPL\x02` signature at 0x2014; embedded DT name `allwinner/sun55i-a527-cubie-a5e`. [CONFIRMED]
* FIT `u-boot.itb`: images `uboot` (arm64, load 0x4a000000), `atf` (load and entry 0x54000), `fdt-1`; configuration `config-1` (firmware = atf, loadables = uboot, fdt = fdt-1); FIT timestamp 0x6a38d5a1 = 2026-06-22 06:26:41 UTC. Full structure: `device-tree/u-boot-itb-fit-structure.txt`. [CONFIRMED]
* **No SCP image** in the FIT and no SCP-related strings anywhere in the bootloader. **No separate DRAM-parameter blob.** See `bootloader/scp/NOT_FOUND.txt` and `bootloader/other/NOT_FOUND.txt`.

Other early-boot files:

| File | Source | What it is | Size | SHA-256 |
|---|---|---|---|---|
| `boot-config/BOOT.SCR` | FAT p1 /BOOT.SCR | legacy U-Boot script image (mkimage -T script) | 375 | `6b6e846f0f386c9809f33c51d2219c6da281337bbcd6f805752806db9ca5f814` |
| `boot-config/uImage` | FAT p1 /uImage | FIT: LZMA kernel 6.18.34 + DTB | 6439800 | `092b355715b481ecd4791eb923f4f2533ab0b4dfd4a39c62c54413e9acde3c5e` |
| `device-tree/kernel-fit-radxa_cubie-a5e.dtb` | inside uImage FIT (fdt-1), byte 6410524 of uImage | kernel DTB | 28438 | `5ad1e6abbcc85b4a801441942ccb92d41192ec595ce9da3c7dfee93ac04ba621` |
| `partition-info/mbr-sector0.bin` | image offset 0, 512 bytes | MBR | 512 | `391ad508f4422881f92172f021776ced188410febaae1bb53c9b2a7a1bda185c` |
| `image/image-trailer-metadata.bin` | image offset 337641472, 274 bytes | OpenWrt fwtool metadata trailer | 274 | `cd8f2a41d6b39c1e5ce3012181c32702f2b498844cd7fbbd207e400850f243b3` |

## 6. Boot sequence

1. SoC BROM reads the `eGON.BT0` SPL from the SD card at byte 8192 and runs it from SRAM. [KNOWLEDGE for BROM behaviour; the header/checksum validity of what sits at 8192 is CONFIRMED]
2. SPL initialises DRAM (code compiled into the SPL, see section 8), then loads `u-boot.itb`. It sits directly after the SPL, at image byte 57344 = sector 112. [CONFIRMED location; the SPL's own sector constant was not disassembled, so that the SPL reads from exactly this sector is INFERENCE, supported by the fact that the image boots]
3. FIT loading places BL31 at 0x54000, U-Boot proper at 0x4a000000 and selects the DTB; BL31 hands over to U-Boot (EL2). [CONFIRMED from FIT properties; execution order is KNOWLEDGE]
4. U-Boot runs its built-in `bootcmd=run distro_bootcmd` with `boot_targets=fel nvme0 mmc_auto usb0 pxe dhcp` and `boot_scripts=boot.scr.uimg boot.scr`; it scans for a **bootable (active) MBR partition**, which is partition 1, and finds `BOOT.SCR` (FAT is case-insensitive). [CONFIRMED env text; behaviour KNOWLEDGE]
5. `BOOT.SCR` (legacy-uImage script, header CRC and payload CRC both valid) contains:
```
setenv mmc_rootpart 2
part uuid mmc ${mmc_bootdev}:${mmc_rootpart} uuid
setenv loadkernel fatload mmc \$mmc_bootdev \$kernel_comp_addr_r uImage
setenv bootargs coherent_pool=2M console=ttyS0,115200 root=PARTUUID=${uuid} rootwait
setenv uenvcmd run loadkernel \&\& bootm \$kernel_comp_addr_r
run uenvcmd
```
   i.e. root = partition 2 by PARTUUID, console `ttyS0,115200`, load `uImage` from the FAT partition into `kernel_comp_addr_r` (0x44000000) and `bootm` it. [CONFIRMED]
6. `uImage` is a **FIT** (not a legacy uImage) built 2026-06-10: `kernel-1` = "ARM64 OpenWrt Linux-6.18.34", LZMA (6409969 bytes compressed, 22349832 decompressed), load/entry 0x40080000, plus `fdt-1` = "radxa_cubie-a5e device tree blob" (28438 bytes). The kernel has no embedded config (no IKCFG). [CONFIRMED]

## 7. Filesystem inspection

* Partition 1 (FAT16) contains only `BOOT.SCR`, `uImage`, and a Windows-created `System Volume Information` folder (`WPSettings.dat` 12 bytes, `IndexerVolumeGuid` 76 bytes). The Windows artefacts are not boot-relevant and were not extracted. There is no `/boot` directory in the ext4 rootfs. [CONFIRMED]
* Partition 2 (ext4): ImmortalWrt rootfs. Relevant files copied to `image/rootfs-metadata/`: `/etc/openwrt_release`, `/etc/openwrt_version`, `/usr/lib/os-release`, `/etc/board.json`, `/etc/board.d/*`, `/etc/fstab`, `/lib/upgrade/platform.sh`, `/lib/uboot-envtools.sh`, `/lib/upgrade/keep.d/uboot-envtools`.
* Not found: kernel `.config` (no IKCFG, no `/proc/config.gz` file), `/etc/fw_env.config`, U-Boot environment storage, `extlinux.conf`, any `.dtb` in the rootfs. Kernel modules for 6.18.34 are present (105 entries, listed in `firmware/FIRMWARE_INVENTORY.txt`), including `aic8800_bsp/fdrv` for the on-board Wi-Fi.
* Firmware in the rootfs (`regulatory.db`, `aic8800/sdio/*` ≈ 5 MiB, `brcm`) is Wi-Fi/regulatory firmware used after boot, not part of the early boot chain. It is inventoried with SHA-256 but **not copied**.
* Not collected on purpose: `/etc/config/*` (user configuration), `/root`, SSH/host keys, shadow files, application data.

## 8. DRAM-related findings

Confirmed:
* DRAM initialisation is **inside the SPL** (strings: "Timeout initialising DRAM", "This DRAM setup is currently not supported.", "Unsupported DRAM configuration: column number invalid", "...row number invalid"). No separate DRAM parameter/timing file, boot0 DRAM table or Allwinner `dram_para` package exists in the image (signature scan, section 4).
* The SPL header fields for DRAM size and boot media (offsets 0x24 and 0x28) are **0** in the stored image. [CONFIRMED value; that these are filled in at run time by the SPL follows the mainline `boot_file_head` layout from memory, so KNOWLEDGE]
* Neither DTB (U-Boot nor kernel) has a `/memory` node, and U-Boot proper contains the `/memory` fixup string, so the memory size is not stored statically in a DTS. The board DTB defines regulators named `dvdd-dram` and `dvcc-pm-lpddr4`.

Inference / unverified:
* The regulator name suggests LPDDR4 [INFERENCE].
* **[UNVERIFIED] Whether this SPL is specific to the 1 GB variant, or whether one SPL auto-detects the size.** The only "1GB" evidence is the file name; nothing inside the image states a capacity. Establishing it would need either a disassembly of the SPL DRAM parameters or comparison with a known-good 2 GB/4 GB build from the same source. The specific DRAM clock, type, ODT and timing values were **not recovered**.

## 9. Board identification [CONFIRMED]

SPL DT name `allwinner/sun55i-a527-cubie-a5e`; U-Boot DTB model "Radxa Cubie A5E", compatible `radxa,cubie-a5e` / `allwinner,sun55i-a527`; U-Boot prints `CPU: Allwinner A523 (SUN55I)`; kernel DTB model "Radxa Cubie A5E"; U-Boot env `fdtfile=allwinner/allwinner/sun55i-a527-cubie-a5e.dtb` (unused here, since the kernel FIT carries its own DTB); `board.json` id `radxa,cubie-a5e`; trailer metadata `supported_devices: radxa,cubie-a5e`.

## 10. Which files are necessary

Necessary to reproduce the known-working boot chain:
1. `bootloader/integrated/u-boot-sunxi-with-spl.bin` (written at byte 8192 of the image). This single file replaces the separate SPL, U-Boot, BL31 and DTB.
2. The partition layout: MBR, partition 1 active FAT starting at 1 MiB, partition 2 as rootfs (so `part uuid ... 2` and `root=PARTUUID=` resolve), keeping the disk signature scheme in mind.
3. `boot-config/BOOT.SCR` (script contents above) and a kernel FIT named `uImage` in the FAT partition.

Useful reference / optional: the component files (`sunxi-spl.bin`, `u-boot.itb`, `u-boot-proper.bin`, `bl31.bin`, `u-boot.dtb`), `uboot-builtin-default-env.txt`, both DTBs, FIT structure dumps, `image/rootfs-metadata/*`, `firmware/FIRMWARE_INVENTORY.txt`, MBR sector, trailer metadata.

## 11. Confirmed vs assumed (summary)

| Statement | Status |
|---|---|
| Integrated SPL + FIT at 8192, 896329 bytes, boundaries from headers | CONFIRMED |
| SPL checksum valid; length field 49152 | CONFIRMED |
| FIT holds uboot / atf / fdt-1, no SCP | CONFIRMED |
| U-Boot 2026.01 Armbian-flavoured build, TF-A v2.13.0 (debug) armbian | CONFIRMED (version strings) |
| Armbian build system is the origin of these binaries | INFERENCE (from version strings; no network to check upstream) |
| FIT is read by SPL from sector 112 | INFERENCE |
| DRAM parameters embedded in SPL | CONFIRMED that no external blob exists; parameter values NOT recovered |
| 1 GB-specific SPL | UNVERIFIED |
| DRAM type is LPDDR4 | INFERENCE |
| The extracted binary boots on hardware | NOT TESTED here (the source image is reported by you as booting; no flashing was done) |

## 12. Final engineering question: A, B or C?

**Choose A: reuse the complete known-working integrated bootloader binary.**

Evidence:
1. The DRAM initialisation is compiled into this SPL and no external DRAM data exists, so rebuilding (option C) would require re-deriving DRAM parameters that this analysis did not recover; getting them wrong is exactly the failure mode you want to avoid.
2. SPL, U-Boot proper and BL31 come from one build (same version string and timestamps) and are tied together by layout (FIT directly after a 48 KiB SPL, BL31 at 0x54000). Mixing a new U-Boot or BL31 with this SPL (option B) gains nothing and risks layout/size mismatches. Splitting is only justified if you must swap one part, and then the SPL and FIT must be kept compatible.
3. The binary fits below the 1 MiB partition start (144055 bytes spare), so it can be dropped into the existing layout with no partition changes.
4. Its validity is established structurally: valid eGON checksum, valid FIT, all images accounted for.

Caveats: [UNVERIFIED] it may or may not be usable on 2 GB/4 GB boards; the TF-A is a `(debug)` build (extra console output); source/licence compliance applies when redistributing (U-Boot is GPL, TF-A BSD), so keep a pointer to the matching Armbian U-Boot and TF-A source for the version strings above.

### Recommended integration into the ImmortalWrt build
* Add the file to the repo as a pinned prebuilt asset (record its SHA-256) and, when generating the SD image, write it at 1024-byte offset 8, without truncating: `dd if=u-boot-sunxi-with-spl.bin of=<image> bs=1024 seek=8 conv=notrunc` (done on the image file in the build, never on a device).
* Keep partition 1 starting at sector 2048 and flagged active, FAT, containing `BOOT.SCR` (or `boot.scr`) and the kernel FIT `uImage`; keep rootfs as partition 2.
* Keep the `boot.scr` content, or make sure any new script keeps `mmc_rootpart 2`, the `uImage` load via `kernel_comp_addr_r` and `bootm`.
* Do not let the generic sunxi U-Boot package overwrite this bootloader for this device.
* Optional follow-up to close the 1 GB question: compare against an SPL from a different-capacity board image or disassemble the SPL DRAM setup.

## 13. Reproducibility

`scripts/extract_bootchain.py` regenerates this package from the image; `scripts/commands.md` lists equivalent `dd` commands; `scripts/verify_against_image.sh` re-cuts each component from the original image and compares it byte for byte. `SHA256SUMS` covers every file in the package.
