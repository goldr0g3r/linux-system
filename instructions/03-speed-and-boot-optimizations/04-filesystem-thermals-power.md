[Home](../README.md) · [↑ 03 Speed & Boot](README.md) · [← Previous: 3.3 Service masking](03-service-masking-and-journald.md) · **3.4 Filesystem, thermals, power** · [Next: 04 Daily Driver →](../04-daily-driver-stack/README.md)

---

# 3.4 Filesystem Mount Options, fstrim, Thermals, and Power Profiles

The final optimization page: BTRFS mount options that compound across every filesystem operation, SSD maintenance via fstrim, and the power-profile / thermal policy that matters once hardware is in steady state.

## Snapshot first

```bash
sudo timeshift --create --comments "before-filesystem-tuning $(date -I)" --tags D
```

## 3.4.1 BTRFS mount options

Inspect current mount:

```bash
findmnt /
# TARGET SOURCE           FSTYPE OPTIONS
# /      /dev/nvme1n1p3[/@] btrfs rw,relatime,space_cache=v2,subvol=@
```

The Ubuntu installer gives you `rw,relatime,space_cache=v2,subvol=@`. Three things are missing that we want:

- `compress=zstd:1` — transparent compression, ~30 % disk saving on a typical mix of source code + logs + binaries at a single-digit-% CPU cost.
- `noatime` — don't update the last-access time on every file read. Significantly reduces write amplification on the SSD, especially for `find`-heavy workflows (IDE indexing, Distrobox container `ls -la` loops).
- `discard=async` — trim deleted blocks in the background (kernel-async), complementing the weekly `fstrim.timer`.

Edit `/etc/fstab`:

```bash
sudoedit /etc/fstab
```

Find the `/` entry and the other BTRFS subvolume entries (`/home`, and if created, `/.snapshots`, `/var/log`, `/var/cache`). Change each options field:

**Before:**
```
UUID=<uuid>  /           btrfs  defaults,subvol=@       0 0
UUID=<uuid>  /home       btrfs  defaults,subvol=@home   0 0
```

**After:**
```
UUID=<uuid>  /           btrfs  noatime,compress=zstd:1,space_cache=v2,discard=async,subvol=@       0 0
UUID=<uuid>  /home       btrfs  noatime,compress=zstd:1,space_cache=v2,discard=async,subvol=@home   0 0
```

For `@snapshots`, `@var-log`, `@var-cache` — same options.

Apply without rebooting:

```bash
sudo mount -o remount,noatime,compress=zstd:1,discard=async /
sudo mount -o remount,noatime,compress=zstd:1,discard=async /home
# Repeat for other BTRFS subvolumes.

# Verify:
findmnt /
# New mount options visible.
```

### Retroactively compress existing files

`compress=zstd:1` applies to new writes. Existing files stay uncompressed until modified. Force compression on existing files:

```bash
sudo btrfs filesystem defragment -rvf -czstd /
# This is a long operation (10-30 min on a full root) that rewrites every
# file with zstd:1 compression. Run it once after enabling compression,
# then BTRFS keeps new files compressed automatically.
```

Measure the savings:

```bash
sudo compsize /
# Expect: Type Perc Disk Usage Uncompressed Referenced
#         none 30 %
#         zstd 70 % ...
# Depending on your data, the "Disk Usage" column is typically 60-75 % of Uncompressed.
```

### `zstd:1` vs `zstd:3` vs `zstd:15`

BTRFS compression levels 1-15. Default is 3. Higher = better compression, more CPU.

- **`zstd:1`** (recommended for desktop): minimal CPU cost, ~20-30 % savings. Imperceptible overhead.
- **`zstd:3`** (default if you just say `compress=zstd`): slightly better ratio, 2-3x CPU.
- **`zstd:15`**: archival. Don't use on a root filesystem.

On a modern Ryzen, `zstd:1` is fast enough that it often makes the filesystem *faster* overall: the compressed data transfers less, and the CPU cost of decompression is less than the SSD transfer-time saved.

## 3.4.2 fstrim

`fstrim` tells the SSD controller which blocks are unused, so it can reclaim them for wear leveling. Ubuntu enables `fstrim.timer` by default (weekly). Verify:

```bash
systemctl is-enabled fstrim.timer
# Expect: enabled

systemctl list-timers fstrim.timer
# Shows next scheduled run. Weekly.
```

Between the weekly timer and the `discard=async` mount option (above), TRIM is handled without any effort from you.

### Don't use both `discard` and `discard=async`

Older `/etc/fstab` entries sometimes have `discard` (synchronous TRIM, bad for performance on older SSDs). Replace with `discard=async`.

## 3.4.3 Enable BTRFS scrub timer

`btrfs scrub` reads every block, verifies checksums, and repairs corrupt blocks from redundant metadata. It's the BTRFS equivalent of `ZFS scrub`. A monthly scrub is cheap insurance against silent bitrot.

```bash
# Create a monthly timer:
sudo tee /etc/systemd/system/btrfs-scrub.service >/dev/null <<'EOF'
[Unit]
Description=BTRFS scrub of root filesystem
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/btrfs scrub start -B /
EOF

sudo tee /etc/systemd/system/btrfs-scrub.timer >/dev/null <<'EOF'
[Unit]
Description=Run BTRFS scrub monthly

[Timer]
OnCalendar=monthly
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now btrfs-scrub.timer
systemctl list-timers btrfs-scrub.timer
```

Scrub takes 15-30 min on a ~300 GB BTRFS; runs in background, minor IO impact.

