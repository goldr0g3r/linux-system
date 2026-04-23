[Home](../README.md) · [↑ 01 Install](README.md) · [← Previous: 1.1 Hardware & USB](01-hardware-check-and-usb-prep.md) · **1.2 Windows prep & BIOS** · [Next: 1.3 Installer flavor →](03-installer-flavor-minimal-vs-full.md)

---

# 1.2 Windows 11 Preparation + BIOS/UEFI Configuration

Two things to do before you boot the Kubuntu live USB for real:

1. **Windows side** — tell Windows to release its grip on the disk subsystem so the Linux installer sees a consistent view, and ensure Windows will boot cleanly after the dual-boot is in place.
2. **BIOS side** — put the firmware in a state where the Kubuntu installer can land its bootloader on Disk B's ESP without fighting Secure Boot or AHCI vs RAID mode issues.

## Part 1 — Windows 11 preparation

### 1.2.1 Disable Fast Startup (mandatory)

Fast Startup is a hybrid-shutdown feature: instead of a real power-off, Windows hibernates kernel + drivers to `hiberfil.sys` and "boots" by restoring that image. It's harmless on Windows-only systems but catastrophic on dual-boot: if Linux (or a Kubuntu live environment) mounts an NTFS partition that was "shut down" by Fast Startup, it corrupts the hibernated image. Disabling Fast Startup is non-negotiable for dual-boot.

```powershell
# Open PowerShell as Administrator.
powercfg /hibernate off
```

Then also manually disable Fast Startup in the GUI so Windows cannot silently turn hibernate back on during updates:

1. Control Panel → Power Options → **Choose what the power buttons do**.
2. Click **Change settings that are currently unavailable**.
3. Uncheck **Turn on fast startup (recommended)**. Save.

### 1.2.2 Suspend BitLocker (if enabled)

Windows 11 Home and Pro both now turn on Device Encryption (a BitLocker variant) by default during the initial setup. If your C:\ drive is encrypted, the installer **should** not touch it, but we are going to be flashing firmware-adjacent things (the BIOS boot order) and BitLocker can panic and demand the recovery key on next boot.

```powershell
# Check if BitLocker is on:
manage-bde -status C:
# Look for "Protection Status: Protection On" and "Percentage Encrypted: 100%"
```

If it is on:

```powershell
# Suspend BitLocker for 3 reboots (enough for install + first boot verifications):
manage-bde -protectors -disable C: -RebootCount 3
```

Also: **write down the BitLocker recovery key** right now in case suspend is not honoured. Sign in to https://account.microsoft.com/devices/recoverykey and save the 48-digit numeric key somewhere you can read it without a working Windows machine (password manager, printed paper).

### 1.2.3 Confirm AHCI mode (not Intel RST / RAID)

In BIOS, the SATA/NVMe controller can be in AHCI mode or RAID (Intel RST / AMD RAIDXpert). RAID mode hides the actual NVMe from the Linux installer. You must be in AHCI.

- On TUF A17 (AMD chipset), the controller mode is usually **AHCI** by default — this is not the Intel world where RST is opt-in.
- Confirm in BIOS (F2 at POST): **Advanced → SATA Configuration → SATA Mode**. Should read **AHCI**.
- If it reads RAID, Windows may have been installed on RAID. **Switching modes without preparing Windows crashes it on next boot.** If this is your case, follow Microsoft's "[Enable AHCI mode without reinstalling Windows 11](https://learn.microsoft.com/en-us/troubleshoot/windows-client/deployment/error-0x0000007b-when-moving-hard-disk-to-another-computer)" guide first, or reinstall Windows in AHCI before continuing.

### 1.2.4 Free up target space on Disk B (if it is not already blank)

Disk B is what Kubuntu will use. If it currently holds Windows data (a second-drive D:\, Games folder, etc.):

1. Back up anything important off the machine.
2. In Disk Management, **delete all volumes on Disk B**. It should become one big "Unallocated" space.
3. You do NOT need to convert Disk B to GPT from Windows — the Kubuntu installer's partitioner creates a fresh GPT when you tell it to use the whole disk.

### 1.2.5 Confirm Windows boots cleanly

Do a clean shutdown (Start → Power → **Shut down**, *not* Restart). Let the machine power off fully. Wait 10 seconds. Power back on to Windows, confirm the desktop appears without errors. **Then** shut down again (clean shutdown — Fast Startup is off, so this will be a true power-off).

