**instructions/** · [Archived 24.04 reference (docs/)](../README.md)

---

# Kubuntu 26.04 LTS "Resolute Raccoon" — Standalone Install & Daily-Driver Guide

A from-scratch, opinionated install and configuration guide for **Kubuntu 26.04 LTS** (released **2026-04-23**) on an **ASUS TUF Gaming A17** dual-booted with Windows 11, tuned for daily driving (browsing, video, Bluetooth headset, music, web dev) with first-class **ROS 2** and **AI/ML** support.

This tree **supersedes** the archived [`docs/` reference tree](../README.md). The `docs/` folder is frozen as the archived 24.04-era paper trail; every page below is the current canonical answer for 26.04.

---

## Target system (identical to `docs/` hardware baseline)

| Component     | Value                                                                  |
| ------------- | ---------------------------------------------------------------------- |
| Laptop        | ASUS TUF Gaming A17 (2020 / FA706IU chassis family)                    |
| CPU           | AMD Ryzen 7 4800H (8c / 16t Zen 2 Renoir)                              |
| iGPU          | AMD Radeon RX Vega 7 (Renoir)                                          |
| dGPU          | NVIDIA GeForce RTX 3050 Laptop GPU, 4 GB VRAM (GA107 Ampere)           |
| RAM           | 16 GB DDR4-3200 (2x8)                                                  |
| Storage       | 2x NVMe SSD, 512 GB each — Disk A = Windows 11, Disk B = Linux target  |
| Boot strategy | Separate ESPs per disk; ASUS F8 one-time menu picks OS (no shared GRUB)|
| Display       | 17.3" 1080p 144 Hz                                                     |
| Wi-Fi         | MediaTek MT7921 (expected in 2020-era TUF) — kernel 7.0 native         |

If your hardware differs, the install-time chapters ([01-install/](01-install/README.md)) still apply but the TUF-specific pages ([02.03](02-post-install-foundations/03-asus-tuf-hybrid-gpu.md), [03.04](03-speed-and-boot-optimizations/04-filesystem-thermals-power.md)) will need adaptation.

---

## Eyes-open risk acceptance (why we are installing `.0` today)

The [v2 verdict in `docs/`](../docs/01-verdict_v2.md) explicitly recommended **waiting until Kubuntu 26.04.1 (~Aug 2026)** before upgrading. You are overriding that on day-of-release. The four specific risks that motivated the "wait" advice, and the mitigation we bake in:

| Risk                                                         | Likelihood | Mitigation in this guide                                                                                               |
| ------------------------------------------------------------ | ---------- | ---------------------------------------------------------------------------------------------------------------------- |
| Wayland + NVIDIA 580+ Explicit Sync regressions              | Medium     | X11 fallback documented in [10-troubleshooting.md](10-troubleshooting.md); Plasma 5.27-on-X11 not available on 26.04 so the escape hatch is `plasma-workspace-x11` manual install |
| `sudo-rs` edge-case incompatibility with existing sudoers.d  | Low        | Pre-flight `sudo --validate` check after every sudoers edit; Timeshift snapshot before first `sudoers.d` drop-in       |
| Python 3.14 wheel lag (PyTorch, scipy, opencv-python)        | High       | AI/ML venvs pin `uv venv --python 3.12`; host 3.14 is only the default, never the only option                          |
| Linux 7.0 novelty (scheduler, XFS self-heal, RT via preempt) | Medium     | Timeshift snapshots **before** any kernel parameter change; `preempt=full` treated as opt-in in [3.1](03-speed-and-boot-optimizations/01-kernel-boot-parameters.md), not default |

**Baseline discipline: `sudo timeshift --create` before every non-trivial change.** This is re-stated in [02.05](02-post-install-foundations/05-snapshot-policy-btrfs-timeshift.md) and hard-wired into the provisioning script in [99](99-provisioning-script.md).

---

## How to read this guide

Every page has **top and bottom navigation bars** with Home / ↑ Section / ← Previous / Next → links.

### Quick entry points

| You want to…                                           | Start here                                                              |
| ------------------------------------------------------ | ----------------------------------------------------------------------- |
| See the one-paragraph verdict + 15-step quickstart     | [00-executive-summary.md](00-executive-summary.md)                      |
| Install Kubuntu 26.04 right now                        | [01-install/](01-install/README.md)                                     |
| Tune a fresh install (drivers, Wayland, debloat)       | [02-post-install-foundations/](02-post-install-foundations/README.md)   |
| Improve boot time / responsiveness / battery           | [03-speed-and-boot-optimizations/](03-speed-and-boot-optimizations/README.md) |
| Daily driving — browse, video, BT headset, music       | [04-daily-driver-stack/](04-daily-driver-stack/README.md)               |
| Set up web development                                 | [05-web-development/](05-web-development/README.md)                     |
| Install ROS 2 (and which version / approach to pick)   | [06-ros2-robotics/](06-ros2-robotics/README.md)                         |
| Set up AI/ML (CUDA, PyTorch, local LLMs)               | [07-ai-ml-workstation/](07-ai-ml-workstation/README.md)                 |
| Productivity, backups, security                        | [08-productivity-security/](08-productivity-security/README.md)         |
| Find an app for a specific task                        | [09-apps-catalog.md](09-apps-catalog.md)                                |
| Fix a broken system                                    | [10-troubleshooting.md](10-troubleshooting.md)                          |
| Replay the whole post-install unattended               | [99-provisioning-script.md](99-provisioning-script.md)                  |

---

## Full table of contents

### [00 — Executive Summary](00-executive-summary.md)
TL;DR verdict, the 15-step quickstart, and the non-negotiable technical positions table.

### [01 — Prerequisites & Install](01-install/README.md)
1. [Hardware check and USB preparation](01-install/01-hardware-check-and-usb-prep.md)
2. [Windows 11 prep + BIOS/UEFI configuration](01-install/02-windows-prep-and-bios.md)
3. [Installer flavor: Minimal vs Normal vs Extended — which and why](01-install/03-installer-flavor-minimal-vs-full.md)
4. [Installation & partitioning (BTRFS subvolumes, systemd-boot)](01-install/04-installation-and-partitioning.md)
5. [First boot, updates, and initial Timeshift snapshot](01-install/05-first-boot-and-snapshot.md)

### [02 — Post-Install Foundations](02-post-install-foundations/README.md)
1. [Debloat: snap removal, Flatpak + Flathub, Firefox via Mozilla APT, KDE PIM prune](02-post-install-foundations/01-debloat-snap-flatpak-pim.md)
2. [NVIDIA driver 580 + native `nvidia-cuda-toolkit` (new in 26.04)](02-post-install-foundations/02-nvidia-driver-580-and-cuda.md)
3. [ASUS TUF hybrid-GPU enablement (`asusctl`, `supergfxctl`, fan curves, battery cap)](02-post-install-foundations/03-asus-tuf-hybrid-gpu.md)
4. [Wayland + Plasma 6.6 + NVIDIA tuning (Explicit Sync, XWayland, fractional scaling)](02-post-install-foundations/04-wayland-plasma-66-nvidia.md)
5. [Snapshot policy: BTRFS + Timeshift rollback discipline](02-post-install-foundations/05-snapshot-policy-btrfs-timeshift.md)

### [03 — Speed, Boot Time, and Responsiveness](03-speed-and-boot-optimizations/README.md)
1. [Kernel boot parameters (`preempt=full`, NVIDIA DRM KMS, NVMe power states)](03-speed-and-boot-optimizations/01-kernel-boot-parameters.md)
2. [zram, swappiness, and sysctl tuning](03-speed-and-boot-optimizations/02-zram-swap-sysctl.md)
3. [Service masking + journald capping](03-speed-and-boot-optimizations/03-service-masking-and-journald.md)
4. [Filesystem mount options, `fstrim`, thermals, power profiles](03-speed-and-boot-optimizations/04-filesystem-thermals-power.md)

### [04 — Daily-Driver Stack](04-daily-driver-stack/README.md)
1. [Browsers (Firefox via Mozilla APT, Brave, Zen, Vivaldi — no snap)](04-daily-driver-stack/01-browsers.md)
2. [Video + audio + PipeWire architecture + VA-API hardware decode](04-daily-driver-stack/02-video-audio-pipewire.md)
3. [Bluetooth headset (LDAC / AptX HD, mic switching, auto-reconnect)](04-daily-driver-stack/03-bluetooth-headset-hifi.md)
4. [Music: Spotify, local libraries, terminal players, podcast ripping](04-daily-driver-stack/04-music-streaming-local.md)
5. [Communication + productivity apps](04-daily-driver-stack/05-comms-and-productivity.md)
6. [Plasma 6.6 UX: tiling, Activities, Konsole / Yakuake, fonts](04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md)

### [05 — Web Development](05-web-development/README.md)
1. [Shell and terminal (Zsh + Starship, tmux/zellij, fzf/rg/bat/eza)](05-web-development/01-shell-and-terminal.md)
2. [Git, SSH, GPG (Ed25519 + ML-KEM hybrid, commit signing)](05-web-development/02-git-ssh-gpg.md)
3. [Node.js runtimes and package managers (`fnm`, Corepack, Bun, Deno)](05-web-development/03-runtimes-and-package-managers.md)
4. [Editors and IDEs (VS Code, Cursor, Zed, JetBrains, Helix)](05-web-development/04-editors-and-ides.md)
5. [Frameworks and local databases (Next.js, Astro, Postgres/Redis via Podman)](05-web-development/05-frameworks-and-databases.md)

### [06 — ROS 2 for Robotics](06-ros2-robotics/README.md)
1. [ROS 2 landscape 2026 (Jazzy, Kilted, Lyrical — Ubuntu version matrix)](06-ros2-robotics/01-ros2-landscape-2026.md)
2. [Install approaches compared (native apt, Distrobox, Podman Compose, rocker, VM)](06-ros2-robotics/02-install-approaches-compared.md)
3. [Bridge phase — Distrobox + Jazzy on Ubuntu 24.04 base (use this today)](06-ros2-robotics/03-distrobox-bridge-jazzy-now.md)
4. [Native phase — Lyrical Luth apt install on 26.04 host (post-May 22, 2026)](06-ros2-robotics/04-native-lyrical-after-may-22.md)

### [07 — AI / ML Workstation](07-ai-ml-workstation/README.md)
1. [CUDA 13 + cuDNN native install on 26.04](07-ai-ml-workstation/01-cuda-13-cudnn-native.md)
2. [Python via `uv`, pinned 3.12 venv, PyTorch + JAX](07-ai-ml-workstation/02-python-uv-pytorch-jax.md)
3. [Local LLMs: Ollama + Open WebUI Quadlet + Continue.dev, 4 GB VRAM model picks](07-ai-ml-workstation/03-local-llms-ollama-openwebui.md)

### [08 — Productivity and Security](08-productivity-security/README.md)
1. [Keyboard automation (KRunner, KeyD, AutoKey) + Activities](08-productivity-security/01-keyboard-automation.md)
2. [3-2-1 backup strategy (Restic + rclone + offsite)](08-productivity-security/02-backup-strategy-3-2-1.md)
3. [Firewall, sandboxing, YubiKey, ClamAV](08-productivity-security/03-firewall-sandboxing.md)

### Top-level support pages

- [09 — Curated apps catalog (~100 apps, 10 categories)](09-apps-catalog.md)
- [10 — Troubleshooting matrix (TUF A17 + Wayland + NVIDIA + sudo-rs pitfalls)](10-troubleshooting.md)
- [99 — One-shot provisioning script](99-provisioning-script.md)

---

## Recommended reading order

### If you are installing from scratch

[Executive Summary](00-executive-summary.md) → [01-install](01-install/README.md) (all 5 steps in order) → [02-post-install-foundations](02-post-install-foundations/README.md) (all 5) → [03-speed-and-boot-optimizations](03-speed-and-boot-optimizations/README.md) (all 4) → then whichever of [04-daily-driver](04-daily-driver-stack/README.md), [05-web-dev](05-web-development/README.md), [06-ros2](06-ros2-robotics/README.md), [07-ai-ml](07-ai-ml-workstation/README.md) apply to your work first.

Estimated time: **3 hours** for install + core tuning (parts 01–03), **2 hours** for daily-driver + web-dev (parts 04–05), **1 hour** each for ROS 2 and AI/ML.  Or run [99-provisioning-script.md](99-provisioning-script.md) after a manual install to compress post-install into ~30 unattended minutes.

### If you already run Linux on this box

Skip to [02-post-install-foundations](02-post-install-foundations/README.md) and [03-speed-and-boot-optimizations](03-speed-and-boot-optimizations/README.md). Skim [01-install/03](01-install/03-installer-flavor-minimal-vs-full.md) in case you want to reinstall from scratch later.

### If you only care about the apps list

[09-apps-catalog.md](09-apps-catalog.md). Every entry has an install channel (APT / Flatpak / .deb / container) and a one-line rationale.

---

## Repository structure

```text
c:\Code\linux-system\
├── README.md                                      (existing — pre-2026 landing page for docs/)
├── docs/                                          (ARCHIVED 24.04-era reference, frozen)
└── instructions/                                  (THIS TREE — canonical for 26.04 LTS)
    ├── README.md                                  (this file)
    ├── 00-executive-summary.md
    ├── 01-install/
    ├── 02-post-install-foundations/
    ├── 03-speed-and-boot-optimizations/
    ├── 04-daily-driver-stack/
    ├── 05-web-development/
    ├── 06-ros2-robotics/
    ├── 07-ai-ml-workstation/
    ├── 08-productivity-security/
    ├── 09-apps-catalog.md
    ├── 10-troubleshooting.md
    └── 99-provisioning-script.md
```

---

## Closing notes

- **This is a baseline, not dogma.** Every decision has a reasoning paragraph next to it; if your workload differs, the reasoning is what you adapt, not the commands.
- **Snapshot before you experiment.** [`sudo timeshift --create --comments "pre-<thing>"`](02-post-install-foundations/05-snapshot-policy-btrfs-timeshift.md) costs 10 seconds and saves hours.
- **Commit the instructions/ directory to git after each meaningful local customisation** so your future self can diff against the shipped baseline.

---

**Next:** [00 — Executive Summary →](00-executive-summary.md)
