# OpenWrt Disk Image Resizer v2.3

A **fully offline**, single-file HTML tool that resizes OpenWrt whole-disk images to a target capacity right in your browser (grow or shrink), with ESXi-ready VMDK virtual disks generated separately on demand. No installation, no network access, and your files never leave your machine — everything is processed locally in the browser.

Companion file: `OpenWrt_Disk_Image_Resizer_v2.3.html` (open it in a recent Chrome or Edge)

---

## Features

- **Whole-disk image resizing**: supports both GPT and MBR partition tables. Resizing updates the partition tables accordingly (GPT primary/backup tables and the protective MBR), with an option to extend the last partition to fill the new disk.
- **Main button outputs only the resized image**: "Resize and Generate Files" produces just the resized `.img.gz` — no ESXi files attached.
- **ESXi pair as a separate output**: the "Generate ESXi Pair" button behind it independently generates the VMDK pair (descriptor file + `-flat.vmdk` data file) for the current target capacity, without pressing the main button first.
- **ESXi-native VMDK format**: a descriptor file (`createType="vmfs"`, `adapterType` is `lsilogic`) plus a `-flat.vmdk` data file — upload them to a datastore and attach directly.
- **Original info shown on file selection**: input file size, decompressed original image size, and partition table type (GPT / MBR) — no need to start processing first.
- **Streaming for large files**: when the decompressed image or the target capacity exceeds 2 GiB, the tool switches to streaming — the input is only probed (first 1 MiB) to determine size and partitions instead of being loaded fully into memory, and output is written in chunks without a full in-memory copy. There is no confirmation dialog for targets over 2048 MiB; large files are processed directly. Verified with a 64 MiB image grown to 4 GiB, a 3 GiB input converted to ESXi, and an 8 GiB image shrunk to 4 GiB.
- **Reset releases memory**: clicking "Reset" actively frees the cached decompressed data and returns the memory right away.

## Usage

1. Open `OpenWrt_Disk_Image_Resizer_v2.3.html` in Chrome / Edge.
2. Click "Image file" and select a whole-disk image (`.img` / `.bin` / `.raw` / `.img.gz`, e.g. the official `combined-efi` images).
   - The results panel immediately shows the original image size and partition table type — check them before continuing.
3. Enter the **target capacity** in MiB (default 1024). It may be larger or smaller than the original image; the minimum is 16 MiB. Shrinking truncates data beyond the new capacity, so shrink the filesystem/partitions inside the system first.
4. Keep "Extend the last partition" checked so the last partition uses the new capacity; uncheck it if you only want a larger disk with untouched partitions (the tool refuses if the last partition does not fit the target).
5. Click "**Resize and Generate Files**" and wait for processing to finish — you get the resized `.img.gz` (a single file).
   - If the target equals the original size, the main button reports that no resize is needed and generates nothing.
   - For large files (decompressed size over 2 GiB, or target over 2 GiB), the browser shows a save dialog to pick the output location; when the page is opened over `http(s)`, a direct download link may be generated instead.
6. If you need the ESXi files, click "**Generate ESXi Pair**" to generate the `.vmdk` descriptor + `-flat.vmdk` data file for the current target capacity. When the target equals the original size, it outputs the original-size pair.

## Output files

For an input named `openwrt.img.gz` with a 1024 MiB target:

| Source | File | Description |
|---|---|---|
| Main button | `openwrt-1024MiB-resized.img.gz` | The resized image (gzip) |
| ESXi button | `openwrt-1024MiB-resized.vmdk` | Descriptor for the resized version |
| ESXi button | `openwrt-1024MiB-resized-flat.vmdk` | Data file for the resized version |

## Using the VMDK in ESXi

1. Upload the `.vmdk` and its matching `-flat.vmdk` into the **same directory** on a datastore.
2. When adding a hard disk to a virtual machine, select the `.vmdk` descriptor file to attach it.
3. Note: the `-flat.vmdk` file is full-size (not sparse), so make sure the datastore has room comparable to the target capacity.

## Supported inputs and compatibility

- **Whole-disk images**: OpenWrt `combined` / `combined-efi` and similar whole-disk images. Standalone sysupgrade, factory, or rootfs packages are not whole-disk images and cannot be resized as disks.
- **All common gzip wrappings are handled**:
  - Signature comments appended after the gzip body in official images (e.g. signify signatures) are ignored automatically, with a note shown;
  - Multi-member (concatenated) gzip files are decompressed in full;
  - If the browser's native decompression is unavailable or returns incomplete data, the tool falls back to a built-in pure-JavaScript inflater that decompresses member by member with verification.
- **Integrity verification**: decompressed results are checked against the disk size declared in the GPT header and against the partition bounds. Incomplete images (declared size larger than the actual data, partitions overrunning the image, entry CRC mismatch) are rejected with an error instead of producing a broken output.

## Limitations and notes

- Capacities are in MiB only. Shrinking truncates data beyond the new capacity — shrink the filesystem and partitions inside the system first (the tool only changes the partition table, not the filesystem).
- This tool updates the **partition-table level** capacity. It does not grow filesystems such as SquashFS themselves, and writable overlay space is not guaranteed to grow with it — always validate on a test device / VM before production use.
- For MBR images using extended/logical partition layouts, verify the resized result carefully.
- For large images, use a 64-bit browser and close unnecessary tabs.
- Display-only quirk (the files themselves are fine): the `.img.gz` trailer's ISIZE field is 32-bit, so when the decompressed size is an exact multiple of 4096 MiB (e.g. 4096 MiB or 8192 MiB) it wraps to 0 and the page may show 0. The generated file itself is complete, with the correct decompressed size and partition table, and works normally.

## Privacy and safety

- Fully offline single file: no network requests, no external resources, and image files never leave your machine.
- The selected original file is only read; all outputs are newly generated by the browser and your source image is never modified.

## FAQ

**Q: Why don't the "original image size" and the .gz file size match?**
That's expected. For a `.gz`, the "input file size" is the compressed package itself, while the "original image size" is the real disk size after decompression.

**Q: A 4096 / 8192 MiB image shows 0 — is the file broken?**
No, it is display-only (see "Limitations and notes" above): the 32-bit ISIZE field in the `.gz` trailer wraps around. The file itself is complete and usable, with the correct decompressed size.

**Q: "Failed to read original info: browser out of memory" — what should I do?**
Close other tabs and select the file again, or click "Resize and Generate Files" to try processing directly. If it still fails, the decompressed image is likely too large for a single browser page on this machine.

**Q: What does "GPT declares a disk size larger than the actual image" mean?**
The `.gz` decompressed incompletely or the file is corrupted (e.g. an incomplete download). Please obtain a complete image file and try again.

**Q: Why is processing a 1 GiB+ image slow?**
Large images that take the built-in inflater fallback path are noticeably slower (pure-JavaScript decompression) — this is normal. On the native decompression path, GiB-scale images usually finish within tens of seconds.