Now the machine is in the correct state to boot the installer.

## Part 2 — BIOS/UEFI configuration (ASUS TUF A17)

Boot into BIOS setup by pressing **F2** repeatedly at POST. (Older ASUS firmware used DEL; TUF A17 2020 uses F2.) You'll land in the **EZ Mode** summary screen. Press **F7** to switch to **Advanced Mode**.

### 1.2.6 Secure Boot — turn OFF for install

Secure Boot + Kubuntu works fine in steady-state (Canonical ships a signed `shim` that most TUF firmwares accept), but it is the single most common cause of "installer boots OK but installed system won't" problems. Turn it off for install; if you want to re-enable it after [1.5](05-first-boot-and-snapshot.md) you can, but for this LTS cycle I recommend leaving it off on this TUF generation — the benefit vs the complexity of MOK (Machine Owner Key) enrollment for NVIDIA DKMS modules is tiny on a laptop without regulated data.

- **Advanced Mode** → **Boot** → **Secure Boot**.
- **OS Type: Windows UEFI Mode** (we are UEFI, even though it says Windows).
- **Secure Boot Mode: Custom**.
- **Key Management → Clear Secure Boot keys** (optional — you are turning Secure Boot off entirely below; this just avoids stale PK/KEK entries).
- Go up one level, set **Secure Boot: Disabled**.

### 1.2.7 TPM — leave ON

Windows 11 requires TPM 2.0; leave it enabled. Linux will not use it (unless you configure `systemd-cryptenroll` for LUKS, which we are not doing).

- **Advanced → Trusted Computing → Security Device Support: Enabled**.
- **AMD fTPM switch: Firmware TPM**.

### 1.2.8 CSM / Legacy Boot — DISABLED

- **Boot → CSM (Compatibility Support Module) → Launch CSM: Disabled**.

If CSM is enabled, Kubuntu may install as a BIOS/MBR system instead of UEFI/GPT, which breaks dual-boot with a UEFI Windows 11. Keep this OFF.

### 1.2.9 Fast Boot (BIOS) — DISABLED

TUF A17's BIOS has its own "Fast Boot" option, separate from Windows Fast Startup. Disable it so F8 / F2 / Esc are reliable during POST.

- **Boot → Fast Boot: Disabled**.

### 1.2.10 Boot Option #1 — keep Windows for now

After install, you'll set this to **Kubuntu** (Disk B's ESP). For now, leave it pointing at Windows Boot Manager on Disk A. The F8 one-time boot menu is what you use to reach the USB.

- **Boot → Boot Option Priorities → Boot Option #1: Windows Boot Manager**.

### 1.2.11 (TUF-specific) Virtualization + IOMMU

These don't affect install but are useful later for Podman/Distrobox, qemu, and VFIO. Enable now so you don't have to reboot into BIOS again.

- **Advanced → CPU Configuration → SVM Mode: Enabled** (AMD virtualisation).
- **Advanced → AMD CBS → NBIO Common Options → IOMMU: Enabled**.

### 1.2.12 Save and exit

F10 → confirm → machine reboots.

### 1.2.13 Verify F8 still works

At POST, mash **F8**. The one-time boot menu should appear with entries for Windows Boot Manager and the USB stick. Select the USB.

If the machine boots straight to Windows without showing F8, re-enter BIOS and double-check Fast Boot (BIOS) is Disabled.

## Summary: the Windows/BIOS pre-install state

Your machine is now:

- Windows Fast Startup **off**.
- BitLocker **suspended for 3 reboots** (or **off**).
- SATA in **AHCI** mode.
- Secure Boot **off**, CSM **off**, BIOS Fast Boot **off**.
- TPM, SVM (AMD-V), IOMMU **on**.
- F8 one-time boot menu reliably shows USB and Windows entries.
- Disk B is either empty or has data you do not care about.

If any of those is not true, fix it before [1.3](03-installer-flavor-minimal-vs-full.md). You cannot safely recover from "I booted the installer with BitLocker active on Disk A" without the recovery key.

---

[Home](../README.md) · [↑ 01 Install](README.md) · [← Previous: 1.1 Hardware & USB](01-hardware-check-and-usb-prep.md) · **1.2 Windows prep & BIOS** · [Next: 1.3 Installer flavor →](03-installer-flavor-minimal-vs-full.md)
