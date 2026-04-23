[Home](../README.md) · [← Previous: /best 2. Ubuntu 26.04 Eval](best/02-ubuntu-26.04-evaluation.md) · **Part 1 (v2): The 2026 Re-verdict** · [Next: Setup Guide →](02-setup/README.md)

---

# Part 1 (v2) — The 2026 Landscape Re-verdict

**Dated: 2026-04-18.** This page supersedes [Part 1 — The Verdict (v1)](01-verdict.md). The v1 page is retained for historical record and still accurately describes the Kubuntu-24.04-vs-Ubuntu-24.04 decision as of late 2024.

The v1 verdict was **Kubuntu 24.04 LTS**. The v2 verdict — after considering four 2026 developments: Kubuntu 26.04 LTS (ships in 5 days), Fedora 43 KDE Spin, EndeavourOS "Ganymede Neo", and Distrobox absorbing the ROS 2 host-distro requirement — is **still Kubuntu**, but the target moves from 24.04 to 26.04.1 on a timeline.

**Navigate within this page:**

- [The verdict in one paragraph](#the-verdict-in-one-paragraph)
- [Three-column decision summary](#three-column-decision-summary)
- [Revised non-negotiable technical positions](#revised-non-negotiable-technical-positions)
- [Why not Fedora, why not EndeavourOS](#why-not-fedora-why-not-endeavouros-in-one-line-each)
- [Trigger conditions to revisit](#trigger-conditions-to-revisit-this-verdict)

---

## The verdict in one paragraph

**Install or stay on [Kubuntu 24.04 LTS](02-setup/README.md) today. Plan the upgrade to Kubuntu 26.04.1 LTS for August 2026**, once the first bug-fix point release ships. This is the right answer because (a) Podman + Distrobox makes the host-distro choice less decisive than it was — any of the three finalists could run ROS 2 Jazzy correctly — but every other axis (dual-boot compatibility with your existing systemd-boot + BTRFS + two-SSD layout, vendor-tested `asus-linux.org` PPA path for TUF hardware, zero-reconfiguration cost, 10-year support horizon with Ubuntu Pro, and the existing paper trail in [Part 2](02-setup/README.md) and [Appendix A](appendix-a-provisioning.md)) favours Kubuntu, and (b) Kubuntu 26.04.0 is a high-risk `.0` release (first Linux 7.0, first sudo-rs, first Plasma 6.6 Wayland-default with NVIDIA as the mainstream LTS path) that should be allowed to bake for four months before landing on a workhorse laptop. Full reasoning in [/best/01](best/01-distro-deep-research.md) and [/best/02](best/02-ubuntu-26.04-evaluation.md).

---

## Three-column decision summary

| Dimension | Kubuntu 24.04 LTS (now) | Kubuntu 26.04.1 LTS (~Aug 2026) | Fedora 43 KDE / EndeavourOS (alternative paths) |
|-----------|--------------------------|----------------------------------|--------------------------------------------------|
| **Install action** | None — already documented in [Part 2](02-setup/README.md) | `sudo do-release-upgrade` with Timeshift rollback | Fresh reinstall, 4–6 h; invalidates paper trail |
| **Kernel** | 6.8 GA → 6.17 HWE | **7.0** (new scheduler, XFS self-healing, RT via `preempt=full`) | Fedora: 6.17 · EndeavourOS: 6.18 LTS / `linux-rt` |
| **Desktop** | Plasma 5.27 LTS on X11 | **Plasma 6.6 on Wayland** (X11 not installed) | Fedora KDE: 6.4.5 · EndeavourOS: 6.5.4 |
| **NVIDIA** | Driver 580, X11 session, `ubuntu-drivers install` | Driver 580+ on Wayland (explicit sync), same install command | Fedora: MOK enrolment dance · EndeavourOS: zero-touch `nvidia-open` |
| **CUDA** | NVIDIA APT repo (extra step in [3.3](03-dev-environment/03-ai-ml.md)) | **Native `nvidia-cuda-toolkit` in archive** — one `apt install` | NVIDIA repo on Fedora · AUR `cuda` on EndeavourOS |
| **PREEMPT_RT** | Ubuntu Pro realtime add-on (free, 5 machines) | `preempt=full` boot param covers soft RT; Pro still available | EndeavourOS: `pacman -S linux-rt linux-rt-lts` (cleanest) |
| **LTS window** | May 2029 (Pro to 2034, Legacy to 2036) | **April 2031 (Pro to 2036, Legacy to 2038)** | Fedora: ~13 months · EndeavourOS: rolling |
| **ASUS TUF tooling** | [`asus-linux.org` PPA](02-setup/06-asus-hardware.md) | Same PPA supports 26.04 from day 0 | Fedora COPR (1–2 wk lag) · EndeavourOS AUR |
| **Dual-boot layout** | Matches [2.3](02-setup/03-install-partitioning.md) exactly | Matches (upgrade preserves layout) | Fedora: reformat subvolumes · EndeavourOS: known systemd-boot bug |
| **Switching cost from current state** | **0 hours** | ~90 min upgrade + 1 weekend Wayland tuning | 4–6 h reinstall + ~2 weeks of reconfiguration |
| **Risk profile** | Lowest (mature) | Medium (wait for `.1` neutralises it) | Medium-High (new paper trail needed) |

**The third column is not ranked lower because the distros are inferior** — Fedora and EndeavourOS are technically excellent. It is ranked lower because *your* paper trail, hardware layout, and stability horizon are all Kubuntu-shaped. The cost of switching is the deciding factor, not the raw quality of the alternative.

---

## Revised non-negotiable technical positions

This replaces the table in [00-executive-summary.md § Non-Negotiable Technical Positions](00-executive-summary.md#non-negotiable-technical-positions) for the 26.04.1 target. Rows marked **(change)** differ from the 24.04 configuration; rows marked **(keep)** are unchanged.

| Decision | Value on 24.04 (today) | Value on 26.04.1 (post-Aug 2026) | Status |
|----------|-------------------------|-----------------------------------|--------|
| Distro | Kubuntu 24.04 LTS | **Kubuntu 26.04 LTS** (via upgrade or fresh install) | (change) |
| Session | **X11** | **Wayland** (Plasma 6.6 + NVIDIA 580 explicit sync is now safe) | (change) |
| Filesystem | BTRFS + zstd:1 subvolumes | BTRFS + zstd:1 subvolumes | (keep) |
| Bootloader | systemd-boot (GRUB acceptable) | systemd-boot | (keep) |
| Swap | 20 GiB BTRFS swapfile + zram | Same | (keep) |
| NVIDIA driver | Proprietary 580 + PRIME offload | Proprietary 580+ (may be 590 by Aug 2026) + PRIME offload | (keep path; version floats) |
| CUDA install | NVIDIA's official Ubuntu APT repo | **Native `nvidia-cuda-toolkit` from the Ubuntu archive** | (change — simpler) |
| Real-time kernel | Ubuntu Pro `realtime-kernel` add-on | **`preempt=full` boot parameter on stock kernel 7.0** for soft RT; Pro still available for hard RT | (change — less dependency) |
| TUF hardware control | `asusctl` + `supergfxctl` via PPA | Same PPA, same tools (PPA already supports 26.04) | (keep) |
| Python | `uv` | `uv` (default interpreter becomes 3.14; pin older via `uv venv --python`) | (keep tool; keep an eye on wheel lag) |
| Node | `fnm` | `fnm` | (keep) |
| Container runtime | Rootless Podman + Distrobox | Same | (keep) |
| sudo | C sudo 1.9 | **sudo-rs** (Rust, memory-safe) | (change — transparent) |
| SSH / TLS | ECDHE / Ed25519 | **Post-quantum ML-KEM hybrid default** | (change — transparent) |
| Support horizon | May 2029 / 2034 Pro / 2036 Legacy | **April 2031 / 2036 Pro / 2038 Legacy** | (improvement) |

Nothing in [Part 2](02-setup/README.md), [Part 3](03-dev-environment/README.md), or [Part 4](04-applications.md) requires rewriting today. The items that will need edits after the Aug 2026 upgrade are listed in [/best/02 § Files to revisit after the upgrade](best/02-ubuntu-26.04-evaluation.md#files-to-revisit-after-the-upgrade).

---

## Why not Fedora, why not EndeavourOS (in one line each)

- **Fedora 43 KDE Spin is excellent tech but wins zero axes outright** in [/best/01](best/01-distro-deep-research.md#verdict). The 13-month force-upgrade cadence is wrong for a 3–4 year stability horizon, the Secure Boot + MOK enrollment dance on install is survivable-but-fragile, and no vendor (STM32, Segger, Canonical's own PPAs) tests Fedora first.
- **EndeavourOS "Ganymede Neo" wins on NVIDIA zero-touch and PREEMPT_RT cleanliness** but is disqualified for *your* specific hardware setup by the known systemd-boot + separate-Windows-drive chainload bug. If that bug is fixed and your primary workload tilts hard toward real-time subsea autopilot control, EndeavourOS becomes a real second choice.

Neither verdict is "Fedora/Endeavour is bad." It is: *for this hardware, this workload, and this paper trail, Kubuntu is the highest-leverage answer.*

---

## Trigger conditions to revisit this verdict

Re-open the question if any of the following happens:

1. **Hardware refresh.** New laptop = new paper trail anyway = freeze 26.04.1 LTS on the new box, consider EndeavourOS on the old one as a secondary RT-kernel dev machine.
2. **Kubuntu 26.04 `.0` is abandoned** (no `.1` by September 2026). Fall back to staying on 24.04 until 2029, or consider 26.10 interim if hardware support demands it.
3. **Canonical drops the `realtime-kernel` add-on from the free Ubuntu Pro tier**, AND `preempt=full` on Linux 7.0 turns out to not meet your ROV autopilot latency targets. Then EndeavourOS with `linux-rt` becomes the correct answer.
4. **You commit to a subsea robotics employer's stack that mandates Fedora or RHEL** (unusual but possible — some US defence primes and research labs run on RHEL). Then Fedora 43/44 KDE becomes the right answer.
5. **A critical upstream (ROS 2, CUDA, KiCad, STM32CubeIDE) drops Ubuntu LTS before Aug 2026.** Unlikely to the point of not being worth planning for.

---

## See also

- [/best — 2026 Distro Landscape Research](best/README.md) — landing page for both research documents
- [/best/01 — Fedora KDE vs EndeavourOS vs Kubuntu Deep Research](best/01-distro-deep-research.md) — the 8-axis comparison this verdict rests on
- [/best/02 — Ubuntu 26.04 LTS "Resolute Raccoon" Evaluation](best/02-ubuntu-26.04-evaluation.md) — the upgrade timing analysis
- [Part 1 (v1) — Original Kubuntu vs Ubuntu 24.04 Verdict](01-verdict.md) — retained for historical record
- [Part 2 — Setup Guide](02-setup/README.md) — unchanged install procedure for Kubuntu 24.04 LTS today

---

[Home](../README.md) · [← Previous: /best 2. Ubuntu 26.04 Eval](best/02-ubuntu-26.04-evaluation.md) · **Part 1 (v2): The 2026 Re-verdict** · [Next: Setup Guide →](02-setup/README.md)
