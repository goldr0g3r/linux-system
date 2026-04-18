[Home](../README.md) · [← Previous: Appendix A](appendix-a-provisioning.md) · **Appendix B: Troubleshooting**

---

# Appendix B — Troubleshooting Matrix (TUF A17 + NVIDIA)

Common failure modes and concrete fixes. Symptoms are ordered roughly by when in the setup they can surface (install → first boot → daily use → suspend/wake → specific tools).

## Install & first-boot issues

| Symptom | Most likely cause | Fix |
|---------|-------------------|-----|
| Live USB boots to black screen after Kubuntu logo | Wrong BIOS graphics setting (CSM on, or RAID mode) | BIOS: AHCI, CSM off, UEFI only. On the GRUB menu press `e` and add `nomodeset` to the live boot as a one-off. See [2.2 BIOS](02-setup/02-bios-uefi.md). |
| Installer shows "no disks found" | SATA mode still on RAID / Intel RST | Switch BIOS to AHCI. If Windows was installed on RAID, see the `bcdedit safeboot` recipe in [2.2](02-setup/02-bios-uefi.md). |
| Installed system boots to black screen after NVIDIA install | NVIDIA DRM modeset mismatch with initramfs | Boot into recovery, `sudo update-initramfs -u`; confirm `nvidia-drm.modeset=1` is in `/etc/kernel/cmdline` or `/etc/default/grub`. See [2.3.4](02-setup/03-install-partitioning.md). |
| Login screen (SDDM) loops after entering password | Display manager cannot initialize X with NVIDIA | `Ctrl+Alt+F3` → login → `sudo dpkg-reconfigure nvidia-driver-580` → `sudo systemctl restart sddm` |

## NVIDIA / hybrid GPU

| Symptom | Most likely cause | Fix |
|---------|-------------------|-----|
| Laptop suspend, wake to black screen, fans spin | NVIDIA suspend-resume bug with older kernel | Install `linux-generic-hwe-24.04`, enable `sudo systemctl enable nvidia-suspend.service nvidia-resume.service nvidia-hibernate.service` |
| `nvidia-smi` says "No devices were found" | `supergfxctl` is in `Integrated` mode | `supergfxctl -m Hybrid` → reboot. See [2.6](02-setup/06-asus-hardware.md). |
| Massive battery drain when idle (2–3 h) | dGPU not going to D3cold; ASPM not forced | Kernel cmdline: add `pcie_aspm=force nvidia.NVreg_DynamicPowerManagement=0x02` |
| `prime-run` shows `OpenGL renderer: llvmpipe` | PRIME offload environment not set | Ensure `nvidia-prime` is installed and use `prime-run <cmd>`; check `glxinfo -B` vs `prime-run glxinfo -B`. See [3.3](03-dev-environment/03-ai-ml.md). |
| MUX switch (Hybrid ↔ dGPU-only) changes ignored | `supergfxd` needs reboot to apply | After `supergfxctl -m <mode>` always reboot; only `Hybrid` ↔ `Integrated` can be switched by logout. |
| CUDA `torch.cuda.is_available()` returns `False` | Wrong wheel (CPU-only) or dGPU off | Reinstall torch with `--index-url https://download.pytorch.org/whl/cu130`; verify `supergfxctl -g` reports `Hybrid`. |

## Wi-Fi / networking

| Symptom | Most likely cause | Fix |
|---------|-------------------|-----|
| Wi-Fi (MT7921) drops after suspend | MediaTek firmware quirk | `/etc/modprobe.d/mt7921e.conf` → `options mt7921e disable_aspm=Y`; `sudo update-initramfs -u` |
| Wi-Fi is slow / stuck at 2.4 GHz | Regulatory domain wrong | `sudo iw reg set <CC>` (e.g. `IN`, `US`); persist with `crda` configuration. |
| Bluetooth devices won't pair | `bluez` service not started | `sudo systemctl enable --now bluetooth` |

## ASUS TUF-specific

| Symptom | Most likely cause | Fix |
|---------|-------------------|-----|
| Fans never ramp down / stuck loud | `asusd` default profile is Performance | `asusctl profile -P Quiet`; customize curves in `/etc/asusd/asusd.ron`. See [2.6](02-setup/06-asus-hardware.md). |
| Keyboard RGB does not work | Missing udev rules or `asusd` not running | `sudo systemctl status asusd`; check `asusctl led-mode static -c 0xff0000` works |
| AniMe Matrix / backlight flickers | Kernel driver version mismatch | Rebuild `asus-linux` kernel modules after kernel upgrade: re-run the source build from [2.6](02-setup/06-asus-hardware.md). |
| Battery charge threshold ignored after reboot | Charge limit not persisted | `asusctl -c 80` is persistent in newer versions; if stuck, edit `/etc/asusd/asusd.ron` directly. |

## Graphics / GL workloads

| Symptom | Most likely cause | Fix |
|---------|-------------------|-----|
| Gazebo Harmonic crashes on startup | OpenGL context fails on hybrid | Launch with `prime-run gz sim ...`; if still failing, set `LIBGL_ALWAYS_SOFTWARE=1` to confirm it's a GL driver issue. |
| RViz2 renders very slowly | Running on iGPU accidentally | `prime-run rviz2` or export `__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia` |
| Firefox hardware decoding not working | VAAPI driver missing | `sudo apt install mesa-va-drivers libva-drivers-all`; in `about:support` verify "Compositing: WebRender" |
| High-refresh monitor stuck at 60 Hz | X11 auto-detects wrong mode | `xrandr --output eDP-1 --rate 144` (or use System Settings → Display) |
| System Settings shows "Your graphics card does not support..." | Mesa for amdgpu missing | `sudo apt install --reinstall libgl1-mesa-dri libglx-mesa0 mesa-vulkan-drivers` |

