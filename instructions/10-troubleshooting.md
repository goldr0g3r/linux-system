[Home](README.md) · [← Previous: 09 Apps catalog](09-apps-catalog.md) · **10 — Troubleshooting** · [Next: 99 Provisioning →](99-provisioning-script.md)

---

# 10 — Troubleshooting Matrix

Known pitfalls on Kubuntu 26.04 LTS "Resolute Raccoon" on the ASUS TUF A17 + NVIDIA RTX 3050 Mobile, grouped by system layer. Each entry: symptom → likely cause → fix.

## 10.1 Boot / bootloader

### Symptom: F8 does not show Kubuntu entry

- **Cause**: bootloader didn't install on Disk B's ESP (landed on Disk A or failed outright).
- **Fix**: boot live USB → chroot into installed system → reinstall GRUB.

```bash
# From live session:
sudo mkdir /mnt/recover
sudo mount /dev/nvme1n1p3 /mnt/recover -o subvol=@
sudo mount /dev/nvme1n1p2 /mnt/recover/boot
sudo mount /dev/nvme1n1p1 /mnt/recover/boot/efi
for i in proc sys dev dev/pts run; do sudo mount --bind /$i /mnt/recover/$i; done
sudo chroot /mnt/recover
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu --removable
update-grub
exit
# Clean up:
for i in proc sys dev/pts dev run boot/efi boot ; do sudo umount /mnt/recover/$i; done
sudo umount /mnt/recover
```

### Symptom: Kubuntu entry exists in F8 but selecting boots Windows

- **Cause**: ASUS firmware boot-var confusion. The Kubuntu entry points at Windows Boot Manager by name collision or file.
- **Fix**: enter BIOS, delete old boot entries via **Boot → Delete Boot Option**, then re-boot to rescan NVRAM. Or from Windows: `bcdedit /enum firmware` → delete the bad entry.

### Symptom: boot hangs at "Starting Load Kernel Modules…"

- **Cause**: NVIDIA DKMS module failed to build for the new kernel.
- **Fix**: at GRUB menu press `e`, add `nomodeset` to the kernel line, F10 to boot. Once booted, reinstall the driver: `sudo apt install --reinstall nvidia-driver-580`.

## 10.2 NVIDIA / Wayland

### Symptom: screen flickers, artifacts under video playback

- **Cause**: Explicit Sync not active or Firefox not using it.
- **Fix**: verify `nvidia-drm.modeset=1` in `/proc/cmdline`, confirm in `about:config` that `widget.dmabuf.force-enabled=true`. Update Firefox if <148.

### Symptom: `nvidia-smi` says "No devices were found"

- **Cause**: `supergfxctl` in Integrated mode. Or driver not loaded.
- **Fix**:
  ```bash
  supergfxctl -g
  # If "Integrated":
  supergfxctl -m Hybrid
  # Log out, log back in.
  nvidia-smi
  ```
  If still no devices: `lsmod | grep nvidia`. If empty, `sudo modprobe nvidia`. If that errors, check `journalctl -b -u dkms` for build failures.

### Symptom: black screen after waking from suspend

- **Cause**: NVIDIA driver failed to restore VRAM state.
- **Fix**: ensure `nvidia.NVreg_PreserveVideoMemoryAllocations=1` is in kernel cmdline (it should be from [3.1](03-speed-and-boot-optimizations/01-kernel-boot-parameters.md)). Also ensure these systemd services are enabled (they should be from the nvidia-driver-580 package):
  ```bash
  sudo systemctl enable nvidia-suspend.service nvidia-hibernate.service nvidia-resume.service
  ```
  If still broken, fall back to `supergfxctl -m Integrated` for travel until fix lands.

### Symptom: Plasma-Wayland login loop

