[Home](../README.md) · [↑ 01 Install](README.md) · [← Previous: 1.3 Installer flavor](03-installer-flavor-minimal-vs-full.md) · **1.4 Partitioning & install** · [Next: 1.5 First boot →](05-first-boot-and-snapshot.md)

---

# 1.4 Installation and Partitioning (BTRFS subvolumes, systemd-boot)

The meaningful Calamares screen. You will **manually partition Disk B**, create a BTRFS filesystem with subvolumes matching the layout Snapper / Timeshift expects, and land the bootloader on Disk B's own ESP.

**Do not pick "Install Kubuntu alongside Windows" or "Erase disk and install Kubuntu."** Those options will Do The Wrong Thing (the first tries to install on Disk A, the second will nuke whichever disk Calamares decides is "the" disk). Pick **Manual partitioning** every time.

## Target layout on Disk B

```
Disk B (512 GB NVMe, e.g. /dev/nvme1n1):

  ┌────────────────────────────────────────────────────────────────────┐
  │ GPT                                                                │
  ├────────────────────────────────────────────────────────────────────┤
  │ /dev/nvme1n1p1   1 GiB    FAT32      flagged 'boot,esp'  → /boot/efi
  │ /dev/nvme1n1p2   2 GiB    ext4                           → /boot
  │ /dev/nvme1n1p3   ~473 GiB BTRFS (subvolumes)             → see below
  │ /dev/nvme1n1p4   20 GiB   swap (actually a BTRFS file,   → noauto
  │                           created after install)
  └────────────────────────────────────────────────────────────────────┘

BTRFS subvolume layout inside /dev/nvme1n1p3:

  @            → /          (rootfs)
  @home        → /home
  @snapshots   → /.snapshots (Timeshift target)
  @var-log     → /var/log   (excluded from root snapshots)
  @var-cache   → /var/cache (excluded from root snapshots)
```

This layout matches what Timeshift 26 (BTRFS mode) auto-detects, and matches what Snapper would expect if you switch later. It is identical to the `docs/02-setup/03-install-partitioning.md` baseline.

### Why these choices (one-liners each)

- **Separate 1 GiB ESP** — Calamares insists on a boot/esp flag; 1 GiB is grossly oversized for ESP's typical contents, but NVMe cells are cheap and it leaves room if you ever install a second kernel line (mainline PPA, XanMod, rt) side by side.
- **Separate 2 GiB /boot ext4** — not on BTRFS because `systemd-boot`'s UKIs and stub kernels live there; keeping it on ext4 avoids BTRFS `grub-btrfs` complications and avoids the edge case where a snapshot rollback rolls back your `/boot` too.
- **BTRFS with subvolumes** — we want compression (`zstd:1`), copy-on-write for snapshots, and the ability to roll back `/` without rolling back `/home`. Subvolumes are how.
- **Swap as a file, not a partition** — BTRFS swapfiles have been stable since kernel 5.8; files are easier to resize than partitions and the 2020-era TUF hibernate path is non-critical for us.
- **`/var/log` and `/var/cache` as separate subvolumes excluded from the root snapshot** — so a snapshot rollback doesn't revert your logs (destroying post-incident forensics) or your apt cache (forcing re-download of GBs).

## Step-by-step in Calamares

### 1.4.1 Boot into the live session

- Power on, F8, select the USB, boot Kubuntu.
- At the Ventoy menu (if applicable), select `kubuntu-26.04-desktop-amd64.iso`, then "Boot in normal mode".
- At the GRUB menu, select **Try or Install Kubuntu**.
- Wait for the Plasma 6.6 live desktop to appear. Click **Install Kubuntu** on the desktop.

### 1.4.2 Welcome, location, keyboard

Self-explanatory. Your timezone, your keyboard layout. Proceed.

### 1.4.3 Software selection (see [1.3](03-installer-flavor-minimal-vs-full.md))

**Minimal**. ☑ third-party software. ☐ updates-during-install.

### 1.4.4 Partitioning — pick Manual

On the "Installation type" screen, select **Manual partitioning** (radio button at the bottom). Click Next.

You see a table of all disks. Identify Disk B by the serial number you noted in [1.1](01-hardware-check-and-usb-prep.md) — typically `/dev/nvme1n1` (Windows is on `/dev/nvme0n1`).

