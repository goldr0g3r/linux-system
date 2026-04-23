[Home](../../README.md) · [↑ /best](README.md) · [← Previous: 1. Deep Research](01-distro-deep-research.md) · **2. Ubuntu 26.04 Evaluation** · [Next: Part 1 (v2) Re-verdict →](../01-verdict_v2.md)

---

# 2. Kubuntu 26.04 LTS "Resolute Raccoon" — Upgrade Evaluation

Written 2026-04-18, five days before the 26.04 release on **April 23, 2026**. Decides whether you should upgrade from Kubuntu 24.04 LTS, and if yes, when.

**Spoiler: stay on 24.04 now, upgrade to 26.04.1 around August 2026.** Reasoning below.

**Navigate within this page:**

- [What Kubuntu 26.04 changes](#what-kubuntu-2604-changes)
- [Impact on your current workflow](#impact-on-your-current-workflow)
- [The .0 LTS release risk](#the-0-lts-release-risk)
- [Decision matrix — upgrade now vs wait](#decision-matrix--upgrade-now-vs-wait)
- [Recommendation & timeline](#recommendation--timeline)
- [Upgrade procedure sketch](#upgrade-procedure-sketch)

---

## What Kubuntu 26.04 changes

Sourced from the Ubuntu 26.04 [release notes summary](https://documentation.ubuntu.com/release-notes/26.04/summary-for-lts-users/) and the [Kubuntu 26.04 Beta announcement](https://www.kubuntu.org/news/kubuntu-26-04-beta/):

| Layer | Kubuntu 24.04 (today) | Kubuntu 26.04 (Apr 23) | Delta |
|-------|------------------------|------------------------|-------|
| Kernel | Linux 6.8 (HWE 6.17) | **Linux 7.0** | First major version bump in years; RT merged into mainline at 6.12 so `preempt=full` is now a boot parameter |
| Desktop | KDE Plasma 5.27 LTS | **KDE Plasma 6.6** | Qt6 throughout, per-desktop tiling, HDR calibration wizard, new Spectacle screenshot tool |
| Default session | X11 | **Wayland** (X11 session not installed by default, unsupported) | Invalidates the v1 "X11 on NVIDIA" recommendation |
| Qt | 5.15 + 6.4 | 6.10.2 | Qt5 apps still run; Qt6 is the primary target |
| Frameworks | KF5 | KF 6.24.0 | |
| sudo | C sudo 1.9 | **sudo-rs** (Rust rewrite, memory-safe) | First LTS to ship a non-C sudo by default |
| SSH / TLS | ECDHE/Ed25519 | **Post-quantum (ML-KEM) enabled by default** | Transparent for most workflows; may break strict FIPS-only stacks |
| Mesa | 24.x → 25.2 HWE | **Mesa 26.0** | Better RDNA support; Vulkan 1.4 stable |
| systemd | 255 | 259 | `systemd-run0` replaces sudo for isolated user commands |
| GCC | 13.2 | **15.x** | Default C++26 mode; most older code builds unchanged |
| Python | 3.12 | **3.14** | Some wheels lag; test `uv pip install torch` before migrating |
| OpenJDK | 21 | 25 | Long-term branch |
| Go | 1.22 | 1.25 | |
| Firefox | 124 (APT, via Mozilla team) | 148 | |
| LibreOffice | 24.2 | 26.2.2 | |
| CUDA toolkit | NVIDIA's APT repo required | **Native `nvidia-cuda-toolkit` in Ubuntu archive** | One APT line instead of repo dance in [3.3](../03-dev-environment/03-ai-ml.md) |
| ROCm | Third-party | **Native AMD ROCm in archive** | Irrelevant for your RTX 3050 but notable |
| Desktop RAM floor | 4 GiB | **6 GiB** (recommended; will boot on less) | Still fine on your 16 GiB |
| Support window | May 2029 / 2034 Pro / 2036 Legacy | **April 2031 / 2036 Pro / 2038 Legacy** | +2 years on every tier |

Plus a raft of GNOME-side changes (GNOME 50 Wayland-exclusive, Ptyxis replaces GNOME Terminal, Papers replaces Evince, Loupe replaces Eye of GNOME, Showtime replaces Totem) that do **not** affect Kubuntu — those are the Ubuntu (GNOME) flavour only.

---

## Impact on your current workflow

Mapped to your workload from [00-executive-summary.md](../00-executive-summary.md):

### What upgrades cleanly

- **Podman + Distrobox** ([3.1](../03-dev-environment/01-containers.md)): unchanged. Your `ros2-jazzy`, `ros2-humble`, `embedded-fedora` containers keep their Ubuntu 22.04 / 24.04 / Fedora 40 bases regardless of host version.
- **`uv` + Python 3.14 on the host**: `uv` itself is version-agnostic. PyTorch nightly builds for 3.14 exist as of 2026-03; stable wheels lag ~2–4 weeks after release.
- **`fnm` + Node / Next.js** ([3.4](../03-dev-environment/04-web.md)): unchanged. `fnm` is a single binary with no system dependencies.
- **KiCad 9, STM32 toolchain, OpenOCD, PlatformIO** ([3.2](../03-dev-environment/02-embedded.md)): all `.deb` installs; no version conflict with 26.04 libc or Python.
- **Timeshift/Snapper snapshots** ([2.5](../02-setup/05-kernel-boot.md)): format unchanged; your pre-upgrade snapshot is a valid rollback target.
- **BTRFS subvolumes, systemd-boot, 20 GiB swapfile** ([2.3](../02-setup/03-install-partitioning.md)): no migration needed.

### What changes or may break

| Area | Change | Mitigation |
|------|--------|------------|
| **X11 session on Kubuntu** | Not installed by default; unsupported. You would have installed `plasma-workspace-x11` manually. | Move to Wayland. Plasma 6.6 + NVIDIA 580 with explicit sync and `fifo-v1` is materially better than Plasma 5.27 on Wayland, but still Wayland. See [v1 Wayland section](../01-verdict.md#wayland--nvidia-rtx-3050-mobile-compatibility) for the failure modes to watch for. |
| **NVIDIA on Wayland** | Driver 580+ supports explicit sync; XWayland apps (Chromium, Firefox, older Qt apps) flicker less than in the 5.27 era | Expect one weekend of reconfiguring screen capture (OBS needs PipeWire portal), screen sharing (works out-of-box on Plasma 6.6), and VS Code integration |
| **ROS 2 GUI (RViz2, Gazebo)** | Runs inside the `ros2-jazzy` Distrobox on Ubuntu 24.04 base | Host session change does not affect container; RViz2 on XWayland works |
| **Python 3.14 on host** | Some PyTorch wheels, opencv-python, scipy may lag | Use `uv venv --python 3.12` explicitly for projects with pinned dependencies; host Python 3.14 is only the default, not the only option |
| **GCC 15 default C++26 mode** | Third-party source builds (Yocto, PX4 SITL) may warn about deprecations | Set `CXXFLAGS=-std=c++20` or `-std=c++17` in project Makefiles; not breaking |
| **sudo-rs** | Different binary; `/etc/sudoers` syntax is compatible; some edge-case plugins do not load | Zero impact unless you use `sudoers.d` plugins beyond `@include` |
| **Snap situation** | Same as 24.04 — snapd still preinstalled, Firefox still APT (Mozilla key) on Kubuntu, same default KDE PIM preinstall | Your [2.4 Debloat](../02-setup/04-debloat.md) script still applies verbatim |
| **Post-quantum SSH/TLS** | ML-KEM enabled by default; hybrid with classical | Zero impact on GitHub, AWS, Vercel, or any mainstream server |
| **Ubuntu Pro realtime add-on** | Still works on 26.04; also — `preempt=full` on stock kernel 7.0 gives you soft RT without Pro | Both paths coexist |
| **CUDA install path** ([3.3](../03-dev-environment/03-ai-ml.md)) | `nvidia-cuda-toolkit` is native in 26.04 archive | Simplifies your install — the NVIDIA APT repo step becomes optional |
| **`asusctl` / `supergfxctl`** ([2.6](../02-setup/06-asus-hardware.md)) | `asus-linux.org` PPA supports 26.04 from day 0 (Luke Jones' track record) | No action |
| **STM32CubeIDE, J-Link, ST-Link** | Vendor `.deb` packages are libc-portable; install on 26.04 unchanged | No action |

### What gets genuinely better

- **Plasma 6.6** brings KDE's per-virtual-desktop tiling configuration — meaning your "Automotive" Activity and "ROV" Activity ([v1 UI workflow](../01-verdict.md#ui-workflow-for-heavy-multitasking)) can have different default tile layouts. Qualitatively a step up from `krohnkite` / `bismuth` on 5.27.
- **Native CUDA packaging** shrinks [3.3](../03-dev-environment/03-ai-ml.md) by 5–10 minutes per reinstall.
- **Linux 7.0 with `preempt=full`** covers 80% of the "soft real-time" use case without needing Ubuntu Pro realtime. Still get Pro for hard RT (< 100 µs guarantees).
- **10-year support window** ([Ubuntu Pro free for 5 personal machines](https://ubuntu.com/pro)) pushes your end-of-life from 2034 to 2036. Relevant because this laptop may last that long.
- **sudo-rs, post-quantum crypto, memory-safe by-default** — aligns with the 2026 state-of-the-art. Resume-item-grade if you care.

---

## The .0 LTS release risk

Historical pattern across three Ubuntu LTS releases:

| Release | `.0` date | `.1` date | Notable `.0` issues |
|---------|-----------|-----------|----------------------|
| 20.04 LTS | Apr 23, 2020 | Aug 6, 2020 | ZFS root corruption, fwupd breakage |
| 22.04 LTS | Apr 21, 2022 | Aug 11, 2022 | GDM Wayland regressions, NVIDIA driver stall on resume |
| 24.04 LTS | Apr 25, 2024 | Aug 29, 2024 | Installer bug on dual-boot, `needrestart` interactive prompts, `snapd` refresh loops |

The consistent pattern: **~4 months of bug-fix work land between `.0` and `.1`**. Canonical's own release quality statement acknowledges that conservative users should wait for `.1`. Given 26.04 ships:

- Linux 7.0 (first major bump; RT path, scheduler, drivers all novel)
- sudo-rs (first time shipping in any LTS)
- Plasma 6.6 on Wayland as default with NVIDIA (new combination for Kubuntu)
- GCC 15 and Python 3.14 (toolchain churn)
- Native CUDA/ROCm packages (interaction with NVIDIA's own repo is untested at scale)

This is a **higher-risk `.0`** than 22.04 or 24.04 was. The probability that at least one of the above bites you in the first 4 months is high. Your laptop is the tool you use to earn grades, income, and your marine-robotics portfolio. Do not use it as a beta-testing platform.

---

## Decision matrix — upgrade now vs wait

| Option | Pros | Cons | Recommended? |
|--------|------|------|--------------|
| **Upgrade day 1 (Apr 23, 2026)** | Earliest access to Plasma 6.6, kernel 7.0, CUDA in archive | `.0` bug exposure on a workhorse laptop; Wayland-on-NVIDIA still maturing; 3 days from today | **No** |
| **Upgrade on `26.04.1` (~Aug 2026)** | Bug-fix rollup landed; kernel 7.0 point releases stable; Wayland-on-NVIDIA has 4 months of production testing | 4-month wait | **Yes** |
| **Upgrade on `26.04.2` (~Feb 2027)** | Maximum stability; HWE stack freshly baked | 10-month wait; no material advantage over `.1` for your workload | No |
| **Never upgrade — stay on 24.04 LTS until 2029 standard / 2034 Pro / 2036 Legacy** | Zero risk, zero work | Miss Plasma 6.6 DX, miss 10-year support on 26.04, eventually forced by package upstreams | Viable fallback |
| **Jump to Fedora 43 KDE / EndeavourOS** | Bleeding edge tech | Everything in [Axis 8 of Part 1](01-distro-deep-research.md#axis-8--cost-of-switching-from-kubuntu-2404) | No for the primary host |

---

## Recommendation & timeline

**Install or stay on [Kubuntu 24.04 LTS](../02-setup/README.md) today.**

**Plan to upgrade to Kubuntu 26.04.1 LTS around August 2026**, immediately after Canonical releases the first bug-fix ISO.

### Concrete timeline

| Date | Action |
|------|--------|
| **2026-04-18 (today)** | No change. Continue using Kubuntu 24.04 LTS per [Part 2](../02-setup/README.md) and [Part 3](../03-dev-environment/README.md). |
| **2026-04-23** | Kubuntu 26.04.0 ships. **Do not upgrade.** Optionally: install 26.04 in a [VirtualBox VM](../04-applications.md) or a spare partition to evaluate Plasma 6.6 + Wayland + NVIDIA on your exact hardware. |
| **2026-05 → 2026-07** | Monitor: `/r/kubuntu`, [KDE Plasma bug tracker](https://bugs.kde.org/) for 6.6 regressions, and the Kubuntu devel list for sudo-rs / Plasma-Wayland-NVIDIA incident reports. |
| **2026-08 (approx)** | Kubuntu 26.04.1 LTS ships. Before upgrading: make a Timeshift snapshot per [2.5](../02-setup/05-kernel-boot.md), verify your `ros2-jazzy` container has been exercised in the last 24 hours so its state is known-good. |
| **2026-08 (upgrade day)** | Run `sudo do-release-upgrade` (Kubuntu will start offering this once `.1` is out). Allow 60–90 minutes uninterrupted. On first reboot, log in to the Plasma (Wayland) session — X11 is not installed. |
| **2026-08 first week** | Validate: CUDA works (`nvidia-smi`, `torch.cuda.is_available()`), RViz2 renders, STM32CubeIDE programs a blue-pill, `asusctl` reads the CPU temperature. Re-run the relevant parts of [Appendix A provisioning script](../appendix-a-provisioning.md) if anything regressed. |

### Trigger conditions to deviate

Re-evaluate this plan if:

- A specific upstream you depend on (CUDA 13.2, ROS 2 Jazzy, `asusctl`, KiCad 9) **drops 24.04 support before Aug 2026**. Probability: very low for any of those; 24.04 is LTS.
- You do a hardware refresh (new laptop) before Aug 2026 — install 26.04.1 fresh on the new box instead of upgrading the existing machine.
- A `.0` critical bug is reported that Canonical does not ship a `.1` patch for by Aug 2026 — unlikely, but check before upgrading.

---

## Upgrade procedure sketch

Not a full [Part 2](../02-setup/README.md) rewrite — a checklist for the Aug 2026 upgrade day. Full details when the time comes.

```bash
sudo timeshift --create --comments "pre-26.04.1 upgrade $(date -I)"
distrobox stop --all
systemctl --user stop podman.service

sudo apt update && sudo apt full-upgrade -y
sudo apt install -y update-manager-core
sudo do-release-upgrade

sudo apt full-upgrade -y
sudo apt autoremove -y --purge

nvidia-smi
uname -r
echo "expect 7.0.x-generic"

kinfocenter
echo "expect Plasma 6.6.x · Qt 6.10.2 · KF 6.24.0"

distrobox enter ros2-jazzy
ros2 topic list
```

If anything fails, `sudo timeshift --restore` returns you to the pre-upgrade state in ~5 minutes. This is why [2.5](../02-setup/05-kernel-boot.md) mandates a snapshot before non-trivial operations.

### Files to revisit after the upgrade

Once on 26.04.1, these repo pages need minor edits (out of scope for /best):

- [00-executive-summary.md](../00-executive-summary.md) — bump "Kubuntu 24.04 LTS" references
- [01-verdict.md](../01-verdict.md) — mark explicitly superseded by [01-verdict_v2.md](../01-verdict_v2.md)
- [2.3 Partitioning](../02-setup/03-install-partitioning.md) — minor ISO filename update
- [3.3 AI/ML](../03-dev-environment/03-ai-ml.md) — simplify CUDA install (native package, no NVIDIA repo)
- [Appendix A provisioning script](../appendix-a-provisioning.md) — retarget to 26.04.1 codename

---

Continue to [Part 1 (v2) — 2026 Re-verdict](../01-verdict_v2.md) for the consolidated conclusion that cites both this document and the [Deep Research](01-distro-deep-research.md).

---

[Home](../../README.md) · [↑ /best](README.md) · [← Previous: 1. Deep Research](01-distro-deep-research.md) · **2. Ubuntu 26.04 Evaluation** · [Next: Part 1 (v2) Re-verdict →](../01-verdict_v2.md)