- **Cause**: KWin crashes on initial Wayland setup with NVIDIA.
- **Fix**: switch to X11 session as break-glass. Log out → SDDM → click your user → session picker (bottom-left) → Plasma (X11). First install:
  ```bash
  # From TTY (Ctrl+Alt+F2):
  sudo apt install -y plasma-workspace-x11
  ```
  Investigate Wayland separately via `journalctl -b _COMM=kwin_wayland`.

### Symptom: Chromium / Brave / Electron apps show black window on Wayland

- **Cause**: GPU process crash.
- **Fix**: launch with `--disable-gpu` as a temporary workaround. Permanent fix: update to latest version of the app. If persistent, add `--ozone-platform=x11` to force XWayland for that app.

### Symptom: fractional scaling blurry text

- **Cause**: app not using `wp_fractional_scale_v1`. Falls back to integer scale + bitmap-scaling.
- **Fix**: mostly native Qt6 apps work. For Chromium-based, launch with `--enable-features=WaylandFractionalScaleV1`. For GTK apps, `GDK_BACKEND=wayland` + `GDK_SCALE=2` override.

## 10.3 Audio / Bluetooth

### Symptom: Bluetooth headset connects but no sound

- **Cause**: wrong default sink.
- **Fix**:
  ```bash
  pactl list sinks short
  pactl set-default-sink bluez_output.XX_XX_XX_XX_XX_XX.a2dp-sink
  ```

### Symptom: Bluetooth headset codec stuck on SBC

