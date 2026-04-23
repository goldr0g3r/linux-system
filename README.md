# The Marine-Robotics Linux Workstation

An opinionated, battle-tested Kubuntu 24.04 LTS setup and developer environment guide for an automotive-to-marine-robotics engineer. Tuned specifically for:

- **Hardware:** ASUS TUF Gaming A17 (AMD Ryzen 7 4800H, NVIDIA RTX 3050 Mobile, 16 GB, 2x 512 GB NVMe)
- **Dual-boot:** Windows 11 on a dedicated SSD, Linux on a dedicated SSD
- **Workload:** Automotive HW/SW integration, ROV/AUV robotics transition, M.Tech in AI, KiCad PCB design, STM32 embedded, Next.js blog
- **Toolchain:** Podman/Distrobox, ROS 2 Jazzy, CUDA 13, KiCad 9, STM32CubeIDE, PlatformIO, `uv`, `fnm`

> **Factual note:** Kubuntu 24.04 LTS ships **KDE Plasma 5.27 LTS**, not Plasma 6 (Plasma 6 arrived in Kubuntu 24.10). This guide compares what you can actually install today on a supported LTS. See the [Verdict](docs/01-verdict.md) for the full reasoning.

---

## How to Use This Guide

Read sequentially for a full build-from-scratch, or jump directly to the section you need. Every page has **top and bottom navigation bars** with previous / next / home links so you can move through the guide without returning here.

### Quick entry points

| You want to... | Start here |
|----------------|-----------|
| Understand why Kubuntu over Ubuntu (2024 analysis) | [Part 1 — The Verdict (v1)](docs/01-verdict.md) |
| Review the 2026 landscape verdict | [Part 1 (v2) — 2026 Re-verdict](docs/01-verdict_v2.md) |
| Compare Fedora vs EndeavourOS vs Kubuntu, evaluate Ubuntu 26.04 | [/best — 2026 Distro Research](docs/best/README.md) |
| Install Linux right now | [Part 2 — Setup Guide](docs/02-setup/README.md) |
| Set up the dev environment on an existing install | [Part 3 — Dev Environment](docs/03-dev-environment/README.md) |
| Find an app for a specific task | [Part 4 — Application Stack](docs/04-applications.md) |
| Run a scripted one-shot install | [Appendix A — Provisioning Script](docs/appendix-a-provisioning.md) |
| Fix a broken system | [Appendix B — Troubleshooting](docs/appendix-b-troubleshooting.md) |
| Just see the 10-step TL;DR | [Executive Summary](docs/00-executive-summary.md) |

---

## Full Table of Contents

### [Executive Summary](docs/00-executive-summary.md)
The verdict in one paragraph, plus a 10-step quickstart.

### [Part 1 — The Verdict: Kubuntu 24.04 vs Ubuntu 24.04](docs/01-verdict.md)
Deep technical comparison across six axes (resource management, Wayland + NVIDIA, multitasking workflow, Snap/telemetry, DX vs UX) with a definitive recommendation pinned to your profile. **Superseded by [Part 1 (v2)](docs/01-verdict_v2.md) as of 2026-04-18; retained for historical record.**

### [Part 1 (v2) — The 2026 Re-verdict](docs/01-verdict_v2.md)
Dated 2026-04-18. Consolidated conclusion after the 2026 landscape review: stay on Kubuntu 24.04 LTS today, upgrade to Kubuntu 26.04.1 LTS in ~August 2026. Includes the revised non-negotiable technical positions table.

### [/best — 2026 Distro Landscape Research](docs/best/README.md)
Two deep-research documents that feed the v2 verdict.

1. [Fedora KDE vs EndeavourOS vs Kubuntu — 2026 Deep Research](docs/best/01-distro-deep-research.md) — 8-axis comparison (release cadence, ROS 2 via Distrobox, NVIDIA, PREEMPT_RT, TUF enablement, dual-boot, package ecosystem, switching cost).
2. [Ubuntu 26.04 LTS "Resolute Raccoon" Evaluation](docs/best/02-ubuntu-26.04-evaluation.md) — what changes (kernel 7.0, Plasma 6.6, sudo-rs, Wayland-default, native CUDA/ROCm), the `.0` release risk, and the August 2026 upgrade timeline.

### [Part 2 — The Ultimate OS Setup Guide](docs/02-setup/README.md)
Scratch-to-powerhouse installation and tuning.

1. [Pre-install: Windows 11 prep](docs/02-setup/01-windows-prep.md)
2. [BIOS / UEFI configuration for ASUS TUF A17](docs/02-setup/02-bios-uefi.md)
3. [Installation & partitioning (BTRFS subvolumes, systemd-boot)](docs/02-setup/03-install-partitioning.md)
4. [First-boot debloat (snap removal, Flatpak, KDE PIM pruning)](docs/02-setup/04-debloat.md)
5. [Kernel & boot optimization (zram, sysctl, service masking)](docs/02-setup/05-kernel-boot.md)
6. [ASUS TUF A17 hardware enablement (asusctl, supergfxctl)](docs/02-setup/06-asus-hardware.md)
7. [UI/UX tweaks for engineers (KWin tiling, Activities, Konsole profiles)](docs/02-setup/07-ui-ux.md)

