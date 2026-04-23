[Home](../README.md) · [↑ 01 Install](README.md) · [← Previous: 1.4 Partitioning](04-installation-and-partitioning.md) · **1.5 First boot & snapshot** · [Next: 02 Post-install →](../02-post-install-foundations/README.md)

---

# 1.5 First Boot, Updates, and Initial Timeshift Snapshot

Your Kubuntu 26.04 install is up. This page covers the 15 minutes between "SDDM appeared" and "I have a named Timeshift snapshot I can roll back to." After this page, everything in [02-post-install-foundations/](../02-post-install-foundations/README.md) is a forward step with a known-good fallback.

## 1.5.1 First login

1. Enter your username and password.
2. **Pick the `Plasma (Wayland)` session** in the SDDM session selector (small icon bottom-left). On 26.04 this is the default and is what you want. The X11 session is not installed; if it appears in the selector, that is a bug and the entry is inert.
3. You land on an empty Plasma 6.6 desktop with a KRunner-style application launcher in the system tray.

### What should already work

- Wi-Fi: the `plasma-nm` applet should show your network. Connect.
- Battery icon: should show percentage and plug-in state.
- Sound: the volume tray icon should be visible. Test with the KDE login sound (if enabled) or by opening Dolphin's "about" and toggling sound effects.
- Bluetooth: the `bluedevilmonolithic` applet should appear. Do not pair the headset yet — [4.3](../04-daily-driver-stack/03-bluetooth-headset-hifi.md) covers codec priority which you want set before the first pair.
- Touchpad and keyboard: all keys. Brightness and volume hotkeys may or may not work — TUF A17's `Fn+F7/F8` brightness keys depend on `asusctl` which comes in [2.3](../02-post-install-foundations/03-asus-tuf-hybrid-gpu.md).

### What will NOT work yet

- NVIDIA / CUDA / `nvidia-smi` — driver comes in [2.2](../02-post-install-foundations/02-nvidia-driver-580-and-cuda.md).
- `asusctl` / fan curves / battery cap — in [2.3](../02-post-install-foundations/03-asus-tuf-hybrid-gpu.md).
- Snapshots (`timeshift` itself is not installed by Minimal; we install it below).

## 1.5.2 Verify dual-boot

First and most important sanity check: cold-reboot and confirm you can reach Windows.

```bash
systemctl reboot
# At POST, mash F8.
# Pick "Windows Boot Manager".
# Confirm Windows desktop appears.
# Confirm BitLocker did not demand the recovery key.
# Shut down Windows (clean — Start → Power → Shut down, NOT Restart).
# Boot Kubuntu again via F8.
```