- **Cause**: LDAC/AptX HD not advertised or WirePlumber config not loaded.
- **Fix**:
  ```bash
  systemctl --user restart wireplumber.service
  pactl list sinks | grep "api.bluez5.codec"
  ```
  If still SBC, check [4.3.2 codec priority config](04-daily-driver-stack/03-bluetooth-headset-hifi.md#432-enable-high-quality-a2dp-in-wireplumber).

### Symptom: no sound from laptop speakers after unplugging headphones

- **Cause**: PipeWire port-switching stuck.
- **Fix**: right-click Plasma volume icon → switch output to Built-in Audio. Or from CLI:
  ```bash
  pactl list sinks short
  pactl set-default-sink alsa_output.pci-*.analog-stereo
  ```

### Symptom: microphone doesn't work in Firefox calls

- **Cause**: xdg-desktop-portal-kde not running or missing permission.
- **Fix**:
  ```bash
  systemctl --user restart xdg-desktop-portal.service xdg-desktop-portal-kde.service
  ```
  In browser, check site permissions for microphone.

## 10.4 sudo-rs

### Symptom: `sudo -i` hangs or errors with "unknown directive"

- **Cause**: a `sudoers.d/` snippet uses a directive sudo-rs doesn't support.
- **Fix**: identify the bad snippet:
  ```bash
  sudo -i  # will say which file errored
  sudoedit /etc/sudoers.d/<file>
  # Edit or delete.
  sudo visudo -c  # syntax check
  ```
  Common culprit: custom LDAP/NIS modules; rarely matters on a laptop.

### Symptom: `sudo -E <cmd>` doesn't preserve environment correctly

- **Cause**: sudo-rs handles environment whitelisting slightly differently from C sudo.
- **Fix**: add the variable explicitly:
  ```bash
  sudo --preserve-env=HOME,DISPLAY,XAUTHORITY <cmd>
  ```
  Or in `/etc/sudoers.d/env_keep`:
  ```
  Defaults env_keep += "DISPLAY XAUTHORITY HOME"
  ```

## 10.5 Snap (should not be present but sometimes is)

### Symptom: `apt install firefox` installs the snap wrapper instead of Mozilla APT

- **Cause**: Mozilla APT repo not present, or snap pinning failed.
- **Fix**: re-run [2.1.3](02-post-install-foundations/01-debloat-snap-flatpak-pim.md#213-firefox-via-mozillas-apt-repository). Verify `apt policy firefox` shows Mozilla as the top source.

### Symptom: `snapd` reappeared after some upgrade

- **Cause**: a package (rare) depends on snapd via recommends.
- **Fix**: the `/etc/apt/preferences.d/nosnap.pref` file should have prevented this. Recheck:
  ```bash
  cat /etc/apt/preferences.d/nosnap.pref
  sudo apt update
  sudo apt purge -y snapd
  ```

## 10.6 BTRFS / filesystem

### Symptom: `Transaction commit will overflow memory` or `btrfs: ENOSPC` despite free space showing

- **Cause**: BTRFS metadata chunk exhaustion; common on heavy-churn filesystems.
- **Fix**:
  ```bash
  # Check usage:
  sudo btrfs filesystem usage /

  # Balance (rewrites allocations):
  sudo btrfs balance start -dusage=50 -musage=50 /
  # (takes minutes; -dusage/-musage value chooses what to rewrite.)
  ```

### Symptom: boot falls through to initrd emergency shell

- **Cause**: BTRFS subvolume rename lost `/` or `/home` mount in fstab.
- **Fix**: in initrd shell, mount the BTRFS root manually, chroot, edit fstab.
  ```bash
  mkdir /newroot
  mount -o subvol=@ /dev/nvme1n1p3 /newroot
  mount --bind /dev /newroot/dev && mount --bind /proc /newroot/proc && mount --bind /sys /newroot/sys
  chroot /newroot
  nano /etc/fstab  # fix UUIDs or subvol paths
  exit
  reboot
  ```

### Symptom: Timeshift snapshots growing unexpectedly

- **Cause**: `--exclude` not honoured for `/var/log` or `/home`.
- **Fix**: check Timeshift settings, disable "Include user hidden files" if accidentally enabled. Verify `/home` exclusion under "Users" tab.

## 10.7 ROS 2 / Distrobox

### Symptom: `ros2 topic list` inside container hangs forever

- **Cause**: DDS multicast fails because `--network=host` not set when creating the Distrobox, or firewall blocks.
- **Fix**: recreate container with `--additional-flags "--network=host"` (see [6.3.2](06-ros2-robotics/03-distrobox-bridge-jazzy-now.md#632-create-the-jazzy-container)). Also ensure UFW on host allows loopback:
  ```bash
  sudo ufw allow from 127.0.0.0/8
  ```

### Symptom: `rviz2` opens but renders black

- **Cause**: `/dev/dri` not passed to container, software rendering engaged.
- **Fix**: verify in container: `glxinfo | grep "OpenGL renderer"`. Should say hardware, not `llvmpipe`. If llvmpipe, recreate container with `--device /dev/dri`.

### Symptom: ROS 2 Jazzy in container can't see NVIDIA GPU

- **Cause**: NVIDIA Container Toolkit not configured.
- **Fix**: follow [6.3.8 CDI setup](06-ros2-robotics/03-distrobox-bridge-jazzy-now.md#nvidia-container-toolkit-for-cuda-inside-containers).

## 10.8 TUF A17 specific

### Symptom: Wi-Fi (MT7921) drops after suspend

- **Cause**: MediaTek driver state-machine occasionally gets stuck on resume.
- **Fix**: short-term: `sudo rmmod mt7921e; sudo modprobe mt7921e`. Long-term: kernel 7.0 has better MT7921 handling than 6.x; confirm you're on ≥7.0.4 (any later point release).

### Symptom: battery drains while laptop is plugged in

- **Cause**: dGPU stuck awake or Armoury Crate (Windows) left PPT in Turbo mode at last shutdown.
- **Fix**:
  ```bash
  supergfxctl -m Integrated  # temporarily
  # Boot once into Windows, set profile to "Balanced" in Armoury Crate, shutdown clean.
  # Re-boot Linux. Set asusctl profile back to Balanced.
  ```

### Symptom: `Fn+F5` doesn't cycle profile

- **Cause**: `asusctl` service not running, or user not in `input` group.
- **Fix**:
  ```bash
  sudo systemctl status asusd.service
  sudo usermod -aG input $USER
  # Log out, log back in.
  ```

### Symptom: suspend works once, second suspend fails (black screen on wake)

- **Cause**: known NVIDIA + Wayland + older firmware combo. Typically resolves with BIOS update.
- **Fix**: update BIOS via `sudo fwupdmgr refresh && sudo fwupdmgr get-updates`. If not offered via LVFS, flash latest from ASUS support page (see [1.1 hardware check](01-install/01-hardware-check-and-usb-prep.md#other-physical-checks)).

### Symptom: fans never spin or ramp to 100 % immediately

- **Cause**: `asusctl` fan-curve conflict or sensor reading offset.
- **Fix**:
  ```bash
  # Reset to stock fan curve:
  asusctl fan-curve -m Balanced -e false

  # Confirm sensor readings:
  sensors | grep -E "Tctl|Tdie"
  # Expect 35-55 °C idle. If 150 °C reported, sensor offset broken — try:
  sudo sensors-detect --auto
  ```

## 10.9 Networking

### Symptom: DNS resolution slow or fails for some domains

- **Cause**: `systemd-resolved` DNS-over-TLS misconfiguration or ISP DNS blocking.
- **Fix**:
  ```bash
  resolvectl status
  # Check current DNS servers.

  # Override to Cloudflare:
  sudo tee /etc/systemd/resolved.conf.d/cloudflare-dns.conf <<'EOF'
[Resolve]
DNS=1.1.1.1 1.0.0.1 2606:4700:4700::1111
DNSOverTLS=yes
EOF
  sudo systemctl restart systemd-resolved.service
  ```

### Symptom: Wi-Fi connects but no internet

- **Cause**: captive portal not caught by NM.
- **Fix**: open Firefox to `http://captive.apple.com` — browser opens the portal prompt. If not, the network has NAT but DNS is blocked; use cellular hotspot to proceed.

## 10.10 Plasma-specific

### Symptom: Plasma panel disappears after update

- **Cause**: panel config corruption.
- **Fix**:
  ```bash
  # Backup old:
  cp ~/.config/plasma-org.kde.plasma.desktop-appletsrc ~/.config/plasma-org.kde.plasma.desktop-appletsrc.bak
  rm ~/.config/plasma-org.kde.plasma.desktop-appletsrc
  # Log out, log back in. Default panel restores.
  ```

### Symptom: Global Menu shows "-" instead of app menus

- **Cause**: XWayland app not advertising menu bar.
- **Fix**: accept; works only for Qt6 and modern GTK apps. Legacy apps without D-Bus menu will show their own in-app menu.

### Symptom: KRunner won't launch some apps

- **Cause**: `.desktop` file cache stale.
- **Fix**:
  ```bash
  kbuildsycoca6 --noincremental
  ```

## 10.11 Generic debugging playbook

When something "just stops working":

1. **Is it a recent change?** `sudo apt history | head -50` and `ls -lt /etc/` — what did you edit?
2. **Did a recent package update cause it?** `apt-get install <package>=<prev-version>` to roll back.
3. **Can you reproduce in a clean session?** Log out, log back in. Cold reboot.
4. **Rollback via Timeshift.** `sudo timeshift --list` → restore to a pre-break snapshot. Fix forward from there.
5. **If in a Distrobox**: `distrobox stop <name>; distrobox enter <name>` to reset — state is in the container, not your host.

If none of the above:

- Search exact error message.
- Check `/var/log/syslog`, `journalctl -b -p err`, `dmesg -T | tail -50`.
- Ask on KDE forums, Ubuntu discourse, or the relevant upstream's GitHub issues.
- Post to `/r/kubuntu` with output of `inxi -Fxxx` (system info).

---

[Home](README.md) · [← Previous: 09 Apps catalog](09-apps-catalog.md) · **10 — Troubleshooting** · [Next: 99 Provisioning →](99-provisioning-script.md)