**Confirm, then confirm again, that you are working on Disk B.** Everything below destroys Disk B.

### 1.4.5 Create a new partition table on Disk B

- Select Disk B in the tree.
- Click **New Partition Table** (or right-click → New Partition Table).
- Choose **GPT**.
- Apply.

Disk B is now one big "Free space" entry.

### 1.4.6 Create the ESP

- Select "Free space" under Disk B.
- Click **Create**.
- Size: **1024 MiB**.
- File System: **fat32**.
- Mount Point: **/boot/efi**.
- Flags: tick **boot** and **esp**.
- OK.

### 1.4.7 Create /boot

- Select the remaining "Free space".
- Create.
- Size: **2048 MiB**.
- File System: **ext4**.
- Mount Point: **/boot**.
- No flags.
- OK.

### 1.4.8 Create the main BTRFS partition

- Select the remaining "Free space".
- Create.
- Size: **the rest minus 20 GiB, minus 1 MiB margin**. If total free space is 484 GiB, enter 464 GiB. (The exact number doesn't matter; we just want ~20 GiB left for swap.)
- File System: **btrfs**.
- Mount Point: **/**.
- No flags.
- OK.

### 1.4.9 Create a BTRFS "placeholder" swap partition

This will actually become a swap **file** inside the BTRFS root later, but Calamares's BTRFS support in 26.04 does not create a swapfile directly, so we reserve the space:

- Select the remaining "Free space".
- Create.
- Size: **20480 MiB** (all remaining).
- File System: **btrfs**.
- Mount Point: leave blank for now (`fstab` entry will be adjusted).
- OK.

We will convert this to a swapfile post-install in [3.2](../03-speed-and-boot-optimizations/02-zram-swap-sysctl.md). For now it is unused space.

### 1.4.10 Choose bootloader location

At the bottom of the Manual Partitioning screen: **Boot loader installation:**.

- Set to **Disk B** (e.g. `Master Boot Record of /dev/nvme1n1`) — but on GPT/UEFI this really installs to the ESP you just flagged. Just confirm it says the Disk B device, NOT Disk A. This is the one-click save that prevents the installer from touching Disk A.

### 1.4.11 Review and install

Calamares shows a summary: "Format /dev/nvme1n1p1 as fat32, format /dev/nvme1n1p2 as ext4, ..." etc. **Read it carefully**:

- `/dev/nvme1n1` is Disk B (Linux target). Every operation should be on `nvme1n1p*`.
- `/dev/nvme0n1` should NOT appear anywhere. If it does, back out and revisit the partition table.

Click **Install** when satisfied. Takes 15–25 minutes on USB 3.

## During the install: create the user

Calamares will pop up the "Who are you?" screen. See [1.3 Other installer screens](03-installer-flavor-minimal-vs-full.md#other-installer-screens--quick-notes). Username short, password strong, hostname recognisable, auto-login off.

## When install finishes

Calamares says "Installation Finished" and offers **Restart Now** or **Done**. **Click Done** (do not restart yet). We have subvolume adjustments to make in the live environment before first boot.

### 1.4.12 Adjust subvolumes in the live environment (optional but recommended)

Kubuntu's Calamares in 26.04 creates basic BTRFS subvolumes (`@`, `@home`) by default but does not create the `@snapshots`, `@var-log`, `@var-cache` subvolumes this guide's snapshot policy wants. If you skip this, [2.5 Snapshot policy](../02-post-install-foundations/05-snapshot-policy-btrfs-timeshift.md) will do it for you post-first-boot — slightly messier but fine.

If you want to do it now:

Open Konsole in the live session.

```bash
# Mount the BTRFS root filesystem to a scratch point:
sudo mkdir -p /mnt/install
sudo mount -o subvolid=5 /dev/nvme1n1p3 /mnt/install

# Inspect what Calamares created:
sudo btrfs subvolume list /mnt/install
# Expect: @  (id 256) and @home (id 257)

# Create the additional subvolumes:
sudo btrfs subvolume create /mnt/install/@snapshots
sudo btrfs subvolume create /mnt/install/@var-log
sudo btrfs subvolume create /mnt/install/@var-cache

# Move the existing @/var/log and @/var/cache contents over:
sudo mount /dev/nvme1n1p3 -o subvol=@ /mnt/install-root
sudo mkdir -p /mnt/install-varlog /mnt/install-varcache
sudo mount /dev/nvme1n1p3 -o subvol=@var-log /mnt/install-varlog
sudo mount /dev/nvme1n1p3 -o subvol=@var-cache /mnt/install-varcache
sudo cp -a /mnt/install-root/var/log/. /mnt/install-varlog/
sudo cp -a /mnt/install-root/var/cache/. /mnt/install-varcache/
sudo rm -rf /mnt/install-root/var/log/* /mnt/install-root/var/cache/*

# Edit /etc/fstab in the installed system to mount the new subvolumes:
sudo nano /mnt/install-root/etc/fstab
```

Add these lines (adjust UUID by copying from the existing `@home` line):

```text
UUID=<same-as-root>  /var/log    btrfs  subvol=@var-log,noatime,compress=zstd:1,space_cache=v2,discard=async  0 0
UUID=<same-as-root>  /var/cache  btrfs  subvol=@var-cache,noatime,compress=zstd:1,space_cache=v2,discard=async  0 0
UUID=<same-as-root>  /.snapshots btrfs  subvol=@snapshots,noatime,compress=zstd:1,space_cache=v2,discard=async  0 0
```

Create the mount points:

```bash
sudo mkdir -p /mnt/install-root/.snapshots /mnt/install-root/var/log /mnt/install-root/var/cache
```

Unmount:

```bash
sudo umount /mnt/install-varlog /mnt/install-varcache /mnt/install-root /mnt/install
```

If any of the above confuses you, skip it — [2.5](../02-post-install-foundations/05-snapshot-policy-btrfs-timeshift.md) does the same work post-first-boot.

## Optional: convert GRUB to systemd-boot

Calamares 26.04 installs GRUB by default. It works. `systemd-boot` is arguably cleaner on a single-kernel UEFI system (no BIOS-era baggage, faster boot by ~0.5 s, simpler `loader.conf` vs `grub.cfg`) but **neither is wrong**.

If you want to switch to systemd-boot, do it post-first-boot after confirming GRUB works — trying to do the switch in the live environment requires chroot gymnastics.

The systemd-boot conversion procedure is identical to `docs/02-setup/03-install-partitioning.md § Optional systemd-boot conversion`. Since nothing has changed for 26.04 beyond the codename, see that page for the full procedure. If you do not convert, GRUB on BTRFS works; the only noticeable downside is a ~0.5 s boot splash.

## Reboot into Kubuntu

1. Power off from the live environment (`sudo poweroff`).
2. **Remove the USB.**
3. Power on.
4. **Mash F8** — select `Kubuntu` (the new entry pointing at Disk B's ESP).
5. You'll see the GRUB menu (or systemd-boot if you converted). Kernel boots. SDDM appears.
6. Log in with the username/password you created in [1.3](03-installer-flavor-minimal-vs-full.md#other-installer-screens--quick-notes).
7. You are on Kubuntu 26.04 LTS "Resolute Raccoon" on your own hardware. Proceed to [1.5 First boot](05-first-boot-and-snapshot.md).

## If something goes wrong

- **F8 shows no Kubuntu entry, only Windows:** Reboot, enter BIOS (F2), check **Boot → Boot Option Priorities**. Kubuntu's ESP should appear as "ubuntu" or "Kubuntu" in the firmware boot table. Drag it to position #1, or use the one-time boot to test it. If it doesn't appear at all, the bootloader never landed — re-run Calamares with particular attention to the "Boot loader installation" selection in [1.4.10](#1410-choose-bootloader-location).
- **Kubuntu entry exists but selecting it returns to Windows:** ESP ordering on TUF A17 is occasionally weird. From Windows (or a Windows install USB's recovery command prompt): `bcdedit /enum firmware` and confirm there's an entry pointing at Disk B's ESP `\EFI\ubuntu\shimx64.efi` or `\EFI\systemd\systemd-bootx64.efi`.
- **Installer froze at ~30 %:** usually a failing USB. Reboot, try a different USB port, re-write the USB.

More in [10-troubleshooting.md](../10-troubleshooting.md).

---

[Home](../README.md) · [↑ 01 Install](README.md) · [← Previous: 1.3 Installer flavor](03-installer-flavor-minimal-vs-full.md) · **1.4 Partitioning & install** · [Next: 1.5 First boot →](05-first-boot-and-snapshot.md)
