[Home](README.md) · **Executive Summary** · [Next: 01 — Install →](01-install/README.md)

---

# Executive Summary

**Target:** ASUS TUF Gaming A17 (Ryzen 7 4800H · RTX 3050 Mobile 4 GB · 16 GB · 2x 512 GB NVMe) dual-booted with Windows 11.
**Distro:** Kubuntu 26.04 LTS "Resolute Raccoon", released **2026-04-23** (today).
**Daily workload:** Heavy browsing, video, Bluetooth-headset audio, music streaming, web development. First-class support for ROS 2 and AI/ML on top.
**Support horizon:** April 2031 (standard) · April 2036 (Ubuntu Pro, free for 5 personal machines) · April 2038 (Legacy Support).

---

## The verdict in one paragraph

**Install Kubuntu 26.04 LTS.** Run the default Plasma 6.6 Wayland session on the proprietary `nvidia-driver-580` with Explicit Sync, BTRFS + zstd:1 subvolumes, systemd-boot, `sudo-rs`, rootless Podman + Distrobox, and `uv` for Python. For the first 29 days (until ROS 2 Lyrical Luth ships on 2026-05-22) run ROS 2 **Jazzy in a Distrobox against an Ubuntu 24.04 base**; after 2026-05-22, `apt install ros-lyrical-desktop` **natively on the 26.04 host** and keep the Distrobox around for legacy Humble/Jazzy work. For AI/ML use the new `nvidia-cuda-toolkit` that ships in the Ubuntu archive (one `apt install` instead of the old NVIDIA-repo dance); pin your PyTorch venv to Python 3.12 until 3.14 wheels stabilise. Daily driver apps go on via **APT (no snap)** for anything with a trustworthy upstream repo (Firefox, Brave, VS Code, Cursor) and **Flatpak** for everything else (Spotify, Discord, Signal, Obsidian, Zen Browser).

