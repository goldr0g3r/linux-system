[Home](../README.md) · [← Previous: 2.5 Snapshots](../02-post-install-foundations/05-snapshot-policy-btrfs-timeshift.md) · **03 — Speed & Boot Optimizations** · [Next: 3.1 Kernel params →](01-kernel-boot-parameters.md)

---

# 03 — Speed, Boot Time, and Responsiveness

Targeted optimisations to get a modern Kubuntu 26.04 install under **12 seconds to login prompt** on a Ryzen 4800H + NVMe, and keep it there through software accretion.

## Steps (in order)

| #   | Step                                                                           | Time    | Skippable?                     |
| ---:| ------------------------------------------------------------------------------ | ------- | ------------------------------ |
| 3.1 | [Kernel boot parameters (`preempt=full`, NVIDIA DRM KMS, NVMe power states)](01-kernel-boot-parameters.md) | 10 min | No                             |
| 3.2 | [zram, swappiness, and sysctl tuning](02-zram-swap-sysctl.md)                 | 10 min  | No                             |
| 3.3 | [Service masking + journald capping](03-service-masking-and-journald.md)      | 10 min  | Partially — safe subset vs aggressive |
| 3.4 | [Filesystem mount options, `fstrim`, thermals, power profiles](04-filesystem-thermals-power.md) | 15 min | No                             |

Total: ~45 minutes.

## What "speed" means here

- **Boot time:** POST → SDDM login prompt in ≤ 12 s cold, ≤ 6 s warm. Measured by `systemd-analyze` and `systemd-analyze blame`.
- **Desktop responsiveness:** Plasma login → interactive in ≤ 3 s after greeter.
- **Application launch:** Cold Firefox/VS Code start in ≤ 2 s (warm cache; first launch of the day will be 3–5 s).
- **Battery drag:** Idle discharge ≤ 4 W with screen at 50 % brightness, Wi-Fi on, `Integrated` GPU mode (`supergfxctl -m Integrated`).

## Baseline measurement before you start

```bash
systemd-analyze
systemd-analyze blame | head -20
systemd-analyze critical-chain

# Cold-start single-app measurement (repeat 3x, take minimum):
systemd-run --scope --user -p MemoryLimit=4G bash -c 'time firefox -headless about:blank'
```

Write these numbers in a scratch file (e.g. `~/tuning-baseline-$(date -I).txt`). After [3.1–3.4](01-kernel-boot-parameters.md) re-measure and keep the diff. If the diff is flat, roll back anything that introduced regressions.

## What we deliberately do NOT do

- **Install `preload`, `prelockd`, `readahead-fedora`.** These were valuable on HDDs and old SSDs. On a modern NVMe (sequential reads ≥ 3 GB/s) their benefit is noise. Install only if `systemd-analyze blame` shows a cold-cache application actually being bottlenecked on disk reads, which is rare.
- **Compile a custom kernel.** Linux 7.0 stock on 26.04 is already modern; `preempt=full` exposes the RT scheduler without a custom build.
- **Swap `systemd` for OpenRC / dinit / runit.** Those are projects, not improvements — on Kubuntu you will spend more time chasing unit-file quirks than you save at boot.
- **Use `earlyoom` by default.** Kernel 7.0 has `PSI`-based MemoryPressure slice configuration available through systemd-oomd, which Ubuntu ships enabled. Add `earlyoom` only if OOM events in the logs prove it's needed.
- **Aggressively disable `NetworkManager` in favour of `systemd-networkd`.** `systemd-networkd` saves ~0.3 s at boot at the cost of losing KDE's `plasma-nm` integration. Not worth it.

---

[Home](../README.md) · [← Previous: 2.5 Snapshots](../02-post-install-foundations/05-snapshot-policy-btrfs-timeshift.md) · **03 — Speed & Boot Optimizations** · [Next: 3.1 Kernel params →](01-kernel-boot-parameters.md)
