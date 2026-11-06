# A5E bootloader-preservation helper

Creates a separate, untested image candidate by replacing only bytes at offset
`0x2000` with the verified integrated A5E U-Boot/SPL binary. Supply an **uncompressed
raw .img**, not the `.img.gz` archive. It never edits the source image and refuses
to overwrite an existing output.

## Run

```sh
python3 scripts/preserve_bootloader.py \
  --image path/to/normal-build.img \
  --bootloader path/to/u-boot-sunxi-with-spl.bin \
  --output path/to/normal-build-preserved-bootloader-UNTESTED.img
```

The script is pinned to the reference binary's expected size (896329 bytes)
and SHA-256
`2cc42f955c5d0c5042b307ff67b646aa8d49740e2b0680a411506738b03e73eb`.
It requires an MBR image with a valid `0x55AA` signature, rejects protective
GPT MBRs, extended partitions, malformed entries, partitions extending past
the image, and overlapping partitions; it checks that the payload ends before
the earliest MBR partition. After writing, it checks the bootloader
read-back hash, image size, and byte-for-byte identity of every byte outside
the replacement region. Output publication is atomic and refuses clobbering.

## Tests

```sh
python3 -m unittest discover -s tests -v
```

Tests use synthetic disk images and a patched small test payload; they do not
need the real board image or reference binary.

## Safety

- Candidate output is labelled `UNTESTED — DO NOT FLASH`.
- The normal build image is never modified.
- This tool does not verify DRAM timing compatibility, U-Boot runtime behavior,
  kernel/DTB compatibility, or root filesystem correctness.
- It does not modify a workflow, write to SPI/eMMC/SD, or flash a device.
