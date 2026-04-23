[Home](../README.md) · [↑ 03 Speed & Boot](README.md) · [← Previous: 3.1 Kernel params](01-kernel-boot-parameters.md) · **3.2 zram, swap, and sysctl** · [Next: 3.3 Service masking →](03-service-masking-and-journald.md)

---

# 3.2 zram, Swap, and sysctl Tuning

Three related topics: compressed RAM swap (`zram`), the 20 GiB BTRFS swapfile we provisioned in [1.4](../01-install/04-installation-and-partitioning.md), and the handful of `sysctl` knobs that matter for responsiveness.

## Why this matters

Your TUF A17 has 16 GB of RAM. Modern workflow — browser with 30 tabs, VS Code, Konsole with several panes, a Distrobox container running, and an Ollama model loaded — routinely touches 12–14 GB of committed memory. On a naive Linux install, once you exceed physical RAM the system starts paging to the swap partition on NVMe, which is ~10x slower than RAM and noticeable to the user.

`zram` is a compressed block device backed by RAM. You give it some RAM budget (say 50 % of physical = 8 GB), and it presents itself as a ~24 GB compressed swap. When the kernel needs to page something out, it goes to zram first (RAM → compressed RAM, essentially free), and only falls through to on-disk swap if zram is full. With `zstd:1` compression, typical text-heavy desktop working sets compress ~3:1, effectively extending the 16 GB machine to act like ~24 GB of "real" RAM for most workloads.

## 2026 defaults on Kubuntu

Kubuntu 26.04 pre-installs `zram-tools` in the Minimal install and enables a default `/dev/zram0` at 50 % of physical RAM using `lz4` compression. This is already better than 24.04 (which had no zram by default). Our tuning: swap `lz4` for `zstd` (better compression, negligibly slower), and configure the disk swapfile.

## Snapshot first

```bash
sudo timeshift --create --comments "before-zram-sysctl $(date -I)" --tags D
```

## 3.2.1 Configure zram with zstd

Edit `/etc/default/zramswap`:

```bash
sudoedit /etc/default/zramswap
```

Set:

```text
# zram compressed swap — this is what you're tuning.
# Algorithm: zstd is ~50 % better ratio than lz4 at ~2x CPU cost.
# Priority: 100 is above disk swap (which defaults to 0), so kernel uses zram first.
ALGO=zstd
PERCENT=50
PRIORITY=100
```

Explanation:

