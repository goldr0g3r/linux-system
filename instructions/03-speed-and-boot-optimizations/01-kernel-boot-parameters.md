[Home](../README.md) · [↑ 03 Speed & Boot](README.md) · [← Previous: 03 Speed & Boot (section)](README.md) · **3.1 Kernel boot parameters** · [Next: 3.2 zram & sysctl →](02-zram-swap-sysctl.md)

---

# 3.1 Kernel Boot Parameters

A small set of kernel command-line parameters that noticeably improve boot time, Wayland + NVIDIA stability, NVMe power management, and (optionally) soft-real-time latency on Linux 7.0.

## Two places they are configured

Kubuntu 26.04 with GRUB (the default):

- `/etc/default/grub` — the `GRUB_CMDLINE_LINUX_DEFAULT=` line.
- Regenerate the runtime config with `sudo update-grub` after every edit.

If you converted to systemd-boot (optional, in [1.4](../01-install/04-installation-and-partitioning.md)):

- `/boot/efi/loader/entries/*.conf` — the `options` line.
- No regeneration needed; systemd-boot reads these live.

This page shows the GRUB form; the values are identical in systemd-boot.

## Snapshot first

```bash
sudo timeshift --create --comments "before-kernel-params $(date -I)" --tags D
```

## 3.1.1 The recommended cmdline for TUF A17 + 26.04 + RTX 3050

Open `/etc/default/grub`:

```bash
sudoedit /etc/default/grub
```

Set:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash \
  nvidia-drm.modeset=1 nvidia-drm.fbdev=1 \
  nvidia.NVreg_PreserveVideoMemoryAllocations=1 \
  nvidia.NVreg_EnableGpuFirmware=1 \
  nvidia.NVreg_UsePageAttributeTable=1 \
  nvme_core.default_ps_max_latency_us=5500 \
  mitigations=auto \
  amd_pstate=active"
```

Save. Regenerate the GRUB config:

```bash
sudo update-grub
```

Reboot, and verify with `cat /proc/cmdline`.

## Parameter-by-parameter explanation

### `quiet splash` (default, keep)

Suppresses kernel-log output during boot and shows the Plasma splash. Remove `quiet` if you want to see kernel messages during boot (useful while diagnosing boot hangs; not useful day-to-day).

### `nvidia-drm.modeset=1`

**Mandatory for Wayland on NVIDIA.** Tells the NVIDIA kernel module to register with the DRM subsystem, exposing the device as a proper DRM device that KWin can use for modesetting and framebuffer allocation. Without this, KWin-Wayland falls back to a software renderer → desktop flickers and is unusable.

Kubuntu 26.04 ships a /etc/modprobe.d/nvidia-drm.conf that forces this by default, so technically you don't have to add it to the cmdline, but double-redundancy here is harmless and makes the system's expectations explicit.

### `nvidia-drm.fbdev=1`

Enables an NVIDIA-driver-provided framebuffer console (early boot text on dGPU-driven displays). On hybrid-GPU, the iGPU drives the display in Hybrid mode, so this mostly matters when you switch to `AsusMuxDgpu` mode — without it, the boot text splash on dGPU-only setups shows as a tiny 80x25 fallback. Harmless if you never use AsusMuxDgpu.

### `nvidia.NVreg_PreserveVideoMemoryAllocations=1`

**Critical for suspend/resume reliability.** Before this flag (default-enabled on 580+ but not always picked up without the explicit setting), NVIDIA's driver would discard VRAM contents on suspend, leading to black screens on resume if the dGPU had an active context. Setting this to 1 forces VRAM preservation across suspend-to-RAM. The cost is slightly higher resume time (~0.5 s) and slightly more RAM used to back VRAM during suspend.

### `nvidia.NVreg_EnableGpuFirmware=1`

**Enables GSP (GPU System Processor) firmware.** Required on Turing and later for full feature support; Ampere (your 3050 Mobile) has a GSP that the driver talks to. Default is 1 on 580+, but explicit here. Do not set to 0 — that mode is unsupported on modern drivers.

### `nvidia.NVreg_UsePageAttributeTable=1`

Tells the NVIDIA driver to use the CPU's Page Attribute Table for MMIO memory regions, which gives a small performance boost on many workloads. Has been a safe default since at least 470.

### `nvme_core.default_ps_max_latency_us=5500`

**TUF A17 NVMe stability fix.** Without this, on some TUF A17 firmware revisions the NVMe controller enters aggressive low-power states (PS4, PS5) and occasionally fails to wake in time, causing `nvme0: Device not ready` errors and filesystem hangs. Capping max latency at 5500 microseconds keeps the NVMe in PS3 at worst, which is still low-power but reliable. Costs ~0.5 W extra idle; worth every milliwatt for a stable NVMe.

If you have a different NVMe manufacturer (Samsung 980 Pro, Kingston NV2, WD SN850X, etc.), 5500 is safe across all I've seen. Increase to 10000 if you encounter NVMe errors; decrease to 2000 if battery-life-obsessed and you have verified your NVMe is stable.

### `mitigations=auto`

The default. Enables all kernel CPU-vulnerability mitigations (Meltdown, Spectre, MDS, Retbleed, ...). **Do not set `mitigations=off`** unless you are benchmarking — the performance win is ~5 % on some workloads and the security cost is an order of magnitude more risk of a browser JS payload reading kernel memory. Worth being explicit.

### `amd_pstate=active`

Uses AMD's modern `amd-pstate` CPU frequency driver instead of the older `acpi-cpufreq`. On Zen 2 Ryzen 4800H this gives significantly better frequency scaling and better idle power than `acpi-cpufreq`, enabled by default on 26.04 Linux 7.0 but still worth explicit.

Three possible values:

- `amd_pstate=active` — preferred; the AMD driver controls P-state directly via MSR_AMD_CPPC_REQ.
- `amd_pstate=passive` — the kernel governor (schedutil/performance/powersave) controls via the AMD driver. Acceptable.
- `amd_pstate=guided` — the CPU firmware controls within bounds set by the kernel. Most power-efficient, slightly less responsive.

For a dev laptop, `active` is the right pick. If battery life is more important than instantaneous responsiveness, try `guided`.

## 3.1.2 Optional: `preempt=full` for soft real-time

Linux 7.0 merged the PREEMPT_RT fast-path into the mainline scheduler. The `preempt=full` boot parameter enables full preemption — giving you soft-real-time latencies (typically < 500 µs worst-case under load) without needing the out-of-tree PREEMPT_RT patch set.

**When you want this:**

- You are developing PX4 autopilot or ROS 2 real-time controllers locally.
- You are testing latency-critical audio (JACK pro audio, < 5 ms roundtrip).
- You want to benchmark the RT kernel without installing Ubuntu Pro.

**When you don't:**

- Throughput-heavy workloads (compiling, training) — full preemption costs ~3–5 % CPU throughput because of more context switches.
- Battery life priority — full preemption means more wakeups, more dynamic voltage/frequency switching, shorter idle deep-sleep residency.

Add to cmdline:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="... existing ... preempt=full"
```

