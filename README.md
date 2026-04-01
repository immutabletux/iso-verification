# ISO Verification Guide

A practical guide to verifying Linux ISO images before and after writing them to USB.

## 1. GPG Verification (Pre-Burn)

Confirm the ISO you downloaded is authentic and untampered. Most Linux distributions publish a checksum file and a GPG signature alongside their ISOs.

### Download the required files

You need three files from the distro's download page:

- The ISO image
- The checksum file (e.g. `SHA256SUMS`)
- The signature file (e.g. `SHA256SUMS.gpg`)

### Import the signing key

Find the key ID on the distro's official site, then import it:

```bash
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys <KEY_ID>
```

### Verify the signature

```bash
gpg --verify SHA256SUMS.gpg SHA256SUMS
```

Look for `Good signature from ...` in the output. A warning about the key not being certified with a trusted signature is normal if you haven't explicitly marked the key as trusted in your keyring.

### Verify the ISO checksum

```bash
sha256sum -c SHA256SUMS 2>&1 | grep ': OK$'
```

If the ISO filename appears followed by `OK`, the file matches the published checksum.

## 2. Post-Burn Byte-Level Verification

After writing the ISO to USB, read back the exact number of bytes written and compare the hash. This catches bad USB drives, failed writes, and bit-rot.

### Get the ISO size in bytes

```bash
ISO_SIZE=$(stat -c %s your-distro.iso)
```

### Hash the ISO

```bash
sha256sum your-distro.iso
```

### Hash the same number of bytes from the USB

Unmount the USB first, then read back the ISO-sized portion:

```bash
sudo umount /dev/sdX1
sudo dd if=/dev/sdX bs=1M count=$ISO_SIZE iflag=count_bytes status=none | sha256sum
```

Replace `/dev/sdX` with your USB device.

If both hashes match, the USB is a faithful byte-for-byte copy.

> **Why `count_bytes` matters:** The USB drive is larger than the ISO. Hashing the entire device would include trailing empty space and produce a different result.

> **Note:** `iflag=count_bytes` requires GNU coreutils. This command will not work on macOS or other BSD-based systems.

> **Note:** This method works for tools that do raw writes (`dd`, `cp`, balenaEtcher). Tools like Ventoy modify the USB structure, so a byte-for-byte comparison will not apply.

## 3. Post-Burn Per-File Hash Verification

Mount both the ISO and USB, then hash every file individually. This is slower than the byte-level method but tells you exactly which files are bad.

### Mount both sources

```bash
sudo mkdir -p /mnt/iso /mnt/usb
sudo mount -o loop your-distro.iso /mnt/iso
sudo mount -o ro /dev/sdX1 /mnt/usb
```

### Generate per-file checksums

```bash
cd /mnt/iso && find . -type f -exec sha256sum {} \; | sort -k2 > /tmp/iso_hashes.txt
cd /mnt/usb && find . -type f -exec sha256sum {} \; | sort -k2 > /tmp/usb_hashes.txt
```

### Compare

```bash
diff /tmp/iso_hashes.txt /tmp/usb_hashes.txt
```

No output means every file matches.

### Interpreting results

Missing files appear as lines present on only one side:

```
< abc123...  ./some/missing/file
```

Corrupted files appear as the same path with different hashes:

```
< abc123...  ./boot/vmlinuz
---
> def456...  ./boot/vmlinuz
```

### Show a summary

```bash
# Count mismatched entries
diff /tmp/iso_hashes.txt /tmp/usb_hashes.txt | grep '^[<>]' | wc -l

# List affected filenames
diff /tmp/iso_hashes.txt /tmp/usb_hashes.txt | grep '^[<>]' | awk '{print $3}' | sort -u
```

### Clean up

```bash
sudo umount /mnt/iso /mnt/usb
rm /tmp/iso_hashes.txt /tmp/usb_hashes.txt
```

> **Note:** This method requires the ISO to have a mountable filesystem (ISO 9660, FAT32, etc.). It will not work with hybrid formats that are not designed to be mounted as regular filesystems.

## Quick Reference

| Method | When to use | Speed | Detail |
|---|---|---|---|
| GPG verification | After downloading, before burning | Fast | Confirms authenticity |
| Byte-level hash | After burning to USB | Fast | Pass/fail for entire image |
| Per-file hash | After burning to USB | Slow | Identifies specific bad files |

## License

This project is released into the public domain under [The Unlicense](https://unlicense.org/).
