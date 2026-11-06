# Current status — 2026-11-06 (latest check)

**Read this note before following older handover instructions.**

## Scope
All inspection and changes are limited to `5H4K1L/cubie-a5e-immortalwrt`. The companion OpenWrt repository is out of scope. No workflow was started, restarted, cancelled, or modified during this review.

## Latest GitHub Actions state
The repository currently reports 18 workflow runs. The newest full-image inspection is run #4:

- Run: https://github.com/5H4K1L/cubie-a5e-immortalwrt/actions/runs/42153635903
- Workflow: `A5E ImmortalWrt full-image inspection (UNTESTED)`
- Branch: `main`
- Project commit: `33f36b81997e6587cd57fa967e731aa6832e1bcc`
- Conclusion: `failure`
- Created/started: 2026-11-06 02:58:21 UTC
- Updated/finished: 2026-11-06 02:58:25 UTC
- Job ID: `127682517508`
- Job reports no runner assignment (`runner_id=0`, empty runner name) and no steps (`steps=[]`).

This is **not a source compilation failure**. The job failed before a runner executed any workflow step. The available metadata does not identify the exact infrastructure cause; do not claim a specific cause without logs or GitHub status details.

The earlier full-image run #3 failed during compilation/image generation because the runner ran out of disk space. The cleanup mitigation was added after that run, but run #4 never reached the cleanup step, so the mitigation remains unvalidated.

Kernel validation run #14 remains a successful kernel/DTB baseline. It is not a full firmware image.

## Workflow runtime summary
Across the 18 visible runs, the Actions timing endpoint reports approximately **1,138.05 minutes (18.97 hours)** of aggregate run duration:
- Full-image inspection: 4 runs, 158.37 minutes.
- Kernel validation: 14 runs, 979.68 minutes.

This is workflow run duration, not account-level billable Actions usage. The run actors shown in the run list are `5H4K1L`; the connected collaborator identity does not appear as the actor for these runs.

## Current repository layout and workflow behavior
- `.github/workflows/full-image-inspection.yml`: manual `workflow_dispatch` only; no push trigger.
- `.github/workflows/kernel-validation.yml`: manual trigger plus push trigger for `build/**`, the workflow itself, and `README.md`.
- `build/cubie-a5e-kernel.config`: kernel config fragment.
- `build/patches/950-sun55i-iommu-attach-dev-linux-6.18.patch`: IOMMU compatibility patch.
- `docs/FULL-IMAGE-BUILD-INSPECTION.md`: image inspection and no-flash safety gate.
- `reference-files` branch: extracted-image report, ZIP archives, checksums, and bootchain decision note.

The full-image workflow currently:
1. Checks out project and pinned ImmortalWrt source.
2. Removes selected large preinstalled runner components/caches and cleans apt/Docker data.
3. Installs dependencies and feeds.
4. Configures `sunxi/cortexa53` / `radxa_cubie-a5e`.
5. Applies the Linux 6.18 IOMMU patch and checks PCIe MSI-X source support.
6. Runs `make -j2 V=s`.
7. Inspects the generated SD image and uploads it as an untested artifact.

## Recommended next work — no workflow dispatch yet
1. Keep doing static review of the image recipe and workflow; no reason to spend Actions minutes until the runner allocation problem is understood.
2. Check GitHub Actions service/status and repository Actions settings/limits before retrying run #4.
3. Once GitHub can allocate a runner, launch only one fresh manual full-image inspection run on the latest `main`; verify the cleanup step output and available disk before compilation.
4. Keep bootloader integration blocked until the actual binary files can be independently inspected and re-hashed.
5. Do not produce or label any image as flashable until the known-working boot chain and exact 1 GB DRAM configuration are validated.

## Bootloader safety gate
The known-working image report identifies the integrated `u-boot-sunxi-with-spl.bin` at byte offset `0x2000`, but those values are currently report-derived and have not been independently re-verified against the ZIP bytes. The standard upstream sunxi image recipe embeds a newly built U-Boot/SPL at the same offset. Do not flash the generated image, write SPI, or guess DRAM timings. Preserve the known-working SD card and SPI contents.

## User upload pending
The user will upload the extracted binary files later. Until then, continue source/workflow/documentation review only; do not claim independent binary verification.


## Additional upstream image-recipe review

Static review of pinned upstream ImmortalWrt commit `4765b657625ec5bbda74b44400465fe043802b30` confirmed that the standard sunxi SD-card generator writes the newly built `u-boot-with-spl.bin` at byte offset 8192 (0x2000), using `dd bs=1024 seek=8`. The Cubie A5E U-Boot package selects the A523 Trusted Firmware (BL31) and generates its boot script from `uEnv-a523.txt`. The boot script loads `uImage` from the FAT partition and uses partition 2's PARTUUID as root.

This confirms the primary bootloader-overwrite risk: a successful standard image build replaces the known-working early boot payload with a newly built one. A bootloader-preserving assembly design is documented separately in [BOOTLOADER-PRESERVATION-PLAN.md](BOOTLOADER-PRESERVATION-PLAN.md). It is intentionally a plan only; no workflow or image recipe was changed.

Source review links:
- [Upstream sunxi image recipe](https://github.com/immortalwrt/immortalwrt/blob/4765b657625ec5bbda74b44400465fe043802b30/target/linux/sunxi/image/Makefile)
- [Upstream SD-card generator](https://github.com/immortalwrt/immortalwrt/blob/4765b657625ec5bbda74b44400465fe043802b30/target/linux/sunxi/image/gen_sunxi_sdcard_img.sh)
- [Upstream A5E U-Boot package definition](https://github.com/immortalwrt/immortalwrt/blob/4765b657625ec5bbda74b44400465fe043802b30/package/boot/uboot-sunxi/Makefile)