## Containers / toolchain

| Symptom | Most likely cause | Fix |
|---------|-------------------|-----|
| VS Code extensions crash (Pylance, C/C++) | GLIBC mismatch because Code was installed as Snap | Uninstall Snap Code, install from Microsoft APT repo. See [3.2](03-dev-environment/02-embedded.md). |
| Podman rootless cannot mount overlay | kernel userns + overlay not configured | `sudo sysctl kernel.unprivileged_userns_clone=1`; confirm `fuse-overlayfs` installed |
| Distrobox GUI apps open on iGPU only | No `--additional-flags "--device /dev/dri"` | Recreate container with the flag or edit via `distrobox-edit`. See [3.1](03-dev-environment/01-containers.md). |
| ST-Link "Permission denied" | User not in `plugdev` yet | Re-login after running [Appendix A script](appendix-a-provisioning.md); `groups` should list `plugdev dialout` |
| `candump vcan0` shows nothing | vcan module not loaded at boot | `echo vcan \| sudo tee /etc/modules-load.d/vcan.conf` |
| `distrobox enter` hangs forever on first entry | Image pull in background | Watch `podman ps -a` and `podman images`; be patient on first run (several-GB Ubuntu image) |

## Filesystem / boot / dual-boot

| Symptom | Most likely cause | Fix |
|---------|-------------------|-----|
| BTRFS "no space left" despite df showing free | Unallocated vs metadata exhaustion | `sudo btrfs balance start -dusage=50 -musage=50 /`; schedule monthly |
| BTRFS scrub reports errors | Actual bit-rot or SSD wear | `sudo btrfs scrub status /`; if corrected, watch; if uncorrectable, restore affected file from Timeshift. |
| Windows loses ~1 minute at boot per session | Both ESPs advertise in firmware; Windows keeps resetting boot order | In `efibootmgr -v` keep Ubuntu entry first but `efibootmgr -n XXXX` only for the next boot. Or use F8 manually. |
| `journalctl` floods with ACPI errors | Firmware quirk, cosmetic | `sudo systemctl mask systemd-journald-audit.socket`; or ignore |
| After Windows update, Linux entry disappeared from boot menu | Windows overwrote EFI order | Boot with F8 → live USB → `sudo efibootmgr -c -d /dev/nvme1n1 -p 1 -L "Kubuntu" -l '\EFI\ubuntu\shimx64.efi'` |

## Debug triage workflow

When something goes wrong and you are not sure where:

1. **Snapshot first.** `sudo timeshift --create --comments "before-troubleshoot"` — so whatever you break during diagnosis is reversible.
2. **`dmesg -w` in one Konsole tab.** Watch for kernel messages as you reproduce the issue.
3. **`journalctl -f` in another.** Systemd journal with follow; narrow via `-u <unit>` or `-p 3` (errors only).
4. **Check the usual suspects:**
   - NVIDIA: `nvidia-smi`, `prime-run glxinfo -B | head -20`, `supergfxctl -g`.
   - Container: `podman ps -a`, `podman logs <name>`, `distrobox list`.
   - Network: `ip a`, `resolvectl status`, `ping -c 3 1.1.1.1`.
   - Disk: `lsblk`, `findmnt`, `sudo btrfs filesystem usage /`.
5. **Search upstream.**
   - ASUS hardware: [asus-linux.org](https://asus-linux.org/) forums and GitLab issues.
   - NVIDIA: [NVIDIA Linux forum](https://forums.developer.nvidia.com/c/gpu-unix-graphics/linux/148).
   - Kubuntu / KDE: `#kde` on Libera Chat, or KDE bugs.kde.org.
   - ROS 2: [answers.ros.org](https://answers.ros.org/), [Robotics Stack Exchange](https://robotics.stackexchange.com/).

## Hard reset workflows

### Full rollback to a clean state

```bash
# List Timeshift snapshots:
sudo timeshift --list

# Restore the one before the breakage:
sudo timeshift --restore --snapshot '2026-04-17_22-00-01'
# Guided; reboots when done.
```

### Re-install NVIDIA cleanly

```bash
sudo apt purge -y '^nvidia-.*' '^libnvidia-.*' 'cuda-*'
sudo apt autoremove --purge -y
sudo rm -rf /etc/modprobe.d/nvidia* /usr/share/modprobe.d/nvidia*
sudo update-initramfs -u
sudo reboot
# Then re-run Part 3.3.
```

### Nuke a Distrobox and recreate

```bash
distrobox stop ros2-jazzy
distrobox rm -f ros2-jazzy
# Optionally remove its home dir:
rm -rf ~/Containers/ros2-jazzy
# Recreate per 3.1.
```

### Regenerate all kernel entries (systemd-boot)

```bash
for k in $(ls /boot/vmlinuz-*); do
  v=$(basename "$k" | sed 's/vmlinuz-//')
  sudo kernel-install add "$v" "/boot/vmlinuz-$v"
done
sudo bootctl list
```

---

## When to ask for help vs keep digging

- **You have a concrete error message and a failing command?** → grep for the message in upstream issues, usually fixed in 10 minutes.
- **Intermittent issue, only happens after X hours?** → collect `journalctl -b > ~/journal.log` + `dmesg > ~/dmesg.log` + `nvidia-bug-report.sh` output, then post in the relevant forum.
- **The system is unbootable and you cannot reach a live session?** → F8 boot menu → live USB → chroot via the steps in [2.3.3](02-setup/03-install-partitioning.md) → fix, or restore Timeshift.

---

[Home](../README.md) · [← Previous: Appendix A](appendix-a-provisioning.md) · **Appendix B: Troubleshooting**
