# Windows 10 ISO Verification Guide

A Windows 10 command-line equivalent of the Linux ISO verification workflow.

---

## 1. GPG Signature Verification

Install [Gpg4win](https://www.gpg4win.org/), then use PowerShell:

```powershell
# Import the signing key
gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys <KEY_ID>

# Verify the signature
gpg --verify SHA256SUMS.gpg SHA256SUMS

# Verify the ISO checksum
(Get-Content SHA256SUMS) -match (Get-FileHash your-distro.iso -Algorithm SHA256).Hash.ToLower()
```

Look for `Good signature from ...` in the gpg output. The last command returns the matching line if the hash is correct, or nothing if it fails.

---

## 2. Post-Burn Byte-Level Hashing

Windows has no native `dd`, so use PowerShell to read exactly ISO-sized bytes from the raw USB device:

```powershell
# Hash the ISO
$isoHash = (Get-FileHash "C:\path\to\your-distro.iso" -Algorithm SHA256).Hash
$isoSize = (Get-Item "C:\path\to\your-distro.iso").Length

# Find your USB disk number (look for your USB size)
Get-Disk

# Read exactly ISO-sized bytes from the USB raw device and hash
# Replace 2 with your actual disk number from Get-Disk
$disk = [System.IO.File]::OpenRead("\.\PhysicalDrive2")
$buf  = New-Object byte[] $isoSize
$disk.Read($buf, 0, $isoSize) | Out-Null
$disk.Close()

$sha = [System.Security.Cryptography.SHA256]::Create()
$usbHash = [BitConverter]::ToString($sha.ComputeHash($buf)) -replace '-',''

# Compare
if ($isoHash -eq $usbHash) { "MATCH - USB is a faithful copy" } else { "MISMATCH" }
```

> **Note:** Run PowerShell as Administrator. `\.\PhysicalDrive2` is the raw disk — get the correct number from `Get-Disk`. For large ISOs (4GB+) this loads the entire buffer into RAM; if that is a concern, hash in chunks instead.

---

## 3. Per-File Hash Comparison

Windows can mount ISOs natively. Mount both, then hash every file:

```powershell
# Mount the ISO (Windows mounts it as a drive letter automatically)
$mount = Mount-DiskImage -ImagePath "C:\path\to\your-distro.iso" -PassThru
$isoLetter = ($mount | Get-Volume).DriveLetter + ":"

# Get your USB drive letter (e.g. D:) — check File Explorer or:
Get-Volume

$usbLetter = "D:"  # replace with your USB drive letter

# Hash all files on the ISO
$isoHashes = Get-ChildItem -Recurse -File $isoLetter | ForEach-Object {
    [PSCustomObject]@{
        Hash         = (Get-FileHash $_.FullName -Algorithm SHA256).Hash
        RelativePath = $_.FullName.Substring($isoLetter.Length)
    }
} | Sort-Object RelativePath

# Hash all files on the USB
$usbHashes = Get-ChildItem -Recurse -File $usbLetter | ForEach-Object {
    [PSCustomObject]@{
        Hash         = (Get-FileHash $_.FullName -Algorithm SHA256).Hash
        RelativePath = $_.FullName.Substring($usbLetter.Length)
    }
} | Sort-Object RelativePath

# Compare
$diff = Compare-Object $isoHashes $usbHashes -Property Hash, RelativePath
if ($diff) {
    $diff | Format-Table -AutoSize
} else {
    "All files match."
}

# Unmount the ISO when done
Dismount-DiskImage -ImagePath "C:\path\to\your-distro.iso"
```

The `Compare-Object` output shows:
- `<=` lines: files only on the ISO (missing from USB or hash differs)
- `=>` lines: files only on the USB (extra or hash differs)
- No output means everything matches

---

## Quick Reference

| Method | Windows tool | Requires admin? |
|---|---|---|
| GPG verify | Gpg4win + PowerShell | No |
| Byte-level hash | PowerShell + `\.\PhysicalDrive` | Yes |
| Per-file hash | PowerShell + `Mount-DiskImage` | No |
