[Home](../README.md) · [↑ 02 Post-Install](README.md) · [← Previous: 2.4 Wayland + Plasma](04-wayland-plasma-66-nvidia.md) · **2.5 Snapshot policy** · [Next: 03 Speed & Boot →](../03-speed-and-boot-optimizations/README.md)

---

# 2.5 Snapshot Policy — BTRFS + Timeshift Rollback Discipline

The discipline that turns "snapshots" from "I took one once" into a muscle-memory operating pattern. This page formalises cadence, retention, coverage, and pre-change hooks.

## What Timeshift snapshots are and are not

They **are**:

- Near-instant, atomic, block-deduplicated points-in-time of your **system subvolumes** (`@`, and in our layout, the root filesystem).
- O(changed-blocks) to create and keep; taking 10 snapshots of an unchanged system costs megabytes.
- The correct tool to roll back "I just installed a bad package" or "I changed `/etc/something` and now X is broken" in ~30 seconds.

They **are not**:

- A backup. A BTRFS snapshot lives on the same filesystem. If the NVMe fails, if the filesystem metadata corrupts, or if `rm -rf /` in a malicious shell goes through, the snapshots go too. Real backups go to a different machine, via Restic, covered in [8.2](../08-productivity-security/02-backup-strategy-3-2-1.md).
- Coverage for `/home`, `/var/log`, or `/var/cache` in our layout — those are separate subvolumes deliberately excluded from root snapshots.

## 2.5.1 The pre-change hook pattern