## 3.4.4 Thermals: verify stock behaviour

On a TUF A17 with `asusctl` installed, temperature management is already handled. But confirm idle temperatures are sane:

```bash
# Install lm-sensors if not present:
sudo apt install -y lm-sensors

# Detect sensors:
sudo sensors-detect --auto

# Read:
sensors
```

Expected readings at idle on battery, Quiet profile, screen at 50 %:

- `k10temp-pci-*` (CPU): Tctl between 35–50 °C.
- `nvme-pci-*`: 30–45 °C.
- `amdgpu-pci-*` (iGPU): 30–45 °C.
- NVIDIA dGPU (if Hybrid + awake): 35–50 °C (via `nvidia-smi`).

If CPU idles above 60 °C, check:

- `asusctl profile -p` is Quiet or Balanced.
- `supergfxctl -g` is Hybrid or Integrated (NOT AsusMuxDgpu).
- No runaway process: `top -bn1 | head -15`.

## 3.4.5 Power profile daemon — the choice

26.04 ships `power-profiles-daemon` (PPD) as the default power manager. It has three modes:

- **power-saver** — conservative frequency, aggressive idle.
- **balanced** (default) — middle ground.
- **performance** — max frequency, shallow idle.

`asusctl` on 26.04 automatically maps its profiles (Quiet / Balanced / Performance) to PPD's modes. So toggling via `asusctl` or the Plasma battery applet both change the same thing.

### Why PPD and not TLP

TLP is the more-classical laptop power manager with hundreds of tunable knobs in `/etc/tlp.conf`. It is very effective but:

1. **Overlaps with `asusctl`**. The TUF platform-specific bits (fan curves, battery limit) are already `asusctl`'s job. TLP wants to manage CPU governor, which on 26.04 is `amd-pstate`'s job.
2. **Older project**. PPD is the Freedesktop-blessed modern answer; KDE / GNOME / Plasma all integrate with PPD directly via D-Bus.
3. **Configuration complexity**. TLP default config is 500 lines. PPD has 3 modes. For a dev laptop the PPD defaults + `asusctl` are enough.

**Recommendation:** use PPD (default on 26.04). Do not install TLP. If you have a specific TLP tweak you miss, replicate via a tiny systemd one-shot or `asusctl fan-curve`.

### Verify PPD is running

```bash
powerprofilesctl get
# Expect: balanced (or whatever you last set).

powerprofilesctl list
# Shows available profiles; TUF A17 always has all three.

# Plasma applet in system tray → battery icon → dropdown shows profiles. Click to change.
```

## 3.4.6 Screen brightness on battery

By default, when going on battery, Plasma dims the screen to 70 %. Tune:

System Settings → **Energy Management** → **On battery** tab:

- **Dim screen after 1 min idle (while on battery)**: 2 min is more comfortable.
- **Turn off screen after 5 min idle (while on battery)**: reasonable.
- **Sleep after 10 min idle**: set to "Never" if you hate it; accept 10 min if you're phone-call-prone.
- **Button events → Lid close action**: Sleep.

## 3.4.7 Hibernate vs suspend

- **Suspend to RAM**: fast (< 2 s in/out), consumes ~0.3-1 W, battery lasts 1-3 days suspended. Default for lid close.
- **Hibernate to disk**: slow (~15 s in, ~30 s out), consumes 0 W while off, battery lasts indefinitely. Requires hibernation image to fit in swap.

We provisioned 20 GB of swap; RAM is 16 GB; hibernation is feasible. But wiring hibernation up correctly requires:

1. Adding `resume=UUID=<swap-uuid>` to the kernel cmdline.
2. Adding `resume_offset=<offset>` to cmdline (only if using a swapfile, not a swap partition; `filefrag -v /swap/swapfile | head -5`).
3. Regenerating the initramfs.

On TUF A17 with NVIDIA on Wayland, hibernation's resume path is still a bit flaky (~1 in 10 chance of a stuck resume). Not recommended for daily use unless you really need it. If you want it anyway:

```bash
# Only applicable if you used Option A (raw swap partition) in 3.2.2:
SWAP_UUID=$(sudo blkid -s UUID -o value /dev/nvme1n1p4)
sudoedit /etc/default/grub
# Append: resume=UUID=${SWAP_UUID}
sudo update-grub
sudo update-initramfs -u
# Test: systemctl hibernate
```

For swapfile (Option B), add also `resume_offset=$(sudo filefrag -v /swap/swapfile | awk '/ 0:/{gsub("\\.\\.",""); print $4; exit}')`.

## 3.4.8 Final snapshot

```bash
sudo timeshift --create --comments "post-optimizations $(date -I)" --tags D
```

This is the "post-all-optimizations" state. Good rollback target.

Compare totals vs baseline:

```bash
systemd-analyze
# Should be 1-3 s faster to graphical.target than the pre-3.1 baseline.

free -h
# Swap shows both zram and disk swap, total ~28 GB.

sudo compsize /
# Compression ratio ~1.4x, disk usage noticeably less than uncompressed.

sensors
# Idle temps 35-50 °C.
```

Section 03 complete. Proceed to [04 Daily Driver Stack](../04-daily-driver-stack/README.md).

---

[Home](../README.md) · [↑ 03 Speed & Boot](README.md) · [← Previous: 3.3 Service masking](03-service-masking-and-journald.md) · **3.4 Filesystem, thermals, power** · [Next: 04 Daily Driver →](../04-daily-driver-stack/README.md)
