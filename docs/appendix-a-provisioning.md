[Home](../README.md) · [← Previous: Part 4 Applications](04-applications.md) · **Appendix A: Provisioning Script** · [Next: Appendix B →](appendix-b-troubleshooting.md)

---

# Appendix A — One-Shot Provisioning Script

A single, idempotent-where-possible bash script that chains all the post-install steps from [Parts 2 and 3](02-setup/README.md). Run it after your first reboot into Kubuntu.

## How to use

1. Complete a fresh Kubuntu 24.04 LTS install per [Part 2.1–2.3](02-setup/README.md). First-login complete.
2. Save the script below as `~/provision.sh`:

    ```bash
    nano ~/provision.sh
    # paste the full script, save, exit
    chmod +x ~/provision.sh
    ```

3. Run it:

    ```bash
    ~/provision.sh
    ```

4. When it finishes, **reboot**, then complete the manual steps it cannot do automatically:
   - **[2.6 ASUS hardware](02-setup/06-asus-hardware.md)** — `asusctl` / `supergfxctl` source build.
   - **[2.7 UI/UX](02-setup/07-ui-ux.md)** — Konsole profiles, Dolphin panels, Activities, KRunner plugins (all GUI-driven).
   - **[3.3 AI/ML](03-dev-environment/03-ai-ml.md)** — finish with `uv venv` + PyTorch (the script installs the OS-level NVIDIA + CUDA + cuDNN; it does not create your personal Python venv).

## What it does

- Upgrades the base system.
- Installs core dev tools, CLI upgrades, embedded toolchain, VM stack, and the logic-analyser packages.
- Enables Flathub.
- Purges snap completely (with priority pin).
- Prunes KDE PIM + Ubuntu telemetry leftovers.
- Masks slow-boot services.
- Configures `sysctl`, zram, fstrim.
- Installs ST-Link and PlatformIO udev rules.
- Adds KiCad 9 PPA + installs KiCad.
- Adds Microsoft APT repo + installs VS Code.
- Installs NVIDIA 580 driver + CUDA 13.2 + cuDNN 9.21.
- Installs `uv` and `fnm`.
- Installs a baseline set of Flatpak applications.
- Creates the `ros2-jazzy` and `ros2-humble` standing Distroboxes.
- Appends useful aliases + PATH exports to `~/.bashrc`.

## The script