If either OS fails to appear in F8, return to [1.2.10 Boot Option #1](02-windows-prep-and-bios.md#1210-boot-option-1--keep-windows-for-now) or [1.4.10 Choose bootloader location](04-installation-and-partitioning.md#1410-choose-bootloader-location) and re-verify.

If BitLocker demanded the recovery key: enter it (you saved it in [1.2.2](02-windows-prep-and-bios.md#122-suspend-bitlocker-if-enabled)); log into Windows; re-suspend BitLocker (`manage-bde -protectors -disable C: -RebootCount 3`); shut down; come back.

## 1.5.3 Run the first system update

The installer ran with ISO-version packages. Update everything.

```bash
sudo apt update
sudo apt full-upgrade -y
```

Expect:

- Kernel might upgrade from 7.0.x-1 to 7.0.x-2 (point release). **If it does, reboot before proceeding to [2.2](../02-post-install-foundations/02-nvidia-driver-580-and-cuda.md)** — NVIDIA's DKMS will build against whatever kernel is booted, so you want the new kernel live before installing NVIDIA.
- `sudo-rs` configuration update. It will show a `configuration file ... sudoers.d / has been modified` prompt. Pick **keep the package maintainer's version** (default).
- ~300 MB download, ~5 minutes.

Reboot if the kernel upgraded:

```bash
uname -r  # note current kernel
sudo apt list --upgradable 2>/dev/null | grep linux-image  # blank if nothing to upgrade
# If you installed a kernel upgrade:
systemctl reboot
```

After reboot, `uname -r` should show the new kernel.

## 1.5.4 Install Timeshift and core utilities for snapshots

Timeshift is not in the Minimal install. Install it and a couple of companions:

```bash
sudo apt install -y timeshift btrfs-compsize btrfs-assistant
```

- **`timeshift`** — the snapshot tool itself.
- **`btrfs-compsize`** — shows actual compressed size per file/dir (useful later to tune `compress=zstd:1` vs `zstd:3`).
- **`btrfs-assistant`** — optional GUI for Snapper/Timeshift; replaces staring at `timeshift-gtk` when you want a quick filesystem overview.

## 1.5.5 Configure Timeshift for BTRFS mode

Timeshift has two modes: RSYNC (copies files) and BTRFS (uses subvolume snapshots). **Pick BTRFS** — it's atomic, per-snapshot-cost is O(changed-blocks), and a rollback is a one-line `mv` instead of a multi-gigabyte copy.

### Interactive setup (GUI)

Launch **Timeshift** (KRunner → "timeshift") → follow the wizard:

1. **Snapshot Type**: **BTRFS**.
2. **Snapshot Location**: the BTRFS-mounted root filesystem (it auto-detects).
3. **Snapshot levels**: pick **Monthly**, **Weekly**, **Daily**. **Uncheck Hourly and Boot**. For a dev laptop, Hourly snapshots either waste disk on tiny-delta changes or create a false sense of "I have recent backups" when really you have 10 MB of text changes replicated 24 times a day. Daily is the right cadence for this machine.
4. **Number of snapshots to keep**: 5 monthly, 5 weekly, 7 daily.
5. **User Home Directories**: **Exclude All** (default). Your `~` is big (media, caches, containers); Timeshift should snapshot **the system**, not your user data. User data is backed up separately via Restic in [8.2](../08-productivity-security/02-backup-strategy-3-2-1.md).
6. Finish wizard.

### Or, via command-line / config file

Edit `/etc/timeshift/timeshift.json`:

```json
{
  "backup_device_uuid": "<paste the UUID of /dev/nvme1n1p3 from `sudo blkid`>",
  "btrfs_mode": "true",
  "include_btrfs_home_for_backup": "false",
  "schedule_monthly": "true",
  "schedule_weekly": "true",
  "schedule_daily": "true",
  "schedule_hourly": "false",
  "schedule_boot": "false",
  "count_monthly": "5",
  "count_weekly": "5",
  "count_daily": "7",
  "count_hourly": "0",
  "count_boot": "0",
  "exclude": [
    "+ /root/**",
    "/home/**",
    "/var/log/**",
    "/var/cache/**",
    "/var/tmp/**"
  ]
}
```

Apply:

```bash
sudo systemctl enable --now cron.service  # timeshift's automation uses cron
```

## 1.5.6 Take the FIRST named snapshot

This snapshot is the reference "fresh install, post apt full-upgrade, before any customisation" state. It's what you roll back to if the next 90 minutes break something in ways you can't un-break.

```bash
sudo timeshift --create --comments "fresh-install-post-upgrade $(date -I)" --tags D
timeshift --list
```

Expect one entry: today's date, with your comment, type `O` (On-Demand) because we invoked explicitly. Verify its size (should be ~4 GB — the actual installed system, because BTRFS shares blocks; subsequent snapshots will be tiny).

## 1.5.7 Pre-apt snapshot hook

Install the `snapshot-and-continue` convention that we'll reuse through this guide: a wrapper function that takes a Timeshift snapshot before `apt install` / `apt upgrade`.

Create `/usr/local/bin/apt-with-snapshot`:

```bash
sudo tee /usr/local/bin/apt-with-snapshot >/dev/null <<'EOF'
#!/usr/bin/env bash
# Take a timeshift snapshot tagged with the command, then run apt.
set -euo pipefail
cmd="apt $*"
tag="before: $cmd"
sudo timeshift --create --comments "$tag" --tags D >&2
sudo apt "$@"
EOF
sudo chmod +x /usr/local/bin/apt-with-snapshot
```

Usage:

```bash
apt-with-snapshot full-upgrade -y
apt-with-snapshot install some-risky-package
```

Not a replacement for plain `apt` (use `apt-with-snapshot` only when you actually want a snapshot), but convenient for the next couple of hours where you will be making large changes.

## 1.5.8 Verify the system's general posture

```bash
# Kernel:
uname -r
# Expect: 7.0.x-generic (or higher)

# Distro:
lsb_release -a
# Expect: Kubuntu 26.04 LTS / Ubuntu 26.04 LTS / Resolute Raccoon

# Desktop:
echo $XDG_SESSION_TYPE  # wayland
echo $XDG_CURRENT_DESKTOP  # KDE
kinfocenter &
# Expect: Plasma 6.6.x · Qt 6.10.2 · Frameworks 6.24.0

# Firmware updates available:
sudo fwupdmgr refresh
sudo fwupdmgr get-updates

# Dual-boot sanity:
efibootmgr -v
# Expect: entries for "ubuntu" (or "Kubuntu"), "Windows Boot Manager", maybe a USB
# The one at BootOrder's first position is what F8's default is.

# BTRFS subvolumes:
sudo btrfs subvolume list /
# Expect: @, @home, and (if you did 1.4.12) @snapshots, @var-log, @var-cache

# Timeshift is configured:
timeshift --list
# One snapshot, with today's date.
```

If any of those unexpected, stop and resolve before proceeding to [2.1](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md). The rest of the guide assumes Wayland, kernel 7.0+, BTRFS root, and working Timeshift.

## 1.5.9 Before you move on

- Browser: Discover is open; do not install anything via Discover yet (it uses Flatpak + snap by default and we are about to nuke snap in [2.1](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md)).
- SSH: `ssh` is available as a client; there is no `sshd` running (good — laptop should not expose SSH).
- Git: `git --version` should return ~2.47+. Identity setup lives in [5.2](../05-web-development/02-git-ssh-gpg.md).

You now have a clean, snapshot-protected Kubuntu 26.04 install. Next stop: [2.1 Debloat](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md).

---

[Home](../README.md) · [↑ 01 Install](README.md) · [← Previous: 1.4 Partitioning](04-installation-and-partitioning.md) · **1.5 First boot & snapshot** · [Next: 02 Post-install →](../02-post-install-foundations/README.md)