Any time you are about to do something that could break the system, take a snapshot with a descriptive name. The wrapper in [1.5.7](../01-install/05-first-boot-and-snapshot.md#157-pre-apt-snapshot-hook) (`apt-with-snapshot`) does exactly this for `apt`. Extend the pattern:

```bash
# Before editing a critical config:
snap-config() {
  local what="$1"
  sudo timeshift --create --comments "before-config-$what $(date -I)" --tags D
}

# Usage:
snap-config "editing /etc/sysctl.conf"
sudoedit /etc/sysctl.conf
```

Add to `~/.zshrc` or `~/.bashrc`. This turns "I'll just edit this quickly" into a 10-second snapshot + edit sequence; the 10 seconds is free insurance.

## 2.5.2 Snapshot retention policy

The [1.5](../01-install/05-first-boot-and-snapshot.md) wizard set:

- **5 monthly** snapshots — ~5 months of calendar coverage.
- **5 weekly** — ~5 weeks of weekly-detail coverage.
- **7 daily** — ~1 week of per-day coverage.
- **0 hourly** and **0 boot** snapshots.
- **Plus ad-hoc `--create --tags D`** snapshots that do NOT count toward the schedule retention.

Rationale:

- **No hourly** — laptop is off 8h/day, snapshots of trivial changes waste IO and mental cost of grepping timestamps.
- **No boot** — uninteresting; almost no system state changes "between boots" that isn't captured by a daily.
- **7 daily, 5 weekly, 5 monthly** — you can reach back 5 months for "whenever that worked" diagnostics without the snapshot list becoming unreadable.

On ~512 GB with `zstd:1` compression and typical Ubuntu system churn, this retention uses ~5–15 GB of snapshot delta over 3 months.

## 2.5.3 Viewing snapshot state

```bash
# List snapshots:
sudo timeshift --list

# Verbose list with size estimates:
sudo btrfs subvolume list /.snapshots

# Disk usage per snapshot (slow — reads all blocks to compute exclusive size):
sudo btrfs qgroup show / 2>/dev/null
# (Requires quota groups enabled; see below if you want this)

# Fast: total compressed size of the snapshots subvolume:
sudo compsize /.snapshots
```

### Optional: enable BTRFS quotas for per-snapshot size tracking

```bash
sudo btrfs quota enable /
sudo btrfs qgroup show /
# Now `timeshift --list` shows real per-snapshot exclusive size.
```

Downside: quota enabled adds ~5 % CPU overhead on filesystem ops. I leave it off and accept that "delta size" is approximate.

## 2.5.4 Rolling back a snapshot

Two cases:

### Case 1: System is still bootable, you want to roll back `/`

```bash
# List:
sudo timeshift --list

# Restore (interactive; will show what changes and ask for confirmation):
sudo timeshift --restore --snapshot "2026-04-23_18-30-55"

# Reboot when told.
```

Timeshift's BTRFS restore is a subvolume-rename operation: the current `/` is renamed `@_backup`, the chosen snapshot becomes the new `@`, and GRUB/systemd-boot points at the new `@` on reboot. Takes ~5 seconds.

After reboot, verify:

```bash
sudo btrfs subvolume list / | head -5
# You should see @, @_backup (the pre-restore root), and the @@ snapshots underneath.
```

If the restore was wrong (you actually wanted a different snapshot), you can restore again from the `@_backup` subvolume or another snapshot. Timeshift keeps the `@_backup` around for exactly this.

### Case 2: System won't boot

Boot from the Kubuntu live USB → open Konsole → mount the BTRFS root → rename subvolumes manually.

```bash
# Identify Disk B's BTRFS partition:
sudo blkid | grep btrfs
# Note the UUID; usually /dev/nvme1n1p3.

sudo mkdir /mnt/recover
sudo mount /dev/nvme1n1p3 /mnt/recover -o subvolid=5

# Current state:
sudo btrfs subvolume list /mnt/recover

# Say the broken root is @ and a known-good snapshot is timeshift-btrfs/snapshots/2026-04-23_18-30-55/@.
# Rename:
sudo mv /mnt/recover/@ /mnt/recover/@-broken-$(date +%s)
sudo btrfs subvolume snapshot \
  /mnt/recover/timeshift-btrfs/snapshots/2026-04-23_18-30-55/@ \
  /mnt/recover/@

sudo umount /mnt/recover
sudo systemctl reboot
```

Remove the USB and boot normally.

## 2.5.5 Timeshift vs Snapper — a one-paragraph comparison

Timeshift and Snapper both do BTRFS snapshotting on Kubuntu 26.04. Timeshift is what this guide uses because: (a) the GUI is better for occasional users, (b) one-command create/restore is sufficient for a laptop, and (c) the default layout matches what this guide partitioned. Snapper's advantages — fine-grained pre/post-operation snapshots wired into `apt` via `apt-btrfs-snapshot`, zypper-like "see exactly what changed in this snapshot" — are real but less relevant here. If you have existing Snapper muscle memory from SUSE, `sudo apt install snapper snapper-gui`, follow [`man snapper`](https://manpages.ubuntu.com/manpages/resolute/man8/snapper.8.html), and the Timeshift config in [1.5](../01-install/05-first-boot-and-snapshot.md) is compatible-coexistent (separate subvolume lists; no collision).

## 2.5.6 The rollback ritual

Memorise this four-step ritual. When something breaks, execute without thinking:

1. **"Is the system booting?"**
   - **Yes:** `sudo timeshift --list` → pick the last known-good → `sudo timeshift --restore --snapshot <ID>` → reboot.
   - **No, but SDDM appears:** log in via TTY (`Ctrl+Alt+F2`) → same as above.
   - **No, not even SDDM:** boot USB live, follow [2.5.4 case 2](#case-2-system-wont-boot).

2. **While the system is running pre-restore, look for the root cause.**
   - `journalctl -b -p err` for this boot's errors.
   - `ls -lt /etc/apt/history.log` or `zcat /var/log/apt/history.log.1.gz` — what was installed/removed in the hour before breakage?
   - `dmesg -T | tail -50` — any kernel splat?

3. **After restore, verify recovery.**
   - `timeshift --list` shows you restored.
   - The thing that was broken, works.
   - Run your usual post-reboot sanity (`nvidia-smi`, `asusctl`, a browser tab).

4. **Learn.**
   - Add a line to a `~/rollback-log.md` file describing what broke and what you rolled back from. Over 5 years of LTS that file becomes a personal troubleshooting history.

## 2.5.7 When NOT to rely on snapshots

There are failure modes snapshots cannot recover from:

| Failure                                     | Snapshot can recover? | Use instead                  |
| ------------------------------------------- | --------------------- | ---------------------------- |
| Broken package install                      | Yes                   | `timeshift --restore`        |
| Bad `/etc/` edit                            | Yes                   | `timeshift --restore`        |
| Wayland session broken after KWin upgrade   | Yes                   | `timeshift --restore`        |
| NVIDIA driver broken after kernel upgrade   | Yes                   | `timeshift --restore`        |
| Deleted file from `~`                       | **No** (~ is excluded) | Restic backup from [8.2](../08-productivity-security/02-backup-strategy-3-2-1.md) |
| NVMe hardware failure                       | **No**                | Offsite Restic backup        |
| `rm -rf /` from a compromised account       | **No** (snapshots go too) | Offsite backup            |
| Data encrypted by ransomware                | **No**                | Offsite backup with versioning |

This is why [8.2](../08-productivity-security/02-backup-strategy-3-2-1.md) backups are separate and mandatory.

## 2.5.8 Snapshot before every chapter

From here on, each post-install step in this guide recommends a `timeshift --create` at the start. The [99 provisioning script](../99-provisioning-script.md) has them built in. After a while it becomes a tic, like saving a document. Good.

Proceed to [03 — Speed, Boot, and Responsiveness](../03-speed-and-boot-optimizations/README.md).

---

[Home](../README.md) · [↑ 02 Post-Install](README.md) · [← Previous: 2.4 Wayland + Plasma](04-wayland-plasma-66-nvidia.md) · **2.5 Snapshot policy** · [Next: 03 Speed & Boot →](../03-speed-and-boot-optimizations/README.md)