Verify after reboot:

```bash
cat /sys/kernel/debug/sched_features 2>/dev/null | head -1
# Or simpler:
dmesg | grep -i "preempt" | head -3
# Expect: "preempt: full"
```

For hard real-time (< 100 µs), you still want the Ubuntu Pro `realtime-kernel` add-on (free for 5 personal machines).

## 3.1.3 NOT recommended parameters to avoid

- `nomodeset` — disables KMS entirely. The opposite of what we want.
- `acpi=off` — fixes nothing; breaks everything ACPI-driven (suspend, battery, thermal).
- `irqpoll` — used to fix stuck interrupts; on 26.04 kernel 7.0 you do not hit this, and enabling it wastes CPU.
- `nolapic` / `noapic` — legacy pre-2000s fixes. Not applicable.
- `pci=noaer` — PCIe Advanced Error Reporting off. Hides useful dmesg output for hardware debugging with no measurable boot-time win.
- `processor.max_cstate=1` — forces CPU out of deep sleep. Terrible for battery; use only to verify a C-state-related bug is the cause of something, then remove.

## 3.1.4 Boot-time measurement

Baseline before kernel params:

```bash
systemd-analyze
# Startup finished in X.Xs (kernel) + X.Xs (initrd) + X.Xs (userspace) = X.Xs
# graphical.target reached after X.Xs in userspace
```

After rebooting with the new params, re-run and compare. Expected wins:

- Kernel time: -0.5 to -1 s (fewer firmware probes wait for NVMe).
- Userspace time: ~unchanged.
- Total: -0.5 to -1.5 s.

Not huge. But the stability wins (Wayland + NVIDIA, NVMe) are the real payoff.

## 3.1.5 Trade the `quiet splash` for verbose boot

If you ever need to see what's actually happening during boot (diagnosis), remove `quiet splash` temporarily:

```bash
# Press 'e' at the GRUB menu, edit the `linux ...` line to remove "quiet splash",
# press F10 to boot with the edit (one-time, no save needed).
```

Or permanently while debugging:

```bash
sudoedit /etc/default/grub
# Remove "quiet splash" from GRUB_CMDLINE_LINUX_DEFAULT
sudo update-grub
```

Put them back after the bug is found.

## 3.1.6 Reboot and verify

```bash
sudo update-grub  # reaffirm
systemctl reboot
```

After reboot:

```bash
cat /proc/cmdline
# Expect the line you wrote.

systemd-analyze
# Total time; compare to pre-reboot baseline in ~/tuning-baseline-<date>.txt.

dmesg | grep -Ei "(nvidia|nvme|amd_pstate|preempt)" | head -20
```

Take a snapshot:

```bash
sudo timeshift --create --comments "post-kernel-params $(date -I)" --tags D
```

Proceed to [3.2 zram, swap, and sysctl](02-zram-swap-sysctl.md).

---

[Home](../README.md) · [↑ 03 Speed & Boot](README.md) · [← Previous: 03 Speed & Boot (section)](README.md) · **3.1 Kernel boot parameters** · [Next: 3.2 zram & sysctl →](02-zram-swap-sysctl.md)
