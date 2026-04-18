[Home](../../README.md) · [← Previous: The Verdict](../01-verdict.md) · **Part 2: Setup Guide** · [Next: 2.1 Windows Prep →](01-windows-prep.md)

---

# Part 2 — The Ultimate OS Setup Guide

Scratch-to-powerhouse installation and tuning for a Kubuntu 24.04 LTS dual-boot on an ASUS TUF A17.

## Setup Model

This part assumes two SSDs:

- **Disk A** = Windows 11, already installed.
- **Disk B** = blank, target for Linux.

We install Linux to Disk B only. Linux's bootloader goes onto Disk B's ESP, Windows keeps its own ESP on Disk A. You select the OS at power-on with **F8** (ASUS one-time boot menu). This avoids any shared GRUB that could break on a Windows feature update.

## Steps (follow in order)

| # | Step | Time | Can be skipped? |
|---|------|------|----------------|
| 2.1 | [Pre-install: Windows 11 prep](01-windows-prep.md) | 5 min | No |
| 2.2 | [BIOS / UEFI configuration for ASUS TUF A17](02-bios-uefi.md) | 5 min | No |
| 2.3 | [Installation & partitioning (BTRFS subvolumes, systemd-boot)](03-install-partitioning.md) | 30–60 min | No |
| 2.4 | [First-boot debloat (snap removal, Flatpak, KDE PIM pruning)](04-debloat.md) | 10 min | No |
| 2.5 | [Kernel & boot optimization (zram, sysctl, service masking)](05-kernel-boot.md) | 10 min | Partially — skim if already tuned |
| 2.6 | [ASUS TUF A17 hardware enablement (`asusctl`, `supergfxctl`)](06-asus-hardware.md) | 15 min | No — TUF laptops need this |
| 2.7 | [UI/UX tweaks for engineers (KWin tiling, Activities, Konsole)](07-ui-ux.md) | 20 min | Yes, but recommended |

Total unattended time: ~2 hours including installer, reboots, and package downloads. The [provisioning script in Appendix A](../appendix-a-provisioning.md) can replace most of steps 2.4, 2.5, and 2.7 with an unattended run.

## Prerequisites

- Kubuntu 24.04 LTS live USB (write via `balenaEtcher` or `sudo dd if=kubuntu-24.04.4-desktop-amd64.iso of=/dev/sdX bs=4M status=progress`).
- A second working device (phone, spare laptop) in case you need to reference this guide mid-install.
- Wi-Fi credentials handy.
- ~2 hours of uninterrupted time.
- Back up anything on the Windows SSD — you should not touch it, but "should not" is not "cannot".

## What This Part Does NOT Cover

- LUKS full-disk encryption — intentionally omitted. For a personal dev laptop with Secure Boot off and no regulated data, the threat model does not justify the boot complexity. Add it later if needed.
- Hibernation configuration — covered implicitly via the 20 GiB swapfile in [2.3](03-install-partitioning.md) but not wired up to the kernel cmdline; enable it later per your power preferences.
- NVIDIA driver installation — that is in [Part 3.3](../03-dev-environment/03-ai-ml.md) because CUDA is bundled in.

---

[Home](../../README.md) · [← Previous: The Verdict](../01-verdict.md) · **Part 2: Setup Guide** · [Next: 2.1 Windows Prep →](01-windows-prep.md)