```bash
#!/usr/bin/env bash
set -euo pipefail

log() { printf '\n\033[1;34m==> %s\033[0m\n' "$*"; }

[[ $EUID -eq 0 ]] && { echo "Run as your normal user, not root. Script uses sudo where needed."; exit 1; }

log "Update base system"
sudo apt update
sudo apt full-upgrade -y

log "Core utilities"
sudo apt install -y \
  build-essential cmake ninja-build make pkg-config \
  git git-lfs curl wget ca-certificates gnupg lsb-release \
  htop btop nvtop iotop \
  ripgrep fd-find fzf bat duf git-delta \
  tmux \
  zram-tools \
  neovim \
  flatpak plasma-discover-backend-flatpak \
  timeshift \
  podman distrobox uidmap slirp4netns fuse-overlayfs \
  wireshark can-utils gtkterm cutecom moserial picocom minicom \
  gcc-arm-none-eabi gdb-multiarch binutils-arm-none-eabi \
    libnewlib-arm-none-eabi libnewlib-nano-arm-none-eabi \
    openocd stlink-tools srecord \
  qemu-kvm libvirt-daemon-system virt-manager \
  sigrok pulseview sigrok-cli sigrok-firmware-fx2lafw

log "Flathub"
sudo flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo

log "Remove snap"
sudo systemctl disable --now snapd.service snapd.socket snapd.seeded.service 2>/dev/null || true
sudo apt purge -y snapd || true
sudo apt-mark hold snapd
sudo rm -rf /var/cache/snapd ~/snap /snap
sudo tee /etc/apt/preferences.d/no-snap.pref >/dev/null <<'EOF'
Package: snapd
Pin: release a=*
Pin-Priority: -10
EOF

log "KDE PIM / Ubuntu telemetry prune"
sudo apt purge -y \
  akonadi-backend-mysql akonadi-server akregator kontact kmail2 korganizer kaddressbook \
  kmines kpatience ksudoku kmahjongg elisa \
  apport apport-gtk whoopsie popularity-contest ubuntu-report 2>/dev/null || true
sudo apt autoremove --purge -y

log "Mask slow boot services"
sudo systemctl mask \
  NetworkManager-wait-online.service \
  plymouth-quit-wait.service \
  motd-news.service motd-news.timer \
  apt-news.service esm-cache.service 2>/dev/null || true

log "sysctl tuning"
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

log "zram"
sudo tee /etc/systemd/zram-generator.conf >/dev/null <<'EOF'
[zram0]
zram-size = min(ram / 2, 8192)
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
EOF

log "fstrim + Podman user socket"
sudo systemctl enable --now fstrim.timer
systemctl --user enable --now podman.socket

log "User groups (logout/login after script to activate)"
sudo usermod -aG plugdev,dialout,kvm,libvirt,wireshark "$USER"

log "udev rules: ST-Link + PlatformIO"
sudo tee /etc/udev/rules.d/49-stlinkv2.rules >/dev/null <<'EOF'
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="3748", MODE="0660", GROUP="plugdev", TAG+="uaccess", SYMLINK+="stlinkv2_%n"
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374b", MODE="0660", GROUP="plugdev", TAG+="uaccess", SYMLINK+="stlinkv2-1_%n"
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374e", MODE="0660", GROUP="plugdev", TAG+="uaccess", SYMLINK+="stlinkv3_%n"
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374f", MODE="0660", GROUP="plugdev", TAG+="uaccess", SYMLINK+="stlinkv3_%n"
EOF
sudo curl -fsSL https://raw.githubusercontent.com/platformio/platformio-core/master/platformio/assets/system/99-platformio-udev.rules \
  -o /etc/udev/rules.d/99-platformio-udev.rules
sudo udevadm control --reload-rules
sudo udevadm trigger

log "KiCad 9 PPA"
sudo add-apt-repository -y ppa:kicad/kicad-9.0-releases
sudo apt update
sudo apt install -y kicad kicad-libraries kicad-footprints kicad-symbols kicad-templates kicad-packages3d

log "VS Code (APT)"
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | sudo tee /usr/share/keyrings/vscode.gpg >/dev/null
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/vscode.gpg] https://packages.microsoft.com/repos/vscode stable main" | sudo tee /etc/apt/sources.list.d/vscode.list
sudo apt update
sudo apt install -y code

log "NVIDIA 580"
sudo apt install -y nvidia-driver-580 libnvidia-gl-580 nvidia-settings nvidia-prime

log "CUDA 13.2 + cuDNN 9.21"
wget -q https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb -O /tmp/cuda-keyring.deb
sudo dpkg -i /tmp/cuda-keyring.deb
sudo apt update
sudo apt install -y cuda-toolkit-13-2 cudnn9-cuda-13
rm /tmp/cuda-keyring.deb

log "Python: uv"
curl -LsSf https://astral.sh/uv/install.sh | sh

log "Node: fnm + corepack"
curl -fsSL https://fnm.vercel.app/install | bash

log "Flatpak baseline apps"
flatpak install -y flathub \
  org.mozilla.Thunderbird \
  org.signal.Signal \
  org.keepassxc.KeePassXC \
  md.obsidian.Obsidian \
  org.zotero.Zotero \
  net.ankiweb.Anki \
  com.github.johnfactotum.Foliate \
  com.obsproject.Studio \
  org.blender.Blender \
  com.usebottles.bottles \
  org.mavlink.qgroundcontrol \
  rest.insomnia.Insomnia \
  io.dbeaver.DBeaverCommunity \
  com.spotify.Client \
  us.zoom.Zoom \
  com.slack.Slack \
  com.discordapp.Discord

log "Create standing Distroboxes"
distrobox create -Y --name ros2-jazzy \
  --image docker.io/library/ubuntu:24.04 \
  --home "$HOME/Containers/ros2-jazzy" \
  --additional-flags "--device /dev/dri --device /dev/bus/usb --network=host" || true

distrobox create -Y --name ros2-humble \
  --image docker.io/library/ubuntu:22.04 \
  --home "$HOME/Containers/ros2-humble" \
  --additional-flags "--device /dev/dri --device /dev/bus/usb --network=host" || true

log "Shell additions"
{
  echo
  echo '# --- engineer additions ---'
  echo 'export PATH="$HOME/.local/bin:$HOME/.cargo/bin:$PATH"'
  echo 'export PATH="/usr/local/cuda-13.2/bin${PATH:+:${PATH}}"'
  echo 'export LD_LIBRARY_PATH="/usr/local/cuda-13.2/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}"'
  echo 'eval "$(fnm env --use-on-cd)"'
  echo 'alias ll="ls -alF"'
  echo 'alias gst="git status"'
  echo 'alias glog="git log --graph --abbrev-commit --decorate --oneline --all"'
  echo 'alias dbe="distrobox enter"'
  echo 'alias dbl="distrobox list"'
  echo 'alias ros2j="distrobox enter ros2-jazzy"'
  echo 'alias ros2h="distrobox enter ros2-humble"'
} >> ~/.bashrc

log "Done. Reboot required for NVIDIA + group membership."
log "After reboot: install asusctl/supergfxctl (see Part 2.6), then nvidia-smi && prime-run glxinfo | grep renderer"
```

## What the script does NOT do

- **BTRFS subvolume layout** — do that manually in [2.3](02-setup/03-install-partitioning.md) before you ever run this script, because it needs a live USB.
- **asusctl / supergfxctl** — source build, covered in [2.6](02-setup/06-asus-hardware.md).
- **KDE UI tweaks** — GUI-driven in [2.7](02-setup/07-ui-ux.md).
- **Inside-container ROS 2 install** — the containers are created empty; run the apt commands from [3.1](03-dev-environment/01-containers.md) inside each one.
- **PyTorch venv** — covered in [3.3](03-dev-environment/03-ai-ml.md); script installs the OS-level stack only.

## Re-running safety

The script uses `-y` flags and `|| true` on purges, so re-running on an already-provisioned system is safe — most commands become no-ops. The two non-idempotent appends are:

1. `~/.bashrc` — running twice duplicates the `# --- engineer additions ---` block. Remove the duplicate manually if you re-run.
2. `/etc/apt/sources.list.d/*.list` — `add-apt-repository` de-dupes automatically; the Microsoft VS Code `echo` over-writes each time, which is fine.

Proceed to [Appendix B — Troubleshooting](appendix-b-troubleshooting.md) if anything goes sideways.

---

[Home](../README.md) · [← Previous: Part 4 Applications](04-applications.md) · **Appendix A: Provisioning Script** · [Next: Appendix B →](appendix-b-troubleshooting.md)
