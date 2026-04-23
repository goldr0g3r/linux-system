[Home](../README.md) · [↑ 01 Install](README.md) · [← Previous: 01 Install (section)](README.md) · **1.1 Hardware & USB Prep** · [Next: 1.2 Windows prep & BIOS →](02-windows-prep-and-bios.md)

---

# 1.1 Hardware Check and USB Preparation

Confirm the target hardware matches what this guide assumes, download the Kubuntu 26.04 ISO, verify integrity, and write a bootable USB.

## Hardware confirmation checklist (do this from Windows)

From Windows 11 on your TUF A17, run PowerShell as a standard user and verify each line:

```powershell
# CPU
Get-CimInstance Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors
# Expect: "AMD Ryzen 7 4800H with Radeon Graphics" 8 cores 16 threads

# RAM
Get-CimInstance Win32_PhysicalMemory | Select-Object Manufacturer, Capacity, Speed
# Expect: 2 entries totalling 16 GB, speed 3200 MT/s

# Storage
Get-PhysicalDisk | Select-Object DeviceId, FriendlyName, MediaType, Size, BusType
# Expect: 2 NVMe SSDs, 512 GB each. Note which is Disk 0 (usually Windows) and Disk 1 (target for Linux)

# GPU
Get-CimInstance Win32_VideoController | Select-Object Name, DriverVersion
# Expect: "AMD Radeon(TM) Graphics" (iGPU) and "NVIDIA GeForce RTX 3050 Laptop GPU" (dGPU)

# BIOS mode
Confirm-SecureBootUEFI
# Expect: True or False — doesn't matter which, but note it for BIOS step in 1.2
bcdedit /enum {current} | Select-String "path"
# Look for "\Windows\system32\winload.efi" — confirms UEFI (not legacy BIOS)
```

If the CPU, RAM, storage, and GPU do not match the guide baseline, **stop and reassess** — the TUF-specific pages in [2.3](../02-post-install-foundations/03-asus-tuf-hybrid-gpu.md) and [3.4](../03-speed-and-boot-optimizations/04-filesystem-thermals-power.md) will not apply as written.

### Identify which SSD is "Disk B" (the one you are wiping)

This is the single most important identification step. You are about to format a whole disk; confirm it is the correct one.

1. In Windows, open **Disk Management** (`Win+X` → Disk Management).
2. You should see two physical disks. The one containing the Windows system (C:\) is **Disk A**. The other is **Disk B**.
3. Note Disk B's capacity and the serial from PowerShell:
   ```powershell
   Get-PhysicalDisk | Where-Object MediaType -eq SSD | Select-Object DeviceId, SerialNumber, Size, FriendlyName
   ```
   Write the serial number of Disk B somewhere you can see during the installer. The Kubuntu installer will show the serial in the partitioning step.
4. If Disk B currently has any data you value, back it up now. After [1.4](04-installation-and-partitioning.md) it will be gone.

## Other physical checks

