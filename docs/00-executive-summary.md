[Home](../README.md) · **Executive Summary** · [Next: The Verdict →](01-verdict.md)

---

# Executive Summary

**Target hardware:** ASUS TUF Gaming A17 (AMD Ryzen 7 4800H, NVIDIA RTX 3050 Mobile 4 GB, 16 GB DDR4, 2x 512 GB NVMe SSD)
**Target workload:** Automotive HW/SW integration today, marine robotics (ROV/AUV, subsea systems) tomorrow, M.Tech (AI) coursework throughout, plus a self-hosted Next.js blog on Vercel.
**Decision scope:** Kubuntu 24.04 LTS vs Ubuntu 24.04 LTS, full installation from scratch, and a reproducible developer environment.

> Factual correction up-front: Kubuntu 24.04 LTS ships **KDE Plasma 5.27 LTS**, not Plasma 6. Plasma 6 first landed in Kubuntu 24.10. This document compares the versions you can actually install today on a supported LTS (Ubuntu 24.04 "Noble Numbat" + Plasma 5.27 LTS on Kubuntu 24.04 "Noble Numbat"). Upgrading 24.04 to Plasma 6 via the KDE backports PPA is covered in [Part 2](02-setup/README.md) as an optional advanced step.

---

## The Verdict in One Paragraph

**The winner is Kubuntu 24.04 LTS.** You will run it with the X11 session (not Wayland) on the proprietary NVIDIA 580 driver, BTRFS + zstd as the root filesystem, systemd-boot as the loader, `uv` for Python, Podman + Distrobox for ROS isolation, and `asusctl` / `supergfxctl` from `asus-linux.org` for TUF-specific hardware control.

Why, in one paragraph: your workload is *heterogeneous* (embedded toolchains, C++ compiles, AI training, KiCad, web dev, VM/container sprawl). You need an OS that (a) stays out of the way, (b) does not impose Snap's confinement or autoupdate on your Firefox/IDE/terminal, (c) hands you a tiling + virtual-desktop workflow without fighting GNOME extension rot, and (d) survives NVIDIA driver updates without mysterious Wayland regressions. Kubuntu 24.04 LTS delivers all four; Ubuntu 24.04 delivers the last one (marginally) and loses the first three.

Full reasoning with six axes of comparison is in [Part 1 — The Verdict (v1)](01-verdict.md).

> **2026 landscape update (Apr 18, 2026):** A re-evaluation against Kubuntu 26.04 LTS "Resolute Raccoon" (ships Apr 23), Fedora 43 KDE Spin, and EndeavourOS "Ganymede Neo" concludes: **install Kubuntu 24.04 LTS today, plan the upgrade to Kubuntu 26.04.1 LTS for ~August 2026.** Full reasoning: [Part 1 (v2) — 2026 Re-verdict](01-verdict_v2.md) · [/best — Distro Landscape Research](best/README.md). The 24.04 install procedure in Part 2 below remains correct and unchanged.

---

## 10-Step Quickstart

Full explanations are in [Part 2](02-setup/README.md). This is the at-a-glance order of operations.

1. **Windows 11 prep:** disable Fast Startup, suspend BitLocker, confirm BIOS in AHCI. See [2.1 Windows prep](02-setup/01-windows-prep.md).
2. **Boot live USB:** Kubuntu 24.04 LTS; pick the Linux-only SSD in the installer. See [2.2 BIOS](02-setup/02-bios-uefi.md).
3. **Manual partition:** 1 GiB ESP / 2 GiB `/boot` (ext4) / rest BTRFS with subvolumes. See [2.3 Partitioning](02-setup/03-install-partitioning.md).
4. **Finish install:** Calamares puts GRUB on the Linux SSD's ESP, not Windows's. See [2.3.2 Optional systemd-boot conversion](02-setup/03-install-partitioning.md).
5. **First boot:** `sudo apt update && sudo apt full-upgrade`, reboot.
6. **Debloat:** nuke snap, enable Flatpak + Flathub, prune KDE PIM. See [2.4 Debloat](02-setup/04-debloat.md).
7. **NVIDIA 580:** install via `ubuntu-drivers`, reboot, verify `prime-run glxinfo`. See [3.3 AI/ML](03-dev-environment/03-ai-ml.md).
8. **ASUS:** install `asusctl` + `supergfxctl` via PPA (or source). See [2.6 ASUS hardware](02-setup/06-asus-hardware.md).
9. **Toolchain:** install CUDA 13.2, cuDNN 9.21, Podman, Distrobox, KiCad 9, `uv`, `fnm`. See [Part 3](03-dev-environment/README.md).
10. **Snapshot:** Timeshift/Snapper before touching anything else. See [2.5 Kernel/boot](02-setup/05-kernel-boot.md).

For the unattended version, the [one-shot provisioning script in Appendix A](appendix-a-provisioning.md) automates steps 5–10.

---

## Non-Negotiable Technical Positions

These are the positions this guide takes. If you want a different choice on any of these, the reasoning for each is spelled out in the referenced section.

| Decision | Value | Why — full explanation in |
|----------|-------|---------------------------|
| Distro | Kubuntu 24.04 LTS | [Part 1](01-verdict.md) |
| Session | X11 (not Wayland) | [Part 1 — Wayland section](01-verdict.md#wayland--nvidia-rtx-3050-mobile-compatibility) |
| Filesystem | BTRFS + zstd:1 subvolumes | [Part 2.3](02-setup/03-install-partitioning.md) |
| Bootloader | systemd-boot (GRUB acceptable) | [Part 2.3.2](02-setup/03-install-partitioning.md) |
| Swap | 20 GiB BTRFS swapfile + zram | [Parts 2.3 and 2.5](02-setup/03-install-partitioning.md) |
| NVIDIA driver | proprietary 580 + PRIME offload | [Part 3.3](03-dev-environment/03-ai-ml.md) |
| TUF hardware control | `asusctl` + `supergfxctl` | [Part 2.6](02-setup/06-asus-hardware.md) |
| Python | `uv` (replaces pip + venv + poetry) | [Part 3.3](03-dev-environment/03-ai-ml.md) |
| Node | `fnm` (replaces `nvm`) | [Part 3.4](03-dev-environment/04-web.md) |
| Container runtime | rootless Podman + Distrobox | [Part 3.1](03-dev-environment/01-containers.md) |

---

[Home](../README.md) · **Executive Summary** · [Next: The Verdict →](01-verdict.md)