Why this is different from the [`docs/` v2 verdict](../docs/01-verdict_v2.md) that said "wait for .1": you are choosing to accept `.0`-release risk in exchange for a clean start on the 5-year LTS window. The [README](README.md#eyes-open-risk-acceptance-why-we-are-installing-0-today) lists the four specific risks and their mitigations.

---

## 15-step quickstart

Full explanations are in the respective sections. This is the order-of-operations you will execute.

| # | Step                                                                                                       | Time      | Section                                                                                   |
| -:|:---------------------------------------------------------------------------------------------------------- |:--------- |:----------------------------------------------------------------------------------------- |
|  1 | Verify hardware, download Kubuntu 26.04 ISO, checksum, write to USB (Ventoy or `dd`)                      | 20 min    | [1.1](01-install/01-hardware-check-and-usb-prep.md)                                       |
|  2 | Windows 11 prep (Fast Startup off, BitLocker suspend, AHCI) + BIOS (Secure Boot off, TPM decision)        | 15 min    | [1.2](01-install/02-windows-prep-and-bios.md)                                             |
|  3 | Pick the **Minimal** install flavor + *install third-party codecs* ON, *download updates during install* OFF | 2 min   | [1.3](01-install/03-installer-flavor-minimal-vs-full.md)                                  |
|  4 | Manual partition of Disk B: 1 GiB ESP / 2 GiB `/boot` ext4 / rest BTRFS with subvolumes; Calamares does the rest | 25 min | [1.4](01-install/04-installation-and-partitioning.md)                                     |
|  5 | First boot, `sudo apt update && sudo apt full-upgrade`, reboot, **first Timeshift snapshot**               | 15 min    | [1.5](01-install/05-first-boot-and-snapshot.md)                                           |
|  6 | Debloat: nuke snap, enable Flatpak + Flathub, prune KDE PIM, set Firefox via Mozilla APT                  | 10 min    | [2.1](02-post-install-foundations/01-debloat-snap-flatpak-pim.md)                         |
|  7 | `sudo ubuntu-drivers install` for NVIDIA 580, then `sudo apt install nvidia-cuda-toolkit cudnn9-cuda-13`; reboot | 20 min | [2.2](02-post-install-foundations/02-nvidia-driver-580-and-cuda.md)                       |
|  8 | ASUS TUF enablement: `asus-linux.org` PPA, `asusctl` + `supergfxctl`, 80 % battery cap, fan curves        | 15 min    | [2.3](02-post-install-foundations/03-asus-tuf-hybrid-gpu.md)                              |
|  9 | Wayland / Plasma 6.6 / NVIDIA polish (Explicit Sync confirmed, fractional scaling, OBS PipeWire portal)    | 15 min    | [2.4](02-post-install-foundations/04-wayland-plasma-66-nvidia.md)                         |
| 10 | Speed + boot optimisations (kernel params, zram, swappiness, service masking, journald cap, fstrim)       | 20 min    | [3.1–3.4](03-speed-and-boot-optimizations/README.md)                                      |
| 11 | Daily driver: browsers, mpv / VLC, PipeWire LDAC/AptX HD, Bluetooth headset codec priority, Spotify Flatpak | 30 min  | [4.1–4.6](04-daily-driver-stack/README.md)                                                |
| 12 | Web dev: Zsh + Starship, Git/SSH/GPG with Ed25519+ML-KEM, `fnm` + Corepack, VS Code + Cursor, Postgres Quadlet | 45 min | [5.1–5.5](05-web-development/README.md)                                                   |
| 13 | ROS 2 **bridge phase**: Podman rootless + Distrobox + `ros2-jazzy` container on Ubuntu 24.04 base         | 30 min    | [6.3](06-ros2-robotics/03-distrobox-bridge-jazzy-now.md)                                  |
| 14 | AI/ML: `uv`, Python 3.12 AI venv, PyTorch cu130, Ollama, Open WebUI Quadlet, Continue.dev wired to VS Code | 45 min   | [7.1–7.3](07-ai-ml-workstation/README.md)                                                 |
| 15 | Productivity + security: KeyD CapsLock→Ctrl/Esc, Restic backup to offsite, UFW, Firejail for Chromium     | 30 min    | [8.1–8.3](08-productivity-security/README.md)                                             |

Unattended replay of steps 6–15 is automated in [99-provisioning-script.md](99-provisioning-script.md).

**Deferred until 2026-05-22:** Replace the Distrobox bridge phase with a native `apt install ros-lyrical-desktop` on the 26.04 host. Checklist in [6.4](06-ros2-robotics/04-native-lyrical-after-may-22.md).

---

## Non-negotiable technical positions

If you want a different choice on any of these, the reasoning for each is spelled out in the linked section.

| Decision                        | Value                                                                                 | Reasoning in                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Distro                          | Kubuntu 26.04 LTS                                                                     | [README eyes-open risk](README.md#eyes-open-risk-acceptance-why-we-are-installing-0-today) |
| Installer flavor                | **Minimal** + codecs ON + updates-during-install OFF                                  | [1.3](01-install/03-installer-flavor-minimal-vs-full.md)                         |
| Session                         | Wayland (Plasma 6.6); X11 only as troubleshooting fallback                            | [2.4](02-post-install-foundations/04-wayland-plasma-66-nvidia.md)                |
| Filesystem                      | BTRFS + `noatime,compress=zstd:1,space_cache=v2,discard=async` + subvolumes           | [1.4](01-install/04-installation-and-partitioning.md) and [3.4](03-speed-and-boot-optimizations/04-filesystem-thermals-power.md) |
| Bootloader                      | systemd-boot (GRUB acceptable but unused)                                             | [1.4](01-install/04-installation-and-partitioning.md)                            |
| Swap                            | 20 GiB BTRFS swapfile + zram 50 % RAM                                                 | [3.2](03-speed-and-boot-optimizations/02-zram-swap-sysctl.md)                    |
| NVIDIA driver                   | `nvidia-driver-580` (proprietary, not `-open`) + PRIME offload                        | [2.2](02-post-install-foundations/02-nvidia-driver-580-and-cuda.md)              |
| CUDA install                    | Native `nvidia-cuda-toolkit` from Ubuntu archive — **no NVIDIA APT repo on the host** | [7.1](07-ai-ml-workstation/01-cuda-13-cudnn-native.md)                           |
| TUF hardware                    | `asusctl` + `supergfxctl` via `asus-linux.org` PPA                                    | [2.3](02-post-install-foundations/03-asus-tuf-hybrid-gpu.md)                     |
| Python                          | `uv` for everything; AI venvs pin `--python 3.12`                                     | [7.2](07-ai-ml-workstation/02-python-uv-pytorch-jax.md)                          |
| Node                            | `fnm` + Corepack (pnpm primary)                                                       | [5.3](05-web-development/03-runtimes-and-package-managers.md)                    |
| Container runtime               | Rootless Podman + Distrobox                                                           | [6.3](06-ros2-robotics/03-distrobox-bridge-jazzy-now.md)                         |
| ROS 2                           | Distrobox + Jazzy now; native Lyrical after 2026-05-22                                | [6 intro](06-ros2-robotics/README.md)                                            |
| Backup                          | Timeshift = rollback only; real backups = Restic → offsite                            | [8.2](08-productivity-security/02-backup-strategy-3-2-1.md)                      |
| Power manager                   | `power-profiles-daemon` (NOT TLP) because `asusctl` integrates with PPD on 26.04      | [3.4](03-speed-and-boot-optimizations/04-filesystem-thermals-power.md)           |
| Audio                           | PipeWire + WirePlumber + `libldac` + `libfreeaptx` (LDAC / AptX HD on BT)             | [4.2](04-daily-driver-stack/02-video-audio-pipewire.md), [4.3](04-daily-driver-stack/03-bluetooth-headset-hifi.md) |
| Shell                           | Zsh + Starship (fish as alternative)                                                  | [5.1](05-web-development/01-shell-and-terminal.md)                               |

---

## What this guide does not cover (and why)

- **LUKS full-disk encryption** — threat model for a personal dev laptop without regulated data does not justify boot complexity. Add later if needed.
- **Hibernation wired to the 20 GiB swapfile** — documented as optional in [3.2](03-speed-and-boot-optimizations/02-zram-swap-sysctl.md) but not a default.
- **Gaming / Steam / Proton / Lutris** — one entry in [09-apps-catalog.md](09-apps-catalog.md), no dedicated section. Ask if you want it expanded.
- **KiCad / STM32 / PlatformIO** — your request did not mention embedded. Port from [`docs/03-dev-environment/02-embedded.md`](../docs/03-dev-environment/02-embedded.md) if and when needed.
- **Marine-robotics learning roadmap** — ditto, see [`docs/03-dev-environment/05-learning-roadmap.md`](../docs/03-dev-environment/05-learning-roadmap.md).

---

[Home](README.md) · **Executive Summary** · [Next: 01 — Install →](01-install/README.md)
