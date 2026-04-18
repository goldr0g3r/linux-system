[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.2 BIOS](02-bios-uefi.md) · **2.3 Installation & Partitioning** · [Next: 2.4 Debloat →](04-debloat.md)

---

# 2.3 Installation & Partitioning

This is the longest section of the setup. Budget 45–60 minutes including installer runtime and the post-install subvolume work.

## 2.3.1 The layout we are building

On the Linux SSD (`nvme1n1`):

| # | Size | Type | Filesystem | Mount point | Flags |
|---|------|------|------------|-------------|-------|
| 1 | 1 GiB | EFI System | FAT32 | `/boot/efi` | `boot`, `esp` |
| 2 | 2 GiB | Linux | ext4 | `/boot` | — |
| 3 | Remainder (~509 GiB) | Linux | BTRFS | `/` | — |

The BTRFS partition will host these subvolumes after install:

- `@` — root filesystem
- `@home` — user home directories
- `@var_log` — logs (excluded from snapshots to keep them small)
- `@var_cache` — apt/dnf caches (same reason)
- `@snapshots` — Timeshift / Snapper snapshots live here
- `@swap` — nodatacow subvolume hosting a 20 GiB swapfile

## 2.3.2 Run the installer

Boot the live session. Connect Wi-Fi. Launch **"Install Kubuntu"** (Calamares installer). Progress through language, keyboard, network until you reach **Partitions**. Pick **Manual partitioning**.

Identify your disks: Windows is on `/dev/nvme0n1` (has Microsoft reserved + NTFS partitions). Linux target is `/dev/nvme1n1` (blank). Confirm by checking sizes. **Do not touch `nvme0n1`.**

Wipe `nvme1n1` and create a GPT partition table. Create the three partitions from the table above.

On the BTRFS partition, Calamares creates a default subvolume layout. **We will re-do it post-install** for precision. Accept Calamares's defaults for now (it will create `@` and `@home`). Set the mount options for the BTRFS partition to `noatime,compress=zstd:1,ssd,discard=async,space_cache=v2`.

**Do not enable an encrypted install.** You can layer LUKS later if you need it; for a dev laptop with Secure Boot off and no regulated data, it is overhead without a clear threat model. If you do need it, use LUKS on the BTRFS partition only (not on `/boot`).

Set username, hostname (suggest `tuf-a17` or similar), password. Let Calamares install. **Reboot into the live USB** (not into the installed system yet) — we need to finish the subvolume layout from a live shell.

## 2.3.3 Post-install subvolume finalization (from live USB)

Open Konsole in the live session.

### Step 1: mount the BTRFS top and create subvolumes

```bash
# Mount the top of the BTRFS tree (subvolid=5 is the root of the filesystem, above @ and @home).
sudo mkdir -p /mnt/btrfs-root
sudo mount -o subvolid=5,noatime,compress=zstd:1 /dev/nvme1n1p3 /mnt/btrfs-root
ls /mnt/btrfs-root

# Create the engineering-grade subvolume layout.
cd /mnt/btrfs-root
sudo btrfs subvolume create @var_log
sudo btrfs subvolume create @var_cache
sudo btrfs subvolume create @snapshots
sudo btrfs subvolume create @swap

# Move existing @var/log and @var/cache contents into the new subvolumes (skip if the dirs are empty).
sudo rsync -aAXH @/var/log/ @var_log/  && sudo rm -rf @/var/log/*
sudo rsync -aAXH @/var/cache/ @var_cache/ && sudo rm -rf @/var/cache/*

# Create the swap subvolume's file.
sudo btrfs property set @swap compression none
sudo mkdir -p @swap/swapfile-mnt
sudo truncate -s 0 @swap/swapfile
sudo chattr +C @swap/swapfile
sudo fallocate -l 20G @swap/swapfile
sudo chmod 600 @swap/swapfile
sudo mkswap @swap/swapfile
```

### Step 2: update `/etc/fstab` on the installed system

Mount the installed root via `subvol=@` and chroot in:

