[Home](../../README.md) · [← Previous: The Verdict (v1)](../01-verdict.md) · **/best — 2026 Distro Landscape Research** · [Next: 1. Deep Research →](01-distro-deep-research.md)

---

# /best — 2026 Linux Distro Landscape Research

Two deep-research documents written on **2026-04-18**, five days before Ubuntu 26.04 LTS "Resolute Raccoon" ships, to answer two questions the original [Verdict (v1)](../01-verdict.md) could not:

1. Now that Podman + Distrobox ([3.1 Containers](../03-dev-environment/01-containers.md)) absorbs ROS 2's host-distro requirement, is Kubuntu still the right host — or does **Fedora KDE 43** / **EndeavourOS "Ganymede Neo"** become competitive?
2. Does **Kubuntu 26.04 LTS** (Linux 7.0, Plasma 6.6, sudo-rs, Wayland-by-default, native CUDA/ROCm) change the calculus enough to jump off 24.04?

The verdict is consolidated in [Part 1 (v2) — 2026 Re-verdict](../01-verdict_v2.md). If you only want the answer, read that page.

## Contents

| # | Document | What it decides | Time to read |
|---|----------|-----------------|--------------|
| 1 | [Fedora KDE vs EndeavourOS vs Kubuntu — 2026 Deep Research](01-distro-deep-research.md) | Which host distro wins across 8 axes (release cadence, ROS 2, NVIDIA, PREEMPT_RT, TUF enablement, dual-boot, package ecosystem, switching cost) | ~15 min |
| 2 | [Ubuntu 26.04 LTS "Resolute Raccoon" — Upgrade Evaluation](02-ubuntu-26.04-evaluation.md) | Whether to upgrade from Kubuntu 24.04 LTS, and if so, when (spoiler: wait for **26.04.1** in August 2026) | ~10 min |

## Why these documents exist

The [original Verdict](../01-verdict.md) was written when:

- ROS 2 Jazzy was only officially supported on Ubuntu 24.04 as a host (binding the distro choice to Ubuntu).
- Kubuntu 24.04 was the shipping LTS, with Plasma 5.27 and X11 as the safe default on NVIDIA.
- Fedora's Podman/Distrobox tooling was excellent but peripheral to the decision.
- EndeavourOS was running old Plasma 6.1 with non-auto NVIDIA driver selection.

Four facts changed between late 2024 and April 2026:

1. **Distrobox matured.** The `ros2-jazzy` container pattern in [3.1](../03-dev-environment/01-containers.md) abstracts the host distro for ROS 2 entirely.
2. **Kubuntu 26.04 LTS ships in 5 days** with Plasma 6.6, Linux 7.0, and a Wayland-only default session that invalidates the v1 "X11 for NVIDIA stability" reasoning.
3. **EndeavourOS "Ganymede Neo" (2026.01.12)** added auto NVIDIA driver detection and switched to `nvidia-open` — the user's RTX 3050 Mobile (Ampere) is supported.
4. **Fedora 43 KDE Spin** (Oct 2025) is the most polished Fedora KDE release to date, with Plasma 6.4.5 and kernel 6.17.

If none of those had happened, v1 would still be the answer. They did, so the question is worth re-asking.

## Reading order

**If you have 25 minutes:** read both documents in order, then [Part 1 (v2)](../01-verdict_v2.md).

**If you have 5 minutes:** skip directly to [Part 1 (v2)](../01-verdict_v2.md) — the verdict cites these research docs as footnotes.

**If you are installing Linux this week:** [Part 1 (v2)](../01-verdict_v2.md) first. The TL;DR is: install Kubuntu 24.04 LTS now (unchanged from v1), plan the upgrade to Kubuntu 26.04.1 LTS for August 2026.

---

[Home](../../README.md) · [← Previous: The Verdict (v1)](../01-verdict.md) · **/best — 2026 Distro Landscape Research** · [Next: 1. Deep Research →](01-distro-deep-research.md)