### [Part 3 — The Developer Environment Blueprint](docs/03-dev-environment/README.md)
Reproducible developer toolchain.

1. [Containerization: Podman (rootless) + Distrobox](docs/03-dev-environment/01-containers.md)
2. [Embedded & hardware toolchain (KiCad, STM32, PlatformIO)](docs/03-dev-environment/02-embedded.md)
3. [AI/ML: NVIDIA 580 + CUDA 13.2 + cuDNN 9.21 + PyTorch via `uv`](docs/03-dev-environment/03-ai-ml.md)
4. [Next.js blog workflow (`fnm`, `pnpm`, Vercel CLI)](docs/03-dev-environment/04-web.md)
5. [Learning roadmap for subsea robotics](docs/03-dev-environment/05-learning-roadmap.md)

### [Part 4 — Curated Application Stack](docs/04-applications.md)
100+ recommended applications across 10 categories (browsers, terminal, IDEs, robotics/EDA, AI/ML, productivity, communication, media, system, virtualization) with install channel (APT / Flatpak / upstream repo / binary / container).

### [Appendix A — One-Shot Provisioning Script](docs/appendix-a-provisioning.md)
Copy-paste bash script that chains the post-install steps for a reproducible rebuild.

### [Appendix B — Troubleshooting Matrix](docs/appendix-b-troubleshooting.md)
TUF A17 + NVIDIA hybrid-GPU pitfalls and fixes (suspend-wake black screens, MT7921 Wi-Fi drops, D3cold battery drain, SDDM login loops, Gazebo GL failures, BTRFS metadata exhaustion, dual-ESP boot-order drift, etc.).

---

## Recommended Reading Order

### If you are installing from scratch
[Exec Summary](docs/00-executive-summary.md) → [Verdict](docs/01-verdict.md) → [Setup (all 7 steps in order)](docs/02-setup/README.md) → [Dev Env (all 5 topics in order)](docs/03-dev-environment/README.md) → [App Stack](docs/04-applications.md)

Estimated time: 3-4 hours for the install phase, 2-3 hours for dev env setup. Or run the [Appendix A script](docs/appendix-a-provisioning.md) after a manual install to compress post-install into ~30 minutes of unattended time.

### If you already run Kubuntu / Ubuntu
Skip to [Dev Env](docs/03-dev-environment/README.md). Skim the debloat and kernel pages in [Part 2](docs/02-setup/README.md) for anything you may have missed.

### If you are evaluating the OS choice only
Read [Executive Summary](docs/00-executive-summary.md) + [Verdict](docs/01-verdict.md). ~15 minutes.

---

## Repository Structure

```text
c:\Code\linux-system\
├── README.md                          (this file — landing page + TOC)
└── docs/
    ├── 00-executive-summary.md        (TL;DR verdict + 10-step quickstart)
    ├── 01-verdict.md                  (Part 1 v1 — Kubuntu vs Ubuntu 24.04)
    ├── 01-verdict_v2.md               (Part 1 v2 — 2026 Re-verdict; supersedes v1)
    ├── best/                          (2026 distro landscape research)
    │   ├── README.md                  (section landing page)
    │   ├── 01-distro-deep-research.md (Fedora KDE vs EndeavourOS vs Kubuntu)
    │   └── 02-ubuntu-26.04-evaluation.md  (Resolute Raccoon upgrade analysis)
    ├── 02-setup/                      (Part 2 — Installation & tuning)
    │   ├── README.md                  (section landing page)
    │   ├── 01-windows-prep.md
    │   ├── 02-bios-uefi.md
    │   ├── 03-install-partitioning.md
    │   ├── 04-debloat.md
    │   ├── 05-kernel-boot.md
    │   ├── 06-asus-hardware.md
    │   └── 07-ui-ux.md
    ├── 03-dev-environment/            (Part 3 — Developer blueprint)
    │   ├── README.md                  (section landing page)
    │   ├── 01-containers.md
    │   ├── 02-embedded.md
    │   ├── 03-ai-ml.md
    │   ├── 04-web.md
    │   └── 05-learning-roadmap.md
    ├── 04-applications.md             (Part 4 — Curated app stack)
    ├── appendix-a-provisioning.md     (one-shot provisioning script)
    └── appendix-b-troubleshooting.md  (TUF A17 + NVIDIA fixes)
```

---

## Closing Notes

- **Keep this repo on the Linux SSD and commit changes as your workflow matures.** Everything here is a starting baseline, not dogma.
- **Snapshot before you experiment.** `sudo timeshift --create --comments "pre-<thing>"` costs 10 seconds and saves hours. See [Part 2.5](docs/02-setup/05-kernel-boot.md).
- **The marine-robotics transition is not an OS problem; it is a skills-stack problem.** This setup removes OS friction so the only variable is how much you learn per week. Items 1–5 of the [learning roadmap](docs/03-dev-environment/05-learning-roadmap.md), practiced daily in the `ros2-jazzy` + `ros2-humble` containers with a real Pixhawk / BlueRobotics test rig, is enough to be credible in six months.

---

**Next:** [Executive Summary →](docs/00-executive-summary.md)