- `ALGO=zstd` — on modern Ryzen the CPU cost of zstd vs lz4 is a rounding error (single-thread, zram compresses in the kernel context, so it doesn't tie up your user cores). The compression ratio win is tangible.
- `PERCENT=50` — 8 GB of RAM reserved for zram, which at ~3:1 compression holds ~24 GB of compressed data. Total "effective swap" = 24 GB zram + 20 GB disk = 44 GB. Plenty.
- `PRIORITY=100` — swap priority (kernel uses highest-priority swap first). Default disk swap is 0, so zram wins.

Apply:

```bash
sudo systemctl restart zramswap.service

# Verify:
swapon --show
# Expect both /dev/zram0 and (after 3.2.2) the disk swapfile with their priorities.

# zram status:
zramctl
# Expect /dev/zram0, algorithm zstd, disksize ~8G, data 0B, compr 0B, total ~0 (until pressure).
```

## 3.2.2 Convert the 20 GiB partition to a swapfile

Recall that in [1.4.9](../01-install/04-installation-and-partitioning.md#149-create-a-btrfs-placeholder-swap-partition) we reserved 20 GiB as an unformatted BTRFS partition. Now we convert that reserved space into a BTRFS swapfile inside the root filesystem — cleaner than using a raw swap partition, and BTRFS swapfiles have been stable since kernel 5.8.

Actually, if you left it as a second BTRFS partition, the simplest path is to reformat it as a real Linux swap partition (not a BTRFS file). Either works. Pick one:

### Option A (recommended): make the second partition a raw Linux swap

```bash
# Identify the partition (should be /dev/nvme1n1p4):
lsblk -f /dev/nvme1n1

# Make it swap:
sudo mkswap /dev/nvme1n1p4

# Grab its UUID:
SWAP_UUID=$(sudo blkid -s UUID -o value /dev/nvme1n1p4)
echo "$SWAP_UUID"

# Add to /etc/fstab:
echo "UUID=${SWAP_UUID} none swap sw,pri=10 0 0" | sudo tee -a /etc/fstab

# Enable:
sudo swapon -a
swapon --show
```

The `pri=10` sets priority to 10 (below zram's 100, above default 0). Kernel uses zram until full, then falls through to this partition.

### Option B: swapfile on the BTRFS root subvolume

Use this if you prefer to reclaim the 20 GiB partition for extra BTRFS capacity. Requires a few extra steps because BTRFS swapfiles need `nodatacow`.

```bash
# Create an empty file on BTRFS with no CoW:
sudo btrfs subvolume create /swap
sudo chattr +C /swap  # no-CoW attribute — MUST be set before any data is written
sudo fallocate -l 20G /swap/swapfile
sudo chmod 600 /swap/swapfile
sudo mkswap /swap/swapfile

# Add to /etc/fstab:
echo "/swap/swapfile none swap sw,pri=10 0 0" | sudo tee -a /etc/fstab

# Enable:
sudo swapon -a

# The old 20GiB partition (/dev/nvme1n1p4) is now unused — you could delete it via
# partitionmanager and grow the BTRFS partition into the freed space. Not mandatory.
```

Either option works; option A is simpler.

## 3.2.3 sysctl tuning

A handful of sysctl knobs help responsiveness on a 16 GB dev laptop. Create `/etc/sysctl.d/99-local-tuning.conf`:

```bash
sudo tee /etc/sysctl.d/99-local-tuning.conf >/dev/null <<'EOF'
# ------------------------------------------------------------------------
# Kubuntu 26.04 local tuning
# 16 GB RAM dev laptop with zram (50 %, zstd) + 20 GB disk swap
# ------------------------------------------------------------------------

# Swappiness: how aggressively the kernel will swap out anonymous pages.
# Default is 60 (server-oriented). With zram, lower-than-default is sensible
# but not zero — we want the kernel to proactively use zram (compressed RAM)
# rather than hold under-used pages in uncompressed RAM.
# 10 is the right balance for 16 GB + 50 % zram.
vm.swappiness = 10

# VFS cache pressure: how aggressively the kernel reclaims inode/dentry cache.
# Lower = keeps cache longer (more memory used for cache, faster file ops).
# 50 is a good balance for 16 GB; default of 100 is overly aggressive for desktops.
vm.vfs_cache_pressure = 50

# Dirty ratio: how much RAM can be filled with dirty (unwritten) pages
# before processes are forced to block on writeback.
# 10 % of 16 GB = 1.6 GB — reasonable for NVMe.
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5

# Dirty expire: how old a dirty page can be before writeback starts.
# 15 s instead of default 30 s — smooths writeback spikes.
vm.dirty_expire_centisecs = 1500
vm.dirty_writeback_centisecs = 1500

# Autogroup: per-session CPU scheduling fairness.
# Default-enabled on modern kernels; explicit for clarity.
kernel.sched_autogroup_enabled = 1

# Allow more file descriptors per process (for Node.js / Electron apps):
fs.file-max = 2097152

# inotify — VS Code/editors watch large trees; default limits are too small.
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 8192

# Network: faster small-packet handling + larger buffers for ~1 Gbit Wi-Fi
# and decent loopback (Podman, localhost HTTP).
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_fastopen = 3
EOF

# Apply:
sudo sysctl --system

# Verify:
sysctl vm.swappiness
# Expect: vm.swappiness = 10
```

## 3.2.4 Pressure Stall Information (PSI)-based OOM handling

Kernel 7.0 ships `systemd-oomd` enabled by default on 26.04, using PSI to detect memory pressure and kill the most memory-hungry cgroup when the system is thrashing. This is a significant improvement over the pre-PSI OOM killer.

Verify:

```bash
systemctl status systemd-oomd.service
# Expect: active (running)

# What it monitors:
systemctl show systemd-oomd.service --property=ManagedOOMMemoryPressureLimit
# Default: 80 %
```

If you want more-aggressive OOM (kill cgroups sooner on pressure), reduce the limit. Rarely needed; systemd-oomd defaults are sane.

## 3.2.5 Watchdog timer

On laptops, the hardware watchdog timer is usually ignorable. On a TUF A17 specifically, there are reports of the `iTCO_wdt` (Intel chipset watchdog) triggering spurious reboots. AMD equivalent `sp5100_tco` is usually clean, but if you hit random reboots:

```bash
# Disable the watchdog module:
sudo tee /etc/modprobe.d/blacklist-watchdog.conf >/dev/null <<'EOF'
blacklist sp5100_tco
blacklist iTCO_wdt
EOF

sudo update-initramfs -u
```

Not recommended by default (watchdog can recover a genuine kernel hang); apply only if diagnosing mystery reboots.

## 3.2.6 Verify

```bash
# All swap:
swapon --show
# Expect /dev/zram0 (priority 100, ~8 GB) and the disk swap (priority 10, 20 GB).

# Total:
free -h
# Expect: Swap: 28 GB (8 + 20) — zram and disk swap summed.

# Live sysctl values:
sysctl vm.swappiness vm.vfs_cache_pressure vm.dirty_ratio
# All match the configured values.

# Memory pressure:
cat /proc/pressure/memory
# Expect: some 0.00 0.00 0.00 values (no pressure on idle).

# OOMD health:
systemctl status systemd-oomd
# Active.
```

## 3.2.7 What you should NOT do

- **Disable swap entirely** — modern kernels assume swap exists; without it you lose access to a lot of page-migration paths (zswap, zram, oomd tuning).
- **Set `vm.swappiness = 0`** — common misconception from the 2010s; it doesn't mean "never swap", it means "only swap under extreme pressure", which kicks in *exactly* when you least want the latency.
- **Create an unencrypted hibernation swap** — hibernate writes the whole memory image to swap, so if you care about privacy on a laptop that can be stolen, either (a) don't hibernate or (b) LUKS the swap. We chose (a).
- **Use `zswap` instead of `zram`** — zswap is a compressed cache in front of disk swap; the win over plain swap is smaller than zram's. Pick one; on a 16 GB dev laptop, `zram` > `zswap`.

Take a snapshot:

```bash
sudo timeshift --create --comments "post-zram-sysctl $(date -I)" --tags D
```

Proceed to [3.3 Service masking and journald](03-service-masking-and-journald.md).

---

[Home](../README.md) · [↑ 03 Speed & Boot](README.md) · [← Previous: 3.1 Kernel params](01-kernel-boot-parameters.md) · **3.2 zram, swap, and sysctl** · [Next: 3.3 Service masking →](03-service-masking-and-journald.md)
