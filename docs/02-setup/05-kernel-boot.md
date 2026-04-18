[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.4 Debloat](04-debloat.md) · **2.5 Kernel & Boot** · [Next: 2.6 ASUS Hardware →](06-asus-hardware.md)

---

# 2.5 Kernel & Boot Optimization

Shave 3–6 seconds off cold boot, tune memory behaviour for compilation and AI workloads, and wire up Timeshift for BTRFS snapshots before you start breaking things in [Part 3](../03-dev-environment/README.md).

## One block

```bash
# Mask services that add 1–4 seconds each to boot, none of which you need on a dev laptop.
sudo systemctl mask \
  NetworkManager-wait-online.service \
  plymouth-quit-wait.service \
  motd-news.service \
  motd-news.timer \
  ubuntu-advantage.service \
  apt-news.service \
  esm-cache.service

# zram: compressed in-memory swap. Use this alongside (not instead of) the BTRFS swapfile.
sudo apt install -y systemd-zram-generator
sudo tee /etc/systemd/zram-generator.conf >/dev/null <<'EOF'
[zram0]
zram-size = min(ram / 2, 8192)
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
EOF

# Tune swappiness so the kernel prefers zram over the BTRFS swapfile.
sudo tee /etc/sysctl.d/99-engineer.conf >/dev/null <<'EOF'
vm.swappiness = 30
vm.vfs_cache_pressure = 50
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15
kernel.sysrq = 1
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
EOF
sudo sysctl --system

# Enable weekly fstrim for the NVMe.
sudo systemctl enable --now fstrim.timer

# Install a newer kernel (HWE) for better AMD Ryzen 4800H thermal/freq handling.
sudo apt install -y linux-generic-hwe-24.04

# Timeshift (root filesystem snapshots, BTRFS-aware).
sudo apt install -y timeshift
# Open timeshift from the menu; pick BTRFS mode; include /home optionally; schedule weekly + before updates.

sudo reboot
```

## What each part does

### Service masking (lines 2–9)

| Service | Cost | Why mask |
|---------|------|----------|
| `NetworkManager-wait-online.service` | 2–30 s blocking wait | Blocks boot until NM reports "online"; we do not need any network-at-boot dependencies. |
| `plymouth-quit-wait.service` | ~1 s | Waits for the splash to finish animating. Kill it. |
| `motd-news.{service,timer}` | fetches ad content | Downloads Ubuntu "news" for the MOTD. Disable it. |
| `ubuntu-advantage.service` | ~0.5 s | Ubuntu Pro client; re-enable if you activate Pro later. |
| `apt-news.service`, `esm-cache.service` | noise | Ubuntu Pro ads surfacing in `apt upgrade` output. |

Together these save 3–6 seconds of boot time and remove all "ubuntu advantage" / "ubuntu pro" nagging.

### zram (lines 11–19)

`zram` is compressed RAM used as swap. Writing to it is much faster than writing to the BTRFS swapfile because there is no IO. Configuration:

- `zram-size = min(ram / 2, 8192)` — on 16 GB RAM, creates an 8 GB zram device.
- `compression-algorithm = zstd` — fast, high-ratio; typically 3:1 or better on code/JSON/text.
- `swap-priority = 100` — higher than the BTRFS swapfile (priority -2 by default), so kernel uses zram first.

With this, your effective memory is ~22 GB (16 GB RAM + 8 GB zram acting as ~24 GB of RAM-equivalent). PyTorch fits, Chromium fits, Gazebo fits.

### sysctl tuning (lines 21–30)

| Param | Value | Why |
|-------|-------|-----|
| `vm.swappiness = 30` | Default 60 | Kernel prefers RAM over swap; important because swap-to-zram is fast but swap-to-BTRFS is not. |
| `vm.vfs_cache_pressure = 50` | Default 100 | Keep more filesystem metadata in cache; helps IDE file-open latency. |
| `vm.dirty_background_ratio = 5` | Default 10 | Start writing dirty pages earlier (smaller bursts). |
| `vm.dirty_ratio = 15` | Default 20 | Cap dirty pages before blocking writes. Reduces hitches under heavy write load. |
| `kernel.sysrq = 1` | Default 176 | Enable all SysRq keys. Useful for hard-hang recovery (Alt+SysRq+REISUB). |
| `net.core.default_qdisc = fq` | | Pair with BBR; FQ reduces bufferbloat. |
| `net.ipv4.tcp_congestion_control = bbr` | Default `cubic` | Google's BBR is faster on typical broadband; noticeable on `apt` / `git clone`. |

### fstrim (line 32)

`fstrim.timer` runs `fstrim` weekly across all mounted filesystems, reclaiming SSD cells marked deleted. Essential for long-term SSD health; irresponsible to leave off. Note: we already set `discard=async` in fstab, so trim happens incrementally too — this timer is belt-and-braces.

### HWE kernel (line 35)

`linux-generic-hwe-24.04` follows the HWE track — pulls in newer kernels during the LTS cycle. As of Ubuntu 24.04.4 LTS (Feb 2026) the HWE kernel is **6.17**, which brings SmartMux support for AMD hybrid laptops (relevant to your TUF A17 dGPU idle), better `amd_pstate` on Zen 2, RDNA 4 support (forward-looking), and improved Wi-Fi 7. Another HWE bump to kernel 6.20/7.0 is expected in the 24.04.5 point release (Aug 2026). Almost always a net win — install it unless you have a specific reason to stay on the 6.8 GA kernel.

### Timeshift (lines 38–40)

After reboot, open Timeshift from the application menu:

- Snapshot type: **BTRFS** (not rsync).
- Schedule: **Weekly**, keep **4 weekly + 3 on-demand**.
- Include `/home`: optional — if enabled, your home dir is in every snapshot, which makes them big; if disabled, snapshots are tiny and fast, but you lose user-file history. Recommend **disabled**, use `syncthing` or `git` for home-dir history.
- Fire a manual snapshot now named `fresh-install-baseline`.

**Take a snapshot before every risky change** (NVIDIA upgrade, kernel upgrade, testing a PPA). `sudo timeshift --create --comments "pre-<thing>"`.

## Verify after reboot

```bash
# zram live?
zramctl
# Expect: /dev/zram0 with ~8G size, zstd compression

# Swap priorities correct?
swapon --show
# Expect: /dev/zram0 priority 100, /swap/swapfile priority -2

# Boot time breakdown:
systemd-analyze
systemd-analyze blame | head -20
# Expect: total under 15 seconds on NVMe

# sysctl values applied?
sysctl vm.swappiness net.ipv4.tcp_congestion_control
# Expect: 30, bbr
```

Proceed to [2.6 ASUS TUF A17 hardware enablement](06-asus-hardware.md).

---

[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.4 Debloat](04-debloat.md) · **2.5 Kernel & Boot** · [Next: 2.6 ASUS Hardware →](06-asus-hardware.md)
