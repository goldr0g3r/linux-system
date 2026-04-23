[Home](../README.md) · [← Previous: Executive Summary](../00-executive-summary.md) · **01 — Install** · [Next: 1.1 Hardware & USB →](01-hardware-check-and-usb-prep.md)

---

# 01 — Prerequisites and Installation

Scratch-to-first-boot install of Kubuntu 26.04 LTS "Resolute Raccoon" on your ASUS TUF A17, preserving an untouched Windows 11 on Disk A.

## Setup model (identical to the `docs/` layout you already run)

- **Disk A** = Windows 11, already installed, untouched by everything in this guide.
- **Disk B** = blank (or to-be-wiped), target for Kubuntu 26.04.
- Kubuntu's bootloader lands on **Disk B's own ESP**. Windows's ESP on Disk A remains the Windows-only ESP.
- You pick the OS at power-on with **F8** (ASUS one-time boot menu). There is no shared GRUB and no cross-drive boot entry. This avoids the classic "Windows feature update breaks GRUB" failure mode.

## Steps (do these in order)

| #   | Step                                                                          | Time     | Skippable?                         |
| ---:| ----------------------------------------------------------------------------- | -------- | ---------------------------------- |
| 1.1 | [Hardware check and USB preparation](01-hardware-check-and-usb-prep.md)       | 20 min   | No                                 |
| 1.2 | [Windows 11 prep + BIOS/UEFI configuration](02-windows-prep-and-bios.md)      | 15 min   | No                                 |
| 1.3 | [Installer flavor: Minimal vs Normal vs Extended](03-installer-flavor-minimal-vs-full.md) | 5 min    | No — the flavor you pick is consequential |
| 1.4 | [Installation & partitioning (BTRFS subvolumes, systemd-boot)](04-installation-and-partitioning.md) | 30–60 min | No          |
| 1.5 | [First boot, updates, and initial Timeshift snapshot](05-first-boot-and-snapshot.md) | 15 min | No                                 |

Total wall-clock: ~2 hours including downloads and reboots.

## Prerequisites you need to have before starting

- A second device (phone, spare laptop) to read this guide mid-install.
- Wi-Fi credentials or a wired ethernet cable at the install location.
- ~8 GB USB stick. (Ventoy is multi-ISO; `dd` is single-ISO. Both documented in [1.1](01-hardware-check-and-usb-prep.md).)
- Up-to-date backup of any important data on Disk A (Windows). You should not touch it, but "should not" is not "cannot."
- ~2 hours of uninterrupted time, plus a 10-minute buffer if your ISP is slow.

## What this section deliberately does NOT cover

- LUKS full-disk encryption — see [00 Executive Summary § non-goals](../00-executive-summary.md#what-this-guide-does-not-cover-and-why).
- Installing NVIDIA drivers or CUDA — those live in [2.2](../02-post-install-foundations/02-nvidia-driver-580-and-cuda.md) because they depend on a working apt after first boot.
- ASUS `asusctl`/`supergfxctl` — in [2.3](../02-post-install-foundations/03-asus-tuf-hybrid-gpu.md) for the same reason.

---

[Home](../README.md) · [← Previous: Executive Summary](../00-executive-summary.md) · **01 — Install** · [Next: 1.1 Hardware & USB →](01-hardware-check-and-usb-prep.md)
