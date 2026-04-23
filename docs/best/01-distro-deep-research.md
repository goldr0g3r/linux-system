[Home](../../README.md) · [↑ /best](README.md) · [← Previous: /best Intro](README.md) · **1. Deep Research** · [Next: 2. Ubuntu 26.04 Eval →](02-ubuntu-26.04-evaluation.md)

---

# 1. Fedora KDE vs EndeavourOS KDE vs Kubuntu — 2026 Deep Research

Written 2026-04-18. Decides which Linux distribution is the right host for an ASUS TUF Gaming A17 (Ryzen 7 4800H, RTX 3050 Mobile 4 GB, 16 GiB DDR4, 2x 512 GB NVMe) doing automotive HW/SW integration, marine robotics (ROV/AUV) transition, M.Tech (AI) coursework, KiCad + STM32 embedded, and a Next.js blog on Vercel.

All three distributions are compared in their **KDE Plasma variant** (Kubuntu, Fedora KDE Spin, EndeavourOS with the KDE option) to keep the comparison fair against the DX reasoning in the [original Verdict](../01-verdict.md). The GNOME flagships are not the subject of this comparison.

**Navigate within this page:**

- [Why re-open the question](#why-re-open-the-question-in-2026)
- [Parity baseline](#parity-baseline-what-is-identical-across-all-three)
- [Axis 1 — Release model & LTS horizon](#axis-1--release-model--lts-horizon)
- [Axis 2 — ROS 2 via Distrobox](#axis-2--ros-2-via-distrobox)
- [Axis 3 — NVIDIA RTX 3050 Mobile](#axis-3--nvidia-rtx-3050-mobile-ampere)
- [Axis 4 — PREEMPT_RT access](#axis-4--preempt_rt-kernel-access-subsea-roadmap-item-2)
- [Axis 5 — ASUS TUF A17 hardware enablement](#axis-5--asus-tuf-a17-hardware-enablement)
- [Axis 6 — Dual-boot + systemd-boot + BTRFS](#axis-6--dual-boot--systemd-boot--btrfs-your-existing-layout)
- [Axis 7 — Package ecosystem depth](#axis-7--package-ecosystem-depth)
- [Axis 8 — Cost of switching](#axis-8--cost-of-switching-from-kubuntu-2404)
- [Verdict](#verdict)

---

## Why re-open the question in 2026

Four things changed since the [original Verdict (v1)](../01-verdict.md) was written:

1. **Distrobox absorbs the ROS 2 host-distro requirement.** The `ros2-jazzy` / `ros2-humble` pattern in [3.1](../03-dev-environment/01-containers.md) means any distro with Podman + Distrobox can run ROS 2 at parity. Ubuntu-as-host is no longer the forcing function.
2. **Kubuntu 26.04 LTS ships on April 23, 2026** (Linux 7.0, Plasma 6.6, Wayland-default, sudo-rs, native CUDA/ROCm). Detailed separately in [Part 2 of /best](02-ubuntu-26.04-evaluation.md).
3. **EndeavourOS 2026.01.12 "Ganymede Neo"** added auto NVIDIA detection and switched to `nvidia-open` modules. Your RTX 3050 Mobile is Ampere (GA107), which is supported by `nvidia-open`.
4. **Fedora 43 KDE Spin** (October 28, 2025) is the most polished Fedora KDE release ever, shipping Plasma 6.4.5 on Linux 6.17 with the most mature Podman/Distrobox stack of any mainstream distro (these tools are Red Hat's).

If any one of those four had not happened, v1 would still be the final word. All four did. Hence this document.

---

## Parity baseline (what is identical across all three)

Before diving into differences, note what is the **same regardless of choice**:

| Axis | Value on all three | Notes |
|------|--------------------|-------|
| Desktop | KDE Plasma 6.x | Kubuntu 26.04: 6.6 · Fedora 43 KDE: 6.4.5 · EndeavourOS: 6.5.4 |
| Qt | 6.9–6.10 | Kubuntu 26.04 ships 6.10.2; others lag by a point release |
| Container runtime | Rootless Podman + Distrobox | Red Hat wrote both; everyone packages them |
| ROS 2 Jazzy | In a Distrobox with Ubuntu 24.04 base | Host distro does not matter — see [Axis 2](#axis-2--ros-2-via-distrobox) |
| Flatpak + Flathub | Available on all three | KiCad 9, QGroundControl, Zotero all installable the same way |
| Wayland compositor | KWin 6.x on all three | Explicit sync + NVIDIA 580 → similar behaviour everywhere |
| Kernel floor | ≥ 6.17 | Fedora 43: 6.17 · EndeavourOS: 6.18.4 LTS · Kubuntu 26.04: 7.0 |

**Conclusion: the core daily-driver experience is comparable.** The differences are in release model, vendor-driver paths, package ecosystem, and switching cost. That is where the decision is made.

---

## Axis 1 — Release model & LTS horizon

| Model | Kubuntu 24.04 LTS | Kubuntu 26.04 LTS (ships Apr 23) | Fedora 43 KDE | EndeavourOS (Ganymede Neo) |
|-------|-------------------|-----------------------------------|---------------|----------------------------|
| Cadence | 2-year LTS | 2-year LTS | 6-month release; Fedora 44 ~May 2026 | Rolling (Arch) |
| Standard support | May 2029 | April 2031 | ~13 months per release | Indefinite (pacman updates) |
| Extended support | +5 yrs Ubuntu Pro (2034), +2 Legacy (2036) | +5 yrs Pro (2036), +2 Legacy (2038) | None | None |
| Force-upgrade event | Every 2 yrs (`do-release-upgrade`) | Every 2 yrs | Every ~13 months | Never (continuous `pacman -Syu`) |
| Breakage risk profile | Rare; batched at release | Rare; batched at release | Higher — new toolchains every 6 mo | Per-update, but small |

For a 3–4 year M.Tech + job-transition horizon, the ranking is:

- **Best:** Kubuntu LTS — one reinstall at most in the next 4 years; 10-year Pro window if you pay.
- **Middle:** EndeavourOS — no reinstall ever; small ongoing upgrade risk instead.
- **Worst:** Fedora — 3–4 force-upgrades in the same window, each with a non-zero chance of breaking a pinned toolchain (STM32CubeIDE, CUDA 13.2, etc.).

**Winner: Kubuntu (24.04 or 26.04 depending on your clock — see [Part 2 of /best](02-ubuntu-26.04-evaluation.md)).**

---

## Axis 2 — ROS 2 via Distrobox

The container pattern from [3.1](../03-dev-environment/01-containers.md):

```bash
distrobox create --name ros2-jazzy \
  --image docker.io/library/ubuntu:24.04 \
  --home ~/Containers/ros2-jazzy \
  --additional-flags "--device /dev/dri --device /dev/bus/usb --network=host"
```

Inside that container you install `ros-jazzy-desktop` from the Open Robotics APT repo, export `rviz2` / `ros2` back to the host, and the host distro never sees a ROS package. This works **identically** on any host with rootless Podman and `--network=host` support for DDS multicast.

Empirical parity:

| Capability | Kubuntu host | Fedora host | EndeavourOS host |
|------------|--------------|-------------|------------------|
| `podman run` rootless | Works | Works (Fedora's own stack) | Works |
| `distrobox enter ros2-jazzy` | Works | Works | Works |
| Cyclone DDS multicast across containers + host | Works with `--network=host` | Works with `--network=host` | Works with `--network=host` |
| `rviz2` OpenGL through `/dev/dri` | Works (requires PRIME offload via `prime-run`) | Works (same, via `supergfxctl --mode hybrid`) | Works (same) |
| `/dev/ttyUSB0` passthrough for CAN adapters | Works | Works (minor SELinux tuning if enforcing) | Works |
| `--device /dev/bus/usb` for STM32 flashing | Works | Works | Works |

Fedora's one edge: the `fedora-toolbox` image is a first-class Distrobox target; the Bluefin/Atomic variants have a known `--nvidia` flag bug (source: upstream issue #2559) but it does not affect the non-Atomic `Fedora 43 KDE` we are comparing here.

**Winner: three-way tie.** This axis, which was the single strongest argument for Ubuntu in v1, is now neutral.

---

## Axis 3 — NVIDIA RTX 3050 Mobile (Ampere)

Your GPU is an Ampere-generation GA107. All three paths deliver the proprietary 580-series driver; they differ in operator burden.

| Path | Command sequence | Secure Boot | Kernel-bump behaviour |
|------|------------------|-------------|------------------------|
| **Kubuntu 24.04 / 26.04** | `sudo ubuntu-drivers install` → reboot → done | Works signed out-of-the-box (Canonical-signed shim) | DKMS rebuilds automatically on kernel update |
| **Fedora 43 KDE** | Enable RPMFusion → `sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda` → enroll MOK → reboot 2x | **You must enroll a MOK** at first boot after install (blue enrollment screen) — fails silently if missed | akmod rebuild is automatic but requires ~60 s kernel-post-install window |
| **EndeavourOS (Ganymede Neo)** | Auto-detected during install; `nvidia-open` DKMS module installed and signed | Works if you used EndeavourOS's own signed shim (Calamares handles this) | DKMS rebuild automatic; rolling kernel can occasionally outrun `nvidia-open` — one pacman cycle of delay |
| **NVIDIA 580 via CUDA repo (Kubuntu 26.04)** | `nvidia-cuda-toolkit` is now **native in the Ubuntu archive** — no third-party repo needed | Signed | Automatic |

Specific to your hardware:

- **RTX 3050 Mobile is Ampere.** `nvidia-open` (the MIT-licensed kernel modules) works perfectly. This is the EndeavourOS default.
- **PRIME offload** (the `prime-run <app>` wrapper you already use for Gazebo) works on all three.
- **`supergfxctl` Hybrid/Integrated/Dedicated switching** — see [Axis 5](#axis-5--asus-tuf-a17-hardware-enablement).

| Aspect | Kubuntu | Fedora | EndeavourOS |
|--------|---------|--------|-------------|
| Install steps to CUDA working | 2 (driver install + reboot) | 5 (RPMFusion + akmod + MOK + CUDA repo + reboot) | 0 (done at OS install) |
| Failure modes | Almost none | MOK skip → no driver → black screen on reboot; fixable but scary | Rolling kernel outpacing `nvidia-open` → temporary nouveau fallback |
| Ongoing burden | None | Low (akmod is automatic) | Very low (`pacman -Syu` discipline) |

**Winner: EndeavourOS (zero-touch) very slightly over Kubuntu (one-touch). Fedora is a clear third** — the MOK dance is survivable but a foot-gun on Secure Boot systems, and the [BIOS setup](../02-setup/02-bios-uefi.md) for the TUF A17 leaves Secure Boot on by default.

---

## Axis 4 — PREEMPT_RT kernel access (subsea roadmap item 2)

[Learning roadmap item 2](../03-dev-environment/05-learning-roadmap.md#2-preempt_rt-real-time-kernel) calls for sub-100 µs worst-case scheduling latency under load — mandatory for autopilot control loops on ROV/AUV platforms. Your path to that kernel differs per distro:

| Distro | How you get PREEMPT_RT | Trust level | One-line install |
|--------|------------------------|-------------|-------------------|
| **Kubuntu 24.04** | Ubuntu Pro (free for 5 personal machines) | High; Canonical-signed; only ships the 6.8 branch | `sudo pro enable realtime-kernel` |
| **Kubuntu 26.04** | `preempt=full` on the stock Linux 7.0 kernel is often sufficient now (RT merged in 6.12); Pro realtime still available for strict cases | High | Boot parameter only for soft RT; `sudo pro enable realtime-kernel` for hard |
| **Fedora 43 KDE** | Either `preempt=full` on 6.17, or `@kwizart/kernel-longterm-rt` COPR, or source build | Medium (COPR is third-party) | `sudo dnf copr enable kwizart/kernel-longterm-rt && sudo dnf install kernel-rt` |
| **EndeavourOS** | Official Arch Extra repo ships both `linux-rt` and `linux-rt-lts` | High (official Arch) | `sudo pacman -S linux-rt linux-rt-lts` |

**Winner: EndeavourOS** for cleanliness (two packages, one command, in the official repo), **tied with Kubuntu 26.04** for convenience once 26.04.1 settles. Fedora is third because COPR is community-maintained and the package can lag the mainline RT patchset by weeks.

If you are serious about roadmap item 2 within the next 6 months, EndeavourOS has a real edge here. The caveat: having a convenient RT kernel does not help until you have also mastered CPU isolation (`isolcpus=`), IRQ affinity, and `cyclictest` methodology — all of which are distro-independent.

---

## Axis 5 — ASUS TUF A17 hardware enablement

The TUF A17 needs three things that no distro ships by default: `asusctl` (fan curves, keyboard backlight, battery charge limit), `supergfxctl` (Hybrid/Integrated/Dedicated GPU mode switching), and `rog-control-center` (GUI front-end). Availability:

| Distro | `asusctl` | `supergfxctl` | `rog-control-center` | Signing / service integration |
|--------|-----------|---------------|----------------------|-------------------------------|
| **Kubuntu 24.04 / 26.04** | `asus-linux.org` PPA (maintained by Luke Jones, upstream) | Same PPA | Same PPA | systemd services shipped; `asusd.service` runs at boot |
| **Fedora 43 KDE** | Official COPR `lukenukem/asus-linux` (same maintainer) | Same COPR | Same COPR | systemd services shipped; sometimes lags Ubuntu PPA by a minor release |
| **EndeavourOS** | AUR `asusctl`, `supergfxctl`, `rog-control-center` (all upstream-blessed) | Same AUR packages | Same | systemd services shipped |

All three paths lead to the same upstream code — the same Luke Jones maintains all three. In practice:

- **PPA on Kubuntu is the most-trodden path.** It is what the upstream project tests against first. It is also what [2.6 ASUS hardware](../02-setup/06-asus-hardware.md) assumes.
- **COPR on Fedora is well-maintained but sometimes 1–2 weeks behind the PPA** after a major release.
- **AUR on EndeavourOS tracks mainline fastest** but AUR packages are built locally — a failed build breaks your whole pacman transaction. This rarely happens for `asusctl` but is a real failure mode.

**Winner: Kubuntu by a narrow margin, specifically because [Part 2.6](../02-setup/06-asus-hardware.md) already documents the PPA path and you would not need to learn anything new.** Fedora and EndeavourOS are workable but come with a small translation tax.

---

## Axis 6 — Dual-boot + systemd-boot + BTRFS (your existing layout)

Your [current layout](../02-setup/03-install-partitioning.md) is:

- Disk A: Windows 11 with its own ESP, untouched.
- Disk B: Linux on BTRFS with zstd:1 subvolumes, systemd-boot as the loader, 20 GiB swapfile.
- F8 one-time boot menu selects the OS at power-on (no shared GRUB).

How each distro maps to that layout:

| Distro | BTRFS-by-default | systemd-boot-by-default | Preserves your existing layout on reinstall? |
|--------|------------------|--------------------------|----------------------------------------------|
| **Kubuntu 24.04 / 26.04** | Calamares offers BTRFS + subvolumes; systemd-boot is a manual post-install conversion ([2.3.2](../02-setup/03-install-partitioning.md)) | No (GRUB default, convertible) | Yes — identical to what you already documented |
| **Fedora 43 KDE** | Default is **BTRFS with Anaconda-chosen subvolumes** (`@`, `@home`) | No (GRUB default) | Partial — subvolume naming convention differs; you would need a reinstall to rename them |
| **EndeavourOS** | Calamares offers BTRFS + subvolumes | Yes (systemd-boot is an installer option since Galileo) | Closest to your current layout **but** has a **known bug**: Windows 11 on a separate drive using systemd-boot can fail to chainload — source: Ganymede Neo release notes. Dual-boot on the same disk works normally |

That last bullet is the decisive one. Your setup is **two separate SSDs with separate ESPs, selected via F8**. On EndeavourOS Ganymede Neo specifically, the installer's systemd-boot + separate-Windows-drive path is currently broken. You can:

1. Install EndeavourOS with GRUB instead (which reintroduces shared-bootloader risk — the exact problem [Part 2](../02-setup/README.md) was designed to avoid).
2. Accept that on F8 boot to Windows, you may see the EndeavourOS menu briefly — survivable but ugly.
3. Wait for the fix (no ETA).

| Scenario | Kubuntu (24.04 or 26.04) | Fedora 43 KDE | EndeavourOS |
|----------|---------------------------|---------------|-------------|
| BTRFS subvolumes identical to [2.3](../02-setup/03-install-partitioning.md) | Yes | Partial (different names) | Yes |
| systemd-boot with F8 dual-boot | Yes | Needs manual conversion | Known bug, not yet fixed |
| Reuse your [Timeshift/Snapper snapshots](../02-setup/05-kernel-boot.md) | Yes | Reformat required | Reformat required |

**Winner: Kubuntu by a wide margin.** Fedora is usable with 2–3 hours of reconfiguration. EndeavourOS has the systemd-boot bug specifically in the layout you already run.

---

## Axis 7 — Package ecosystem depth

| Need | Kubuntu | Fedora 43 KDE | EndeavourOS |
|------|---------|---------------|-------------|
| **STM32CubeIDE** (vendor binary, .deb) | Native `.deb` install | AppImage or Flatpak (vendor does not ship `.rpm`) | AUR `stm32cubeide` |
| **KiCad 9** | `kicad/kicad-9` PPA | `dnf copr enable @kicad/kicad` | `extra/kicad` (in sync with mainline) |
| **CUDA 13.2 toolkit** | NVIDIA official Ubuntu repo (24.04) · native `nvidia-cuda-toolkit` (26.04) | NVIDIA official repo for Fedora 40+ (usable on 43) | AUR `cuda` |
| **STM32 OpenOCD + arm-none-eabi-gcc** | APT stock | `dnf` stock | `extra/openocd` + AUR toolchain |
| **J-Link tools** | Segger official `.deb` | Segger official `.rpm` | AUR `jlink-software-and-documentation` |
| **PX4 / Ardusub SITL** | APT install (Ubuntu 24.04 official) | Source build | Source build or AUR |
| **QGroundControl** | Flatpak `org.mavlink.qgroundcontrol` | Same | Same |
| **ROS 2 Jazzy on host** (if not containerised) | Official Ubuntu 24.04 binaries | Requires source build or container | AUR `ros2-jazzy-*` community packages |
| **asusctl / supergfxctl** | `asus-linux.org` PPA | COPR `lukenukem/asus-linux` | AUR |
| **Raw package count available** | ~60k APT + ~3k Flatpak | ~50k DNF + RPMFusion + Flathub | ~15k pacman + ~80k AUR + Flathub |

The axis is close. A few explicit observations:

- **AUR is the widest ecosystem by raw count** but every AUR package is a source build whose success depends on the community maintainer. For hardware tools (Segger J-Link, STM32CubeIDE), vendor `.deb`/`.rpm` is always preferable to AUR.
- **Fedora 43 KDE has no native `.deb` support** — Segger, STMicro, and ST-Link Server all ship `.deb` first and `.rpm` with a delay. Minor ongoing tax.
- **Kubuntu's APT + PPA coverage is the most vendor-supported path.** Every hardware vendor tests against Ubuntu LTS first.

**Winner: Kubuntu for vendor-supported paths (STM32, J-Link, PX4). EndeavourOS for raw breadth (AUR). Fedora third** — a sometimes-annoying middle ground where vendors ship `.deb` before `.rpm`.

---

## Axis 8 — Cost of switching from Kubuntu 24.04

You are currently on Kubuntu 24.04 LTS per [Part 2](../02-setup/README.md). Cost to move off:

| Target | Time cost | Data-loss risk | Learning-curve tax |
|--------|-----------|----------------|---------------------|
| **Stay on Kubuntu 24.04** | 0 h | None | None |
| **Upgrade to Kubuntu 26.04.1** (Q3 2026) | ~90 min `do-release-upgrade` after Timeshift snapshot | Low — same package format, same subvolume layout | Small — Wayland default and Plasma 6.6 learning |
| **Reinstall to Fedora 43 KDE** | 4–6 h fresh install + reconfigure + re-learn `dnf`/SELinux/firewalld | Moderate if you skip the snapshot step | Significant — new package manager, new firewall, new SELinux contexts on `/dev/ttyUSB0` |
| **Reinstall to EndeavourOS** | 3–5 h fresh install + pacman crash-course + AUR setup (`yay`/`paru`) | Moderate | Significant — rolling-release discipline, AUR vetting, manual config migration |

The time cost is not the main issue. The **hidden cost** is that every distro-specific paper trail in this repository ([Part 2](../02-setup/README.md), [Part 3](../03-dev-environment/README.md), [Appendix A provisioning script](../appendix-a-provisioning.md), [Appendix B troubleshooting](../appendix-b-troubleshooting.md)) is Kubuntu-targeted. Switching to Fedora or EndeavourOS invalidates:

- [2.4 Debloat](../02-setup/04-debloat.md) — Snap removal is irrelevant; KDE PIM prune translates but command names differ.
- [2.6 ASUS hardware](../02-setup/06-asus-hardware.md) — PPA path does not exist; would be COPR or AUR.
- [Appendix A provisioning script](../appendix-a-provisioning.md) — 100% APT-based, full rewrite required.
- [Appendix B troubleshooting](../appendix-b-troubleshooting.md) — many entries are Ubuntu-specific (e.g. Pro realtime enablement, `ubuntu-drivers`).

**Winner: Kubuntu, decisively** — zero cost is hard to beat.

---

## Verdict

Across all eight axes:

| Axis | Winner |
|------|--------|
| 1. Release model & LTS horizon | Kubuntu |
| 2. ROS 2 via Distrobox | Three-way tie |
| 3. NVIDIA | EndeavourOS (narrowly), Kubuntu close second |
| 4. PREEMPT_RT | EndeavourOS, tied with Kubuntu 26.04 |
| 5. ASUS TUF A17 enablement | Kubuntu |
| 6. Dual-boot + systemd-boot + BTRFS | Kubuntu |
| 7. Package ecosystem depth | Kubuntu (vendor) / EndeavourOS (breadth) — split |
| 8. Cost of switching | Kubuntu |

**Primary verdict: Kubuntu wins** — 5 outright + 1 shared + 1 tied. Fedora wins no axis outright.

**Runner-up: EndeavourOS.** It wins NVIDIA and PREEMPT_RT, ties on ROS 2, and has the broadest package ecosystem. Its disqualifier for *your* setup is Axis 6 (the systemd-boot + separate-Windows-drive bug). If you did single-disk dual-boot or if that bug is fixed by the time you read this, EndeavourOS moves to a clear second place for engineers who specifically prioritize real-time work.

**Third place: Fedora 43 KDE.** Technically excellent — Podman and Distrobox are Red Hat's own tools — but the 13-month force-upgrade cadence is wrong for your 3–4 year stability horizon, and it wins no axis outright against Kubuntu or EndeavourOS.

**Concrete decision rule for you:**

| If... | Then install... |
|-------|------------------|
| You are installing Linux today (April 2026) and want maximum stability | [Kubuntu 24.04 LTS](../02-setup/README.md) |
| You are installing in Q3 2026 (August+) | Kubuntu 26.04.1 LTS — see [Part 2 of /best](02-ubuntu-26.04-evaluation.md) |
| You specifically prioritize PREEMPT_RT + real-time subsea autopilot work, accept rolling-release, and are willing to use GRUB for dual-boot | EndeavourOS |
| You want a 6-month leading-edge workstation and enjoy reinstalling annually | Fedora 43 KDE (or 44 when it ships in May 2026) |

The consolidated conclusion is in [Part 1 (v2) — 2026 Re-verdict](../01-verdict_v2.md). Continue to [Part 2 of /best — Ubuntu 26.04 Evaluation](02-ubuntu-26.04-evaluation.md) for the specific question of whether to upgrade off 24.04.

---

[Home](../../README.md) · [↑ /best](README.md) · [← Previous: /best Intro](README.md) · **1. Deep Research** · [Next: 2. Ubuntu 26.04 Eval →](02-ubuntu-26.04-evaluation.md)