- **Battery health:** `powercfg /batteryreport` from PowerShell. If design capacity is under 60 %, consider swapping the battery before committing to a 5-year LTS install. Not a blocker.
- **Fan function:** reboot into Windows, load the CPU/GPU for 60 s (run a video + a stress tool), confirm fans spin up. A dead fan is not something you want to discover on Linux for the first time.
- **BIOS version:** in BIOS setup (F2 at POST), note the BIOS version. If it is older than 2024, consider updating from the [official ASUS support page](https://www.asus.com/supportonly/asus%20tuf%20gaming%20a17%20%28fa706%2C%20amd%20ryzen%204000%20series%29/helpdesk_bios/) **from Windows** before wiping Disk B. BIOS updates on ASUS TUF are safest via the Windows EZ Flash utility; post-Linux-install you lose the easy path.

## Downloading the ISO

### The canonical source and filename

Kubuntu 26.04 LTS ISO is published on `cdimage.ubuntu.com/kubuntu/releases/26.04/release/`. The file you want is the 64-bit desktop ISO, typically named:

```
kubuntu-26.04-desktop-amd64.iso
```

Approximate size: **4.8 GB** (the 26.04 image grew by ~10 % over 24.04 because of Plasma 6.6 assets and the larger kernel tree).

Direct download from a Kubuntu mirror — pick one geographically close, then choose the 26.04 release folder:

- https://cdimage.ubuntu.com/kubuntu/releases/26.04/release/
- https://kubuntu.org/getkubuntu/ — the front page "Get Kubuntu" button, redirects to the above
- A regional mirror from https://launchpad.net/ubuntu/+cdmirrors

Torrent is preferred over HTTPS download if your ISP traffic-shapes large HTTPS: ISOs are signed, so integrity is verified separately.

### Verifying the ISO integrity (non-negotiable)

Download the accompanying `SHA256SUMS` and `SHA256SUMS.gpg` from the same directory. Verify in Windows PowerShell:

```powershell
# In the folder containing the ISO and SHA256SUMS:
Get-FileHash -Algorithm SHA256 .\kubuntu-26.04-desktop-amd64.iso
# Compare the hex string to the line for "kubuntu-26.04-desktop-amd64.iso" inside SHA256SUMS
```

Or, if you prefer GPG-verifying the sums file itself (stronger — the hash file could also be tampered):

```powershell
# In WSL, or in a Kubuntu live USB of a prior release:
gpg --keyid-format long --verify SHA256SUMS.gpg SHA256SUMS
# Good signature from "Ubuntu CD Image Automatic Signing Key (2012) <cdimage@ubuntu.com>"
# or the newer "Ubuntu CD Image Automatic Signing Key <cdimage@ubuntu.com>" (rotated 2025).
```

If the signature does not verify, **delete the ISO and re-download from a different mirror**. Do not proceed.

## Writing the USB

You need an **8 GB or larger** USB stick. Ideally USB 3.0+ (blue port, or a Type-C port) — install time is ~10 minutes on USB 3, ~25 on USB 2.

### Option A: Ventoy (recommended if you install Linux occasionally)

Ventoy is a tool that formats a USB once, then lets you drag-and-drop any number of ISOs onto it. Boot into a menu, pick the one you want. Means this USB becomes a permanent "Linux installer" drive.

1. Download Ventoy for Windows from https://www.ventoy.net (grab `ventoy-X.Y.ZZ-windows.zip`).
2. Extract, run `Ventoy2Disk.exe` as administrator.
3. **Select your USB stick in the Device dropdown.** Confirm it is the USB, not any internal drive.
4. In the **Option** menu:
   - Partition style: **GPT**
   - Secure Boot support: **ON** (Ventoy 1.0.95+ signs its `grubx64.efi` with a key that most UEFI shim-loaders accept)
5. Click **Install**. Confirm the two warnings. Takes ~30 seconds.
6. When done, the USB is partitioned into an `EFI Ventoy` partition (tiny) + an exFAT data partition.
7. Copy `kubuntu-26.04-desktop-amd64.iso` onto the exFAT partition via Windows Explorer.

Ventoy will auto-detect the ISO on boot and show a menu.

### Option B: `dd`-style flash (recommended if this is a one-time install)

Simpler, works on any machine, but the USB becomes single-ISO until you re-flash it.

#### From Windows

Use **Rufus 4.5+** (https://rufus.ie/):

1. Insert the USB.
2. Open Rufus, pick the USB under **Device**.
3. **Select** the Kubuntu ISO.
4. **Partition scheme**: `GPT`. **Target system**: `UEFI (non-CSM)`.
5. **File system**: leave at default (FAT32).
6. **Write in:** `DD Image mode` (Rufus will ask; **pick DD Image mode, NOT ISO Image mode**). This avoids Rufus's "helpful" repartitioning which breaks Kubuntu's built-in UEFI shim.
7. Click **Start**, accept both warnings. Takes ~5 minutes on USB 3.

#### From a Linux system

If you already have a Linux box handy:

```bash
# Find the USB device (plug it in, then):
lsblk -o NAME,SIZE,MODEL,TRAN
# Look for a TRAN=usb entry (e.g. /dev/sdc). DO NOT confuse with /dev/nvme0n1 etc.

# Flash. Replace sdX with the USB device:
sudo dd if=kubuntu-26.04-desktop-amd64.iso of=/dev/sdX bs=4M status=progress conv=fsync oflag=direct
sync
```

Done. Unplug and replug to confirm Windows sees a new boot-volume-shaped partition called `KUBUNTU_26_0`.

### Verifying the USB boots before proceeding

1. Plug the USB into the TUF A17.
2. Power on, mash **F8** (ASUS one-time boot menu).
3. You should see entries for:
   - The Windows SSD (usually `Windows Boot Manager`)
   - The USB drive (either `UEFI: Ventoy` or `UEFI: SanDisk` / `UEFI: Kingston` etc.)
4. Select the USB. You should see the Kubuntu 26.04 boot splash in ~5 seconds.
5. If it does not boot at all, check:
   - Secure Boot is **disabled** in BIOS (we will re-enable selectively in [1.2](02-windows-prep-and-bios.md), but for install it is simplest to be off).
   - USB booted from was the correct port (some TUF A17 models have a flaky front USB-A port; try the back ones).

**Do not proceed past the Kubuntu splash screen yet.** Power off and continue to [1.2](02-windows-prep-and-bios.md) to prep Windows before the live environment.

## What to do with the USB afterwards

Keep it. For this LTS cycle (April 2026 → April 2031) you will:

- Need it if Kubuntu ever fails to boot and you need to rescue from the live environment.
- Need it if you want to do a fresh install on another machine.
- Re-flash it with Kubuntu 26.04.1 or 26.04.2 point-release ISOs when they come out (if you want to install from a newer media, though you can always upgrade in-place).

Label it physically ("Kubuntu 26.04 LTS, SHA ends xxxx, Apr 2026") with a Sharpie on the side.

---

[Home](../README.md) · [↑ 01 Install](README.md) · [← Previous: 01 Install (section)](README.md) · **1.1 Hardware & USB Prep** · [Next: 1.2 Windows prep & BIOS →](02-windows-prep-and-bios.md)
