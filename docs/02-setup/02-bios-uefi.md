[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.1 Windows Prep](01-windows-prep.md) · **2.2 BIOS / UEFI** · [Next: 2.3 Partitioning →](03-install-partitioning.md)

---

# 2.2 BIOS / UEFI Configuration for ASUS TUF A17

Reboot, hammer **F2** to enter BIOS. Set the following values.

## Required BIOS Settings

| Setting | Value | Reason |
|---------|-------|--------|
| Secure Boot | **Disabled** | Simplest path for NVIDIA DKMS signing. Re-enable later with `mokutil` if you want. |
| Fast Boot | Disabled | Prevents skipping USB enumeration on cold boot. |
| CSM | Disabled | UEFI-only, no legacy boot. |
| SATA Mode | AHCI | Do not switch to RAID / Intel RST — Linux will not see the NVMe. |
| Boot Order | Leave Windows as default | You will use **F8** at power-on to pick Linux when you want it. |

## Why each of these

- **Secure Boot disabled** — DKMS-signed NVIDIA modules and a handful of ROS/embedded kernel modules need unsigned loading. You can sign modules yourself with `mokutil` later, but that is a whole project. For a dev laptop, disable it.
- **Fast Boot disabled** — ASUS's Fast Boot skips USB controller enumeration on cold boot. The Kubuntu live USB is then invisible until you reboot once. Just turn it off.
- **CSM (Compatibility Support Module) disabled** — CSM enables legacy BIOS boot. We are doing pure UEFI. Leaving CSM on can make the installer grub fall into legacy mode by accident and produce an unbootable system.
- **SATA Mode AHCI** — This is the single most common installer failure on ASUS/Intel laptops. RAID / Intel RST hides the NVMe behind a controller abstraction that Linux's `nvme` module does not understand. You will see "no disks found" in Calamares. Switch to AHCI (it is safe; Windows will continue to boot fine in AHCI mode if it was installed in AHCI; if Windows was installed in RAID mode, a switch to AHCI requires a Windows-side `bcdedit` adjustment documented by Microsoft).
- **Boot order untouched** — We are not modifying the firmware boot order. Each power-on goes to Windows by default. When you want Linux, press **F8** during the TUF splash to open the ASUS one-time boot menu, and pick the Linux SSD.

## If Windows won't boot after switching to AHCI

This means Windows was installed in RAID mode. From Windows:

```powershell
bcdedit /set {current} safeboot minimal
```

Reboot into BIOS, switch to AHCI, boot Windows into Safe Mode, then:

```powershell
bcdedit /deletevalue {current} safeboot
```

Reboot normally — Windows auto-installs the AHCI driver and is now on AHCI permanently.

## Boot the live USB

Save BIOS settings and exit. Insert the Kubuntu 24.04 LTS live USB (already written with `balenaEtcher` or `dd`), power on, press **F8** during the ASUS splash, pick the USB.

When the live session boots, confirm:

- Wi-Fi works (network icon in the system tray).
- Keyboard backlight and touchpad work.
- The second SSD is visible: open KDE Partition Manager or run `lsblk` — you should see `nvme0n1` (Windows, with Microsoft partitions) and `nvme1n1` (blank, your target).

If any of these fail, stop and troubleshoot before proceeding to [2.3 Partitioning](03-install-partitioning.md). A missing second SSD almost always means you are still on RAID mode (re-check BIOS) or the SSD is not seated — power down, open the bottom panel, reseat the M.2 drive.

---

[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.1 Windows Prep](01-windows-prep.md) · **2.2 BIOS / UEFI** · [Next: 2.3 Partitioning →](03-install-partitioning.md)
