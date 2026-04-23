[Home](../README.md) · [← Previous: 1.5 First boot](../01-install/05-first-boot-and-snapshot.md) · **02 — Post-Install Foundations** · [Next: 2.1 Debloat →](01-debloat-snap-flatpak-pim.md)

---

# 02 — Post-Install Foundations

Everything you do in the first 60 minutes of the fresh Kubuntu 26.04 install. These steps turn a stock Kubuntu into your working baseline: no snap, proprietary NVIDIA with CUDA, ASUS TUF tooling live, Wayland tuned for your hybrid GPU, and a snapshot discipline in place.

## Steps (in order)

| #   | Step                                                                           | Time    | Skippable?                                   |
| ---:| ------------------------------------------------------------------------------ | ------- | -------------------------------------------- |
| 2.1 | [Debloat: snap removal, Flatpak + Flathub, Firefox via Mozilla APT, KDE PIM prune](01-debloat-snap-flatpak-pim.md) | 10 min  | No — snap removal changes later APT behaviour |
| 2.2 | [NVIDIA driver 580 + native `nvidia-cuda-toolkit` (new in 26.04)](02-nvidia-driver-580-and-cuda.md) | 20 min | No — even if you don't do AI/ML you still want hybrid-GPU |
| 2.3 | [ASUS TUF hybrid-GPU enablement (`asusctl`, `supergfxctl`, fan curves, battery cap)](03-asus-tuf-hybrid-gpu.md) | 15 min | No — TUF needs this |
| 2.4 | [Wayland + Plasma 6.6 + NVIDIA tuning (Explicit Sync, XWayland, fractional scaling)](04-wayland-plasma-66-nvidia.md) | 15 min | Partially — defaults are OK for many |
| 2.5 | [Snapshot policy: BTRFS + Timeshift rollback discipline](05-snapshot-policy-btrfs-timeshift.md) | 10 min  | **No. Ever.** |

Total: ~70 minutes of active time including three reboots.

## Ordering rationale

Why debloat first: `snap` installs a daemon, mounts a FUSE per package, and snarls `ubuntu-drivers`' dependency resolution. Kill it before installing NVIDIA.

Why NVIDIA before ASUS: `supergfxctl` inspects the NVIDIA driver state to decide which mode to switch into. Install the driver first, then the TUF tooling knows what to talk to.

Why Wayland tuning after ASUS: some Wayland polish items (e.g. fractional scaling on dGPU) depend on `supergfxctl` being in `Hybrid` mode.

Why snapshot policy last in this section: the first Timeshift was taken at the end of [1.5](../01-install/05-first-boot-and-snapshot.md). This page turns ad-hoc snapshotting into a repeatable policy (retention, pre-apt hook, Snapper vs Timeshift trade-off).

## Reboots you will do in this section

1. After [2.2](02-nvidia-driver-580-and-cuda.md) — NVIDIA driver kernel module needs a cold start.
2. After [2.3](03-asus-tuf-hybrid-gpu.md) — `supergfxctl` first mode switch requires a reboot.
3. Optional after [2.4](04-wayland-plasma-66-nvidia.md) — only if you changed kernel command-line parameters for Explicit Sync.

---

[Home](../README.md) · [← Previous: 1.5 First boot](../01-install/05-first-boot-and-snapshot.md) · **02 — Post-Install Foundations** · [Next: 2.1 Debloat →](01-debloat-snap-flatpak-pim.md)