```bash
sudo mkdir -p /mnt/target
sudo mount -o subvol=@,noatime,compress=zstd:1,ssd,discard=async /dev/nvme1n1p3 /mnt/target
sudo mount /dev/nvme1n1p2 /mnt/target/boot
sudo mount /dev/nvme1n1p1 /mnt/target/boot/efi
sudo mount --bind /dev  /mnt/target/dev
sudo mount --bind /proc /mnt/target/proc
sudo mount --bind /sys  /mnt/target/sys
sudo chroot /mnt/target /bin/bash

# Inside the chroot:
cat >> /etc/fstab <<'EOF'

# Engineering subvolume additions
UUID=$(blkid -s UUID -o value /dev/nvme1n1p3) /var/log      btrfs subvol=@var_log,noatime,compress=zstd:1,ssd,discard=async 0 0
UUID=$(blkid -s UUID -o value /dev/nvme1n1p3) /var/cache    btrfs subvol=@var_cache,noatime,compress=zstd:1,ssd,discard=async 0 0
UUID=$(blkid -s UUID -o value /dev/nvme1n1p3) /.snapshots   btrfs subvol=@snapshots,noatime,compress=zstd:1,ssd,discard=async 0 0
UUID=$(blkid -s UUID -o value /dev/nvme1n1p3) /swap         btrfs subvol=@swap,noatime,ssd,discard=async 0 0
/swap/swapfile none swap defaults 0 0
EOF

# The heredoc leaves literal $(blkid ...) strings — replace them with the real UUID:
UUID=$(blkid -s UUID -o value /dev/nvme1n1p3)
sed -i "s|\\\$(blkid -s UUID -o value /dev/nvme1n1p3)|${UUID}|g" /etc/fstab

mkdir -p /var/log /var/cache /.snapshots /swap
exit
sudo reboot
```

### Step 3: verify on first boot

On reboot, pull the USB and let it boot into the installed Kubuntu. Confirm the layout:

```bash
sudo btrfs subvolume list /
swapon --show
findmnt -t btrfs
```

You should see `@`, `@home`, `@var_log`, `@var_cache`, `@snapshots`, `@swap`, and a 20 GiB swap on `/swap/swapfile`.

## 2.3.4 Optional: convert GRUB to systemd-boot

Calamares installs GRUB 2. A well-tuned GRUB is fine. If you want sub-second boot menus and simpler kernel pinning, convert to systemd-boot:

```bash
sudo bootctl install
sudo apt install -y systemd-boot

# Ubuntu's systemd-boot integration ships /etc/kernel/install.d hooks that generate entries into the ESP.
# Configure the command-line once:
echo 'root=UUID='$(blkid -s UUID -o value /dev/nvme1n1p3)' rootflags=subvol=@ ro quiet splash nowatchdog nvme_load=YES mitigations=off amd_pstate=passive nvidia-drm.modeset=1 nvidia-drm.fbdev=1' | sudo tee /etc/kernel/cmdline

# Regenerate all kernel entries.
sudo dpkg-reconfigure linux-image-$(uname -r)

# Verify:
sudo bootctl list

# Remove GRUB only after confirming systemd-boot works on the next reboot:
# sudo apt purge -y grub-efi-amd64 grub-efi-amd64-bin grub-efi-amd64-signed grub-common grub-pc-bin grub2-common os-prober
```

### If you stay on GRUB

Edit `/etc/default/grub`:

```bash
GRUB_TIMEOUT=0
GRUB_TIMEOUT_STYLE=hidden
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nowatchdog nvme_load=YES mitigations=off amd_pstate=passive nvidia-drm.modeset=1 nvidia-drm.fbdev=1"
GRUB_DISABLE_OS_PROBER=true
```

Then `sudo update-grub`.

## Trade-offs summary

| Choice | Pro | Con |
|--------|-----|-----|
| BTRFS with subvolumes | Snapshots, compression (~25–40% savings on ROS trees), checksumming | Slight metadata overhead; balance needed quarterly |
| ext4 `/boot` (not on BTRFS) | systemd-boot + GRUB both happy without BTRFS quirks | One more partition to manage |
| 20 GiB swapfile | Covers hibernation + AI-training spill | Uses 20 GiB; can shrink later if you never hibernate |
| systemd-boot | Sub-second boot menu, simpler kernel metadata | Fewer boot-time features (no OS prober) |
| GRUB (default) | OS prober can detect Windows (but we do not want that — F8 is cleaner) | Slower boot, more config surface |

Proceed to [2.4 First-boot debloat](04-debloat.md).

---

[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.2 BIOS](02-bios-uefi.md) · **2.3 Installation & Partitioning** · [Next: 2.4 Debloat →](04-debloat.md)
