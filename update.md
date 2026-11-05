# Project update / AI handoff

**Last reviewed:** 2026-11-05 (UTC)  
**Board:** Radxa Cubie A5E V1.20, Allwinner A527 / sun55i, 1 GB RAM  
**Default branch:** `main`  
**Related project:** [OpenWrt companion repository](https://github.com/5H4K1L/cubie-a5e-openwrt)

> Start here when resuming. ImmortalWrt kernel-validation run [#14](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/42018099723) on commit `7b45384289f41c39e4d8c10de5f51ddb170d5b22` completed successfully. Its artifact was downloaded and independently inspected: Linux 6.18.60 Image, A5E DTB, merged kernel config, and build manifest are present. IOMMU config symbols are enabled; the DTB has the IOMMU controller but no `iommus` client properties. This remains a kernel-validation build, not a complete firmware image.

## Goal and safety constraints

Develop reproducible ImmortalWrt support for the Radxa Cubie A5E 1 GB board, ultimately producing a board-specific microSD image.

- Preserve the working ImmortalWrt microSD and original image files.
- Do not write SPI, Boot0/SPL, TF-A, U-Boot, or DRAM timing data.
- Never apply 2 GB/4 GB DRAM settings to this 1 GB board.
- Do not flash current CI artifacts: they are kernel/DTB validation outputs, not firmware.
- Treat undocumented boot offsets, DRAM clocks/geometry, memory ranges, and FIT load/entry addresses as unknown until verified.

## Hardware / working baseline

- Board: Radxa Cubie A5E V1.20; SoC Allwinner A527.
- Compatible strings: `radxa,cubie-a5e`, `allwinner,sun55i-a527`.
- RAM: 1 GB (owner-confirmed); eight Cortex-A55 cores.
- Working installation: ImmortalWrt `sunxi/cortexa53`, Linux 6.18.34 AArch64.
- Boot partition: FAT `/dev/mmcblk0p1` (20 MB); root: ext4 `/dev/mmcblk0p2`.
- U-Boot version string: `2026.01_armbian-2026.01-S127a-Pd40e-Hc6a9-V2b7c-Bd0d2-R448a`.
- Working boot script loads FAT `uImage`, obtains root PARTUUID with `part uuid`, and invokes `bootm`.
- Working FIT uses `config-1`, LZMA kernel and uncompressed A5E DTB. Numeric load/entry addresses remain unknown.

## Source and current workflow

- Repository: [5H4K1L/cubie-a5e-immortalwrt](https://github.com/5H4K1L/cubie-a5e-immortalwrt), default branch `main`.
- Workflow: `.github/workflows/kernel-validation.yml`.
- Pinned upstream: `immortalwrt/immortalwrt` commit `4765b657625ec5bbda74b44400465fe043802b30`.
- Target: `sunxi/cortexa53`; CI kernel source version: Linux 6.18.60.
- Kernel config fragment: `build/cubie-a5e-kernel.config`.
- IOMMU patch: `build/patches/950-sun55i-iommu-attach-dev-linux-6.18.patch`.
- Workflow applies the patch to the checked-out source, builds host tools/toolchain, compiles `target/linux`, then collects the raw kernel `Image` and `sun55i-a527-cubie-a5e.dtb`.

## IOMMU compile blocker — resolved for compilation

### Original error

The first build failed because `sun55i_iommu_attach_dev()` had the old two-argument callback signature while the Linux 6.18 `iommu_domain_ops.attach_dev` function pointer expected three arguments:

```text
initialization of 'int (*)(struct iommu_domain *, struct device *,
struct iommu_domain *)'
from incompatible pointer type
'int (*)(struct iommu_domain *, struct device *)'
[-Wincompatible-pointer-types]
```

### Correction now in the repository

The pinned source's driver is located at:

`target/linux/sunxi/files/drivers/iommu/sun55i-iommu.c`

The compatibility patch changes the callback to:

```c
static int sun55i_iommu_attach_dev(struct iommu_domain *domain,
                                   struct device *dev,
                                   struct iommu_domain *old)
```

and marks `old` unused with `(void)old;`. The driver already serializes domain changes with `domain_lock`, tracks its hardware domain in `active_domain`, and uses that state to restore the previous translation-table base if hardware initialization fails. The patch preserves that existing logic; it does not cast the function pointer or disable IOMMU.

The workflow's explicit patch-application step now succeeds against the pinned source, and the subsequent Linux target compilation succeeds. This confirms the callback type mismatch no longer blocks this pinned Linux 6.18.60 build.

**Runtime caveat:** compilation does not prove that the driver's domain-switching behavior is correct at runtime on A527 hardware. The new `old` API argument is intentionally unused because this driver's existing `active_domain` state is used for rollback. Runtime IOMMU behavior still needs hardware testing if the device tree enables this IOMMU/client path.

## CI history — 2026-11-05

| Run | Commit / branch | Result | Finding |
|---|---|---|---|
| [#1](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41977839204) | `07f36a04`, main | Failure | Original attach_dev callback type mismatch |
| [#2](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41977944553) | `07f36a04`, main | Failure | Original validation attempt also failed |
| [#3](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41989178917) | `871b72cc`, fix branch | Failure | Patch hunk did not match source |
| [#4](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41989205076) | `d5dc11b2`, fix branch | Failure | Intermediate patch attempt |
| [#5](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41989214709) | `354c3e20`, fix branch | Success | Kernel validation passed on fix branch |
| [#6](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41993789031) | `0549f011`, main | Failure | Intermediate callback patch attempt |
| [#7](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41993795193) | `7320be21`, main | Failure | Malformed patch: hunk line count / tab formatting |
| **[#8](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41994608070)** | **`181496d1`, main** | **Success** | **Corrected patch, kernel compile, DTB collection, artifact upload all passed** |

### Latest successful run #8

- Run ID: `41994608070`.
- Commit: `181496d13895d517a8bc13abbd85c62eeafc74ea`.
- Branch: `main`.
- Result: completed / success.
- Patch step: passed.
- Host tools and ARM64 toolchain: passed.
- Linux target compilation: passed.
- Kernel and A5E DTB collection: passed.
- Artifact upload: passed.
- Artifact: `cubie-a5e-kernel-validation`, available and not expired when checked.
- GitHub-reported artifact archive size: 7,021,334 bytes.

The artifact contains the raw kernel `Image` and `sun55i-a527-cubie-a5e.dtb` collected by the workflow. No artifact contents or SHA-256 values have yet been independently inspected outside CI.

## What this result proves — and does not prove

It proves the corrected patch applies to the pinned ImmortalWrt source and that the Linux target compiles far enough for CI to collect and upload the expected kernel and DTB.

It does **not** prove:
- that the kernel boots on the physical A5E;
- that the IOMMU driver is functionally correct at runtime;
- that the generated DTB has the correct memory map after bootloader fixups;
- that a complete FIT, boot script, root filesystem, partitioned disk image, or flashable firmware has been produced;
- that Ethernet, storage, USB, Wi-Fi, thermal, reboot, or other peripherals work.

Do not flash the current artifact.

## Next steps — in order

1. Download the #8 `cubie-a5e-kernel-validation` artifact and inspect the kernel `Image` and A5E DTB. Record sizes, kernel version/config, DTB compatible strings, and SHA-256.
2. Compare the generated DTB against the known-working FIT's embedded DTB and inspect the runtime FDT/memory fixups. Do not invent memory nodes or RAM ranges.
3. Review the complete patch against the pinned driver's full callback implementation and Linux 6.18 API. Keep the minimal patch unless a semantic issue is demonstrated.
4. Compare the ImmortalWrt kernel/DTB/config with the successful OpenWrt validation artifact. Do not assume the two source trees are identical.
5. Inspect ImmortalWrt's sunxi image-generation pipeline and plan a board-specific FIT/rootfs/partition recipe matching the observed `bootm` flow.
6. Build a complete image in CI, inspect its partition table, boot files, FIT metadata, rootfs, PARTUUID behavior, and checksums.
7. Test only from a separate microSD card after static review. Keep the known-working card and SPI bootloader untouched.

## Companion OpenWrt status

[OpenWrt companion repository](https://github.com/5H4K1L/cubie-a5e-openwrt) passed kernel validation in [run #3](https://github.com/5H4K1L/cubie-a5e-openwrt/actions/runs/41975643597), branch `build/initial-a5e-workflow`, commit `ce996798ae6db98c8899c2958cdb8dcbbbe5d6be`. Its artifact is also kernel/DTB only, not a complete image.

## Important: not flashable

Neither repository has yet produced a complete, hardware-validated A5E firmware image. Do not flash a raw kernel/DTB artifact. Before any test boot, verify the FIT format, boot script, rootfs, partition layout, load/entry addresses, A5E DTB, root arguments, bootloader compatibility, and recovery plan. Test from a separate microSD card only.


## Image-generation pipeline review — 2026-11-05

The pinned ImmortalWrt source has a board profile, but its default SD-card image recipe needs careful review before use:

- `target/linux/sunxi/image/cortexa53.mk` defines `Device/radxa_cubie-a5e`, based on `Device/sun55i-a527`, which inherits the generic sun50i FIT/LZMA kernel flow.
- `package/boot/uboot-sunxi/Makefile` defines a matching `radxa-cubie-a5e` U-Boot target and depends on `trusted-firmware-a-sunxi-a523`; the source tree also carries A5E-specific U-Boot patches.
- `target/linux/sunxi/image/Makefile` and `gen_sunxi_sdcard_img.sh` build an MBR SD image, create FAT boot + ext4 root filesystems, and write `u-boot-with-spl.bin` at byte offset `0x2000` (8 × 1024 bytes).

**Important:** the standard generated SD image injects a newly built early boot chain; it is not a kernel/rootfs-only update and may replace the Boot0/SPL/TF-A/U-Boot bytes on the card. Do not build/flash/use it as a drop-in replacement until the new DRAM configuration and bootloader are verified against the exact 1 GB board revision and a recovery path exists.

The earlier read-only comparison of the two owner-supplied images found bytes from `0x2000` through `0x100000` identical, with Allwinner Boot0/SPL, BL31 and U-Boot-related signatures in that region. This is consistent with both images carrying the same early boot payload at the location used by the generic image script, but exact component boundaries and the active boot source (SD vs SPI) remain unconfirmed.

### Required decision before full image generation

1. Establish whether the working board loads early stages from microSD, SPI, or a combination.
2. Extract and fingerprint the known-working early boot payload from the original image if it is stored on the SD card.
3. Compare the newly built A5E U-Boot/TF-A/DRAM configuration with the known-working payload; do not silently substitute it.
4. Plan a safe image variant that reuses the verified bootloader payload unchanged, or omits bootloader writes only if an independent known-good SPI boot chain is proven.
5. Keep the existing FIT/LZMA `uImage`, `bootm` boot path and root PARTUUID behavior unless a deliberate replacement is validated.

The generic `KERNEL_LOADADDR:=0x40080000` in `cortexa53.mk` is a build default, not evidence of the numeric load/entry addresses in the owner's working FIT. Do not use it as a substitute for extracting the actual FIT metadata.
 

## Binary artifact inspection and DTB comparison — 2026-11-05

Both successful workflow artifacts were downloaded and inspected locally (ZIP contents, file type, kernel version string, DTB structure, and SHA-256).

| Artifact file | OpenWrt | ImmortalWrt |
|---|---:|---:|
| Kernel `Image` | 14,764,040 bytes | 15,501,320 bytes |
| Kernel version | 6.18.60 | 6.18.60 |
| Kernel SHA-256 | `0ad0d789f4cdd1c66be70a00b17f1fb5301e5e76afef801a31667a52058127e9` | `5bdd944d7acb3e83bad2f92e3d6232389d4c6b542251dd94897c3d57166c202e` |
| A5E DTB | 16,938 bytes | 30,722 bytes |
| DTB SHA-256 | `623f0efb2e74daea4bc69413699e66e001e964afcf36412014760d60da5b9d20` | `c27efbf42f23f50fccf9202f772de312b170d86a1601a82ba16553a11a702cf6` |
| FDT node count | 99 | 188 |

Both DTBs report model `Radxa Cubie A5E`, compatibles `radxa,cubie-a5e` and `allwinner,sun55i-a527`, and eight Cortex-A55 CPU nodes. Neither has explicit `/memory` or `/reserved-memory` nodes; bootloader/runtime FDT fixups must be understood before packaging.

The ImmortalWrt DTB contains materially more A5E board support than the OpenWrt DTB:
- IOMMU controller node `/iommu@2010000` (no `iommus` client properties were found in this DTB; node presence alone does not prove active client attachment).
- Second GMAC200 controller `/soc/ethernet@4510000`, enabled with `status = "okay"`.
- PCIe root complex and INNO combo PHY, enabled.
- SDIO Wi-Fi child node under `/soc/mmc@4021000`.
- CPU OPP tables, CPU/DDR/GPU thermal zones, regulator and power-domain nodes, DMA controllers, SPI/I2S/SPDIF and other A523/A527 board improvements.
- The OpenWrt DTB currently contains only the first Ethernet controller and lacks these additional nodes.

The principal ImmortalWrt kernel patch series responsible includes:
`030-01` GMAC200 node; `030-02` DMA nodes; `031-01` A5E second Ethernet enablement; `121-*` A523 clock/cpufreq fixes; `123-*` A523 thermal support; `124` Sun55i IOMMU driver; `125` INNO PCIe/USB3 combo PHY; `126` Sun55i PCIe support; `130-*` A523/A527 thermal, regulator, cpufreq and SoC DTS nodes; `131-*` A5E Wi-Fi/BT/PCIe/USB-C board improvements; plus supporting SPI/ASoC/PHY patches.

**Interpretation:** both kernels compile, but their hardware enablement is not equivalent. ImmortalWrt is the better current A5E feature baseline. Porting its relevant kernel patch series into the OpenWrt branch should be a separate, staged change with patch-application and kernel CI validation; do not copy the DTB binary itself or assume every ImmortalWrt patch is safe to transplant without checking dependencies.

The workflow artifacts contain only `Image` and DTB. They do not include the final kernel `.config`, preprocessed DTS, or full build log; future validation runs should consider uploading a kernel config and build manifest alongside the binary outputs.
 

## Follow-up artifact hashes and structural inspection

The downloaded validation ZIPs were extracted and checked:
- OpenWrt `Image`: 14,764,040 bytes; SHA-256 `0ad0d789f4cdd1c66be70a00b17f1fb5301e5e76afef801a31667a52058127e9`.
- OpenWrt DTB: 16,938 bytes; SHA-256 `623f0efb2e74daea4bc69413699e66e001e964afcf36412014760d60da5b9d20`.
- ImmortalWrt `Image`: 15,501,320 bytes; SHA-256 `5bdd944d7acb3e83bad2f92e3d6232389d4c6b542251dd94897c3d57166c202e`.
- ImmortalWrt DTB: 30,722 bytes; SHA-256 `c27efbf42f23f50fccf9202f772de312b170d86a1601a82ba16553a11a702cf6`.
- Both kernel Images identify as ARM64 boot executable Images and contain Linux version 6.18.60.
- Both DTBs identify the board correctly and describe eight Cortex-A55 CPUs.
- A structural FDT parser counted 99 nodes in the OpenWrt DTB and 188 in the ImmortalWrt DTB. Neither DTB contains `/memory` or `/reserved-memory`.
- ImmortalWrt DTB contains the IOMMU controller node, but no `iommus` client property was found in the tree. The presence of the controller node alone does not prove that any device is attached to it.

The current CI artifact ZIPs contain only `Image` and DTB. A workflow update is now being tested in both repositories to add the final kernel `.config` and a `BUILD-MANIFEST.txt` with pinned source revision, kernel version and SHA-256 values. Do not treat that new workflow as validated until its run completes successfully.


## Critical U-Boot / 1 GB DRAM finding — 2026-11-05

The pinned ImmortalWrt source's `package/boot/uboot-sunxi/Makefile` builds U-Boot **2026.04** for the `radxa-cubie-a5e` target and depends on `trusted-firmware-a-sunxi-a523`. The upstream `v2026.04` `configs/radxa-cubie-a5e_defconfig` currently contains:

```text
CONFIG_DRAM_SUNXI_TPR6=0x38000000
CONFIG_DRAM_SUNXI_TPR10=0x802f3333
CONFIG_DRAM_SUNXI_TPR11=0xc7c5c4c2
CONFIG_DRAM_SUNXI_TPR12=0x3533302f
```

These are exactly the "previous" values in the candidate community 1 GB timing change documented in the companion OpenWrt repository's `docs/DRAM-1GB.md`. That community candidate proposes different values for the 1 GB variant. This does **not** prove the default values are wrong for this exact PCB/DRAM chip, nor does it prove the candidate values are correct; it does prove that the current ImmortalWrt U-Boot build has no documented 1 GB timing override in the reviewed defconfig/package configuration.

**Do not flash a full ImmortalWrt SD image built with this default U-Boot payload.** The standard image generator writes the newly built `u-boot-with-spl.bin` at offset `0x2000`, replacing the early boot payload on the target SD card. First identify and fingerprint the known-working early boot payload and establish the active SD/SPI boot path. Any 1 GB timing change must be validated against the exact board revision, DRAM part and a recovery plan before it is included in a candidate bootloader.

Immediate safe direction: continue kernel/DTB and rootfs work separately; treat the default U-Boot binary as a separately gated artifact, not as approved for the user's 1 GB board.


## External repository review — 2026-11-05

A separate review has been committed at [docs/EXTERNAL-REPOSITORY-REVIEW.md](docs/EXTERNAL-REPOSITORY-REVIEW.md). Key decisions:

- Keep the Medvedolog PM1 Wi-Fi reset / BLDO1 `vqmmc` change as an unverified candidate; current A5E DTS already has PL7, mmc1, 35 MHz and 40 mA drive strength. Verify regulator/GPIO references and board behavior before adding it.
- Use Zuyih's older vendor IOMMU driver only as a register/master-ID reference, not as a Linux 6.18 replacement.
- The pinned ImmortalWrt PCIe driver already includes `MSI_FLAG_PCI_MSIX`; no duplicate MSI-X patch is needed.
- Treat the NixOS 1 GB DRAM profile as a research lead only. Do not change U-Boot DRAM values without exact RAM-chip identification and recovery evidence.
- Follow Radxa Image Builder's provenance and bootloader-preservation discipline for future image generation.

**Latest CI status at handoff:** workflow run [#10](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/42012445649), commit `161d6d04af912002ba28476e09d1eff7f4fc1969`, was still `in_progress` when checked. IOMMU patch application passed; host tools/toolchain was still running; kernel compile had not started. Earlier run [#8](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/41994608070) is the last confirmed successful kernel validation.

**Next:** wait for run #10, inspect its logs/artifacts, then add explicit CI assertions for `CONFIG_IOMMU_SUPPORT=y` and `CONFIG_SUN55I_IOMMU=y` and re-check the generated DTB's IOMMU client phandles. Do not import the external Wi-Fi or DRAM changes yet.


## CI guard update — 2026-11-05

The project kernel config fragment now explicitly requests `CONFIG_IOMMU_SUPPORT=y` and `CONFIG_SUN55I_IOMMU=y`. The validation workflow now checks those settings and checks that the pinned Sun55i PCIe RC driver advertises `MSI_FLAG_PCI_MSIX` (already present upstream in this pinned tree). It also runs `git diff --check` after applying the IOMMU callback patch.

The first workflow-file edit was malformed and produced run [#13](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/42017995765), which failed before creating a job. The workflow file was then rewritten and re-read to verify its complete YAML content. The corrected workflow is commit `7b45384289f41c39e4d8c10de5f51ddb170d5b22`; validation run [#14](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/42018099723) was queued when last checked. Older runs #10–#12 were already in progress and are not evidence for the new assertions.

Do not treat the new assertions as passed until run #14 reaches the relevant steps. No external Wi-Fi, PCIe, or DRAM patch was copied into the kernel tree.


## Latest successful artifact inspection — run #14 (2026-11-05)

**Scope:** ImmortalWrt repository only. No OpenWrt repository files or workflows were changed or triggered during this inspection.

- Workflow run: [#14](https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/42018099723)
- Project commit: `7b45384289f41c39e4d8c10de5f51ddb170d5b22`
- Branch: `main`
- Job: `kernel`, completed successfully; all 10 substantive steps passed, including configuration, IOMMU patch dry-run/application, host tools/toolchain, Linux target compile, artifact collection and upload.
- Artifact: `cubie-a5e-kernel-validation`, ID `13415116354`, ZIP size 7,061,627 bytes; expires 2026-11-19.
- Pinned upstream: `immortalwrt/immortalwrt` commit `4765b657625ec5bbda74b44400465fe043802b30`.
- Kernel: Linux 6.18.60, ARM64 Image, 15,501,320 bytes.
- DTB: 30,722 bytes, FDT version 17.
- Artifact contains `Image`, `sun55i-a527-cubie-a5e.dtb`, `kernel.config`, and `BUILD-MANIFEST.txt`.

### Verified checksums from the artifact manifest

| File | SHA-256 |
|---|---|
| `Image` | `77cec8d3385e8b4ba63a06d7a21f9d485deed91bf98b7193cc562b36a13abcc3` |
| `sun55i-a527-cubie-a5e.dtb` | `c27efbf42f23f50fccf9202f772de312b170d86a1601a82ba16553a11a702cf6` |
| `kernel.config` | `7cf9ce235cb5b19c1052b9be341592560a1534ec5fe1d62d49ebbcf610e0571d` |

### Kernel configuration confirmed in the actual compiled config

- `CONFIG_IOMMU_SUPPORT=y`
- `CONFIG_SUN55I_IOMMU=y`
- `CONFIG_PCI=y`
- `CONFIG_PCI_MSI=y`
- `CONFIG_STMMAC_ETH=y`
- `CONFIG_DWMAC_SUN8I=y`
- `CONFIG_WLAN=y`
- `CONFIG_MMC=y`
- `CONFIG_MMC_SUNXI=y`

### DTB structure confirmed from the compiled binary

- IOMMU controller: `/iommu@2010000`, compatible `allwinner,sun55i-a523-iommu`, enabled by default, `#iommu-cells = <2>`.
- **No node in this DTB has an `iommus` property.** The controller is described, but no DMA client is attached through an IOMMU phandle. This does not prove active translation.
- MMC controller `/soc/mmc@4021000` is enabled, marked non-removable, and supplies the SDIO Wi-Fi child placeholder `wifi@1`. That child currently has only a `reg` property and no `compatible` property.
- The board's fixed 3.3 V Wi-Fi regulator is present and referenced by the MMC controller. The current DTS patch sets MMC1 maximum frequency to 35 MHz and MMC1 pin drive strength to 40 mA.
- Both GMAC controllers are enabled: `/soc/ethernet@4500000` and `/soc/ethernet@4510000`.
- PCIe root complex `/soc/pcie@4800000` is enabled and has reset GPIO and PCIe 3.3 V supply properties.
- No explicit `/memory` or `/reserved-memory` nodes were found. Runtime bootloader FDT fixups remain important for memory sizing and reservations.

### Interpretation and decisions

1. **IOMMU compile blocker: resolved for the pinned Linux 6.18.60 build.** The driver is compiled with IOMMU enabled and the callback patch applies. Runtime attachment/translation remains unproven because no DTB client is mapped and hardware testing has not happened.
2. **Do not add arbitrary `iommus` properties.** First establish the A527 IOMMU master IDs, driver expectations, and which DMA-capable peripherals are supported by the current driver/binding. Use the vendor BSP only as a reference and validate against the Linux 6.18 driver.
3. **Wi-Fi is not yet hardware-validated.** The DTB has the SDIO slot and regulator, but the child is only a placeholder. Do not add PM1/BLDO1 power sequencing from the external community patch until the exact board wiring and polarity are confirmed.
4. **PCIe and both Ethernet controllers are described/enabled, not proven working.** Runtime tests are still required.
5. The artifact is still only a raw kernel + DTB + config + manifest. It is not a FIT, root filesystem, complete SD-card image, or safe-to-flash firmware. Do not flash it.

### Next technical steps

1. Review the full Linux 6.18 Sun55i IOMMU driver and binding alongside the vendor BSP to identify documented master IDs and safe candidate clients. Do not modify the DTB until the mapping is evidence-based.
2. Inspect the pinned A5E DTS patch series for PCIe PHY, SDIO Wi-Fi, regulators and both GMACs; distinguish enabled nodes from hardware-tested features.
3. Obtain/read-only inspect the known-working FIT and boot files to record the exact FIT configuration, load/entry addresses, embedded DTB and root PARTUUID behavior.
4. Design a kernel/rootfs image path that preserves the known-working early boot payload. Do not use the generic sunxi image generator's new U-Boot payload on the working card.
5. Only after static image review, test from a separate microSD card with the known-good bootloader preserved and a recovery path available.

**Status rule:** run #14 supersedes the earlier note above that run #14 was queued and supersedes any implication that artifact contents were not yet independently inspected. Historical run #8 details remain as historical CI history.


## IOMMU driver / binding source review — 2026-11-05

Read-only review of the pinned Linux 6.18 source at upstream commit `4765b657625ec5bbda74b44400465fe043802b30`:

- Driver: `target/linux/sunxi/files/drivers/iommu/sun55i-iommu.c`.
- IOMMU controller DT node is introduced by `target/linux/sunxi/patches-6.18/130-04-arm64-dts-sun55i-dtsi-add-iommu-usbc-pcie-combophy-nodes.patch`.
- The A5E DTB's `#iommu-cells = <2>` matches the driver's `of_xlate()` handling: cell 0 is stored as `owner->tlbid`; cell 1 is an optional boolean-like `owner->flag`. The driver later uses the flag when enabling/disabling the selected master.
- The driver bounds the usable master index to 0–6 (`master_id_bitmap` has seven entries). Its A523/A527 platform data labels those entries `ISP`, `CSI`, `VE0`, `VE1`, `G2D`, `DE`, and `DI`; an eighth string is `DEBUG_MODE`, not a normal master index.
- The source contains older generic comments showing a one-cell example, but the actual A523/A527 DT node and `of_xlate()` implementation use two cells. Do not copy the stale generic example as the binding definition.
- The current A5E DTS does not attach any client node to the controller. No evidence yet establishes the correct IOMMU master ID for any enabled A5E peripheral. Do not assign IDs to GMAC, PCIe, MMC/SDIO, USB, display, or other devices based on guesses.
- The driver uses one global `active_domain` for the hardware and switches the global translation-table base in `attach_dev()`. The Linux 6.18 compatibility patch leaves the new per-device `old` callback parameter unused and retains the existing global-domain rollback logic. This is consistent with a minimal API-signature fix, but runtime correctness and multi-client behavior remain untested on hardware.

**Decision:** no IOMMU client DT properties or driver behavior changes are justified by this source review alone. Next evidence needed is a reliable A527/Sun55iw3 master-ID table or vendor DTS showing client-to-master assignments for this exact IP revision, followed by hardware testing with a recoverable test card.
