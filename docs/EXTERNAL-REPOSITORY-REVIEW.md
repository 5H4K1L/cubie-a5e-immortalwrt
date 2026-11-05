# External repository review — Cubie A5E

**Reviewed:** 2026-11-05  
**Board target:** Radxa Cubie A5E V1.20, A527, owner-confirmed 1 GB RAM  
**Project policy:** ImmortalWrt is primary; OpenWrt is the comparison/secondary build.

This is an engineering triage of external references. It does not authorize copying code or flashing firmware. Each candidate must be checked against the exact pinned Linux/U-Boot source and the user's exact board.

## Decisions

| Reference | Finding | Decision |
|---|---|---|
| [Medvedolog/cubie-a5e-openwrt](https://github.com/Medvedolog/cubie-a5e-openwrt) | A5E Linux 6.18 Wi-Fi SDIO power-sequence patch adds PM1 reset GPIO and BLDO1 as `vqmmc`; its own patch says not hardware-verified. | Keep as a candidate only. The current ImmortalWrt 131-01 DTS patch already has PL7 3.3 V Wi-Fi power, mmc1, 35 MHz and 40 mA pin drive, but lacks this PM1/`vqmmc` addition. Before applying, prove the regulator label and GPIO polarity from the exact board DTS/schematic and test cold boot, warm boot and repeated firmware load. Do not assume the external patch is correct. |
| [zuyih/linux-cubie](https://github.com/zuyih/linux-cubie) | Vendor BSP includes Sun55i IOMMU driver and SoC DTS material, mostly from older 5.10/5.15/6.1-era trees. | Use for register/master-ID cross-reference only. Do not replace the ImmortalWrt Linux 6.18 driver with the old BSP driver. Compare register definitions, fault handling, master IDs and DT binding details before any functional IOMMU change. |
| [patryk4815/nixos-cubie-a5e](https://github.com/patryk4815/nixos-cubie-a5e) | Separates 1 GB LPDDR4 and 2/4 GB LPDDR4x U-Boot variants; carries PCIe/combo-PHY work and documents an MSI-X issue. | Useful reference. Do not copy its DRAM values into this board until the exact RAM chip and vendor-trained parameters are identified. Its PCIe MSI-X issue is already addressed in the pinned ImmortalWrt source: `pcie-sunxi-rc.c` includes `MSI_FLAG_PCI_MSIX` in `SUNXI_PCIE_MSI_FLAGS_SUPPORTED`; no duplicate fix is needed. Review the reported fix against our exact driver before future PCIe changes. |
| [psi1703/Radxa-Image-builder](https://github.com/psi1703/Radxa-Image-builder) | Separates kernel/DTB updates from the known-good boot chain, pins inputs, validates outputs and documents read-only target validation. | Reuse the build-safety method, not its Debian rootfs or complete image recipe. Our image must preserve the observed FIT/LZMA `uImage`, `bootm`, PARTUUID and known-good early boot payload until the SD/SPI boot source and offsets are proven. |
| [Yon4800/Cubie_A5E_San_Bot](https://github.com/Yon4800/Cubie_A5E_San_Bot) | Not a relevant firmware/kernel source. | No action. |

## DRAM safety — do not import timings

The NixOS reference's 1 GB profile contains a distinct U-Boot configuration (including TPR2, TPR6, TPR10, TPR11, TPR12 and a 720 MHz DRAM clock). It is a candidate for research, not a universal 1 GB profile. RAM capacity alone does not identify the DRAM die, density, voltage, rank/geometry or trained timing set. Other A5E community reports show that incorrect DRAM parameters can prevent boot.

Required evidence before considering any U-Boot DRAM change:
1. Exact board revision and RAM chip marking.
2. Vendor U-Boot UART output, preferably `sunxi_dram_para` values.
3. Exact source commit and complete generated U-Boot config.
4. Independent recovery route and a separate test SD card.
5. A build/test plan that does not write SPI NOR or overwrite the working boot chain.

Until then, preserve the existing bootloader and do not alter DRAM parameters.

## Current ImmortalWrt code checks

- The pinned source already has a Sun55i PCIe RC driver and explicitly advertises MSI-X support. Do not import a duplicate MSI-X patch.
- The existing A5E Wi-Fi DTS patch provides PL7 3.3 V power, mmc1 SDIO, 35 MHz maximum clock and 40 mA pin drive. PM1 reset sequencing and BLDO1 `vqmmc` remain an unverified candidate delta.
- The IOMMU callback patch is a Linux 6.18 API-signature compatibility fix. The callback's third `old` parameter is per-device prior-domain context; this driver's hardware state is tracked globally in `active_domain` under `domain_lock`. Keep the minimal signature fix for now; do not substitute `old` for the driver's global hardware rollback state without proving the hardware model and multi-client semantics.
- Kernel compilation does not prove IOMMU runtime correctness. Also, the currently inspected generated DTB had an IOMMU controller node but no `iommus` client properties; do not claim client translation is active until the final DTB is rechecked.

## Next engineering actions

1. Finish the active ImmortalWrt kernel-validation run and inspect the actual failure log or artifacts.
2. Add CI assertions for `CONFIG_IOMMU_SUPPORT=y`, `CONFIG_SUN55I_IOMMU=y`, and the PCIe MSI-X capability in the pinned driver.
3. Inspect the generated DTB for IOMMU controller and client phandles, PCIe/combophy, both GMACs, mmc1 and Wi-Fi power sequencing.
4. Compare the external PM1/`vqmmc` candidate with the complete board DTS and schematic. Add it only if all references resolve and the patch applies cleanly; label it unverified until board testing.
5. Keep DRAM and U-Boot changes out of the firmware build until chip identification and recovery are established.
6. Use the Radxa Image Builder's provenance and validation discipline when designing a bootloader-preserving complete image.

## Source links

- [Medvedolog Wi-Fi power-sequence patch](https://github.com/Medvedolog/cubie-a5e-openwrt/blob/claude/zen-babbage-ss68yn/target/linux/sunxi/patches-6.18/950-arm64-dts-allwinner-a527-cubie-a5e-wifi-pwrseq.patch)
- [Zuyih vendor Linux tree](https://github.com/zuyih/linux-cubie)
- [NixOS Cubie A5E U-Boot/DRAM configuration](https://github.com/patryk4815/nixos-cubie-a5e/blob/main/modules/uboot.nix)
- [Radxa Image Builder validation process](https://github.com/psi1703/Radxa-Image-builder/blob/main/docs/BUILD-FLOW.md)
