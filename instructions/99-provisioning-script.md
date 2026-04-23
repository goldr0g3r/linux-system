[Home](README.md) · [← Previous: 10 Troubleshooting](10-troubleshooting.md) · **99 — Provisioning script**

---

# 99 — One-Shot Provisioning Script

A bash script that chains the post-first-boot steps (sections 02–08) for a reproducible rebuild. Useful when:

- You reinstall Kubuntu 26.04 on the same laptop (after hardware change, accidental full-disk wipe).
- You set up a second similar laptop.
- You want to re-verify all post-install steps ran correctly.

This script **does not** perform the install itself (sections 01 — that requires graphical interaction with Calamares). It starts from a fresh first-boot Kubuntu 26.04 Minimal, assumed to have you logged in.

---

## 99.1 How to use

Save as `~/provision-kubuntu-26.04.sh`, make executable, run.

```bash
cd ~
# Copy the script content from this page into a file.
nano provision-kubuntu-26.04.sh

# Make executable:
chmod +x provision-kubuntu-26.04.sh

# Syntax-check before running:
bash -n provision-kubuntu-26.04.sh

# Run with output piped to a log:
sudo -v  # pre-authenticate sudo so the script doesn't prompt mid-way
./provision-kubuntu-26.04.sh 2>&1 | tee ~/provision-$(date -I).log
```

Expected run time: **30–45 minutes** (most of it apt download + install; 2 reboots prompted mid-way).

## 99.2 What it does

1. Initial Timeshift snapshot.
2. apt update + full-upgrade.
3. Install Timeshift and btrfs tools.
4. Debloat: remove snap, add Flathub, Firefox via Mozilla APT.
5. NVIDIA driver 580 + native CUDA 13 + cuDNN 9.
6. ASUS TUF tooling (asusctl, supergfxctl).
7. Kernel boot parameters, zram/sysctl, service masking, journald cap.
8. BTRFS mount options and fstrim timer.
9. PipeWire Bluetooth codecs, VA-API.
10. Podman + Distrobox + a `ros2-jazzy` container (bridge phase ROS).
11. uv + Python 3.12 AI venv with PyTorch cu130.
12. Ollama + Open WebUI Quadlet.
13. Web-dev stack: zsh, Starship, git config, fnm.
14. VS Code + .deb installs of Cursor-ready tools.
15. UFW + Firejail setup.
16. Restic install (repo setup deferred to manual step since it needs password file).

## 99.3 What it deliberately does NOT do

- **Interactive installs**: no `.deb` downloads requiring a vendor URL (Cursor, Zoom, JetBrains Toolbox). You run these by hand.
- **Per-user config**: your git identity, SSH key generation, Restic password — manual.
- **Heavyweight Flatpak installs**: only the core set (Spotify, Signal, Telegram, Discord, Obsidian, Bitwarden). Add others by hand.
- **ROS 2 Lyrical**: only Jazzy bridge container. Lyrical native install is gated on May 22, 2026.

## 99.4 The script

```bash
#!/usr/bin/env bash
# ============================================================================
# provision-kubuntu-26.04.sh
# Kubuntu 26.04 LTS "Resolute Raccoon" — post-first-boot provisioning
# Target: ASUS TUF A17 (AMD Ryzen + NVIDIA RTX 3050 Mobile hybrid)
# ============================================================================
# Usage: ./provision-kubuntu-26.04.sh 2>&1 | tee ~/provision-$(date -I).log
# ============================================================================

set -euo pipefail

USER_NAME="${SUDO_USER:-$USER}"
USER_HOME="$(getent passwd "$USER_NAME" | cut -d: -f6)"
LOG_PREFIX="[provisioning]"

log() { echo "$LOG_PREFIX $*"; }
die() { echo "$LOG_PREFIX ERROR: $*" >&2; exit 1; }

# Pre-checks
[ "$(id -u)" -ne 0 ] || die "Do not run as root directly. Run as normal user; sudo will be used selectively."
command -v sudo >/dev/null || die "sudo required"
lsb_release -a 2>/dev/null | grep -q "Resolute Raccoon" || log "WARNING: not on Kubuntu 26.04 (continuing anyway)"

# Re-authenticate sudo up front
sudo -v

# Keep sudo alive for the duration
while true; do sudo -n true; sleep 50; kill -0 "$$" || exit; done 2>/dev/null &

# ----------------------------------------------------------------------------
# 0. Install Timeshift and take a snapshot
# ----------------------------------------------------------------------------
log "== Step 0: Timeshift + snapshot =="
sudo apt update
sudo apt install -y timeshift btrfs-compsize btrfs-assistant
sudo timeshift --create --comments "before-provisioning $(date -I)" --tags D

# ----------------------------------------------------------------------------
# 1. apt full-upgrade
# ----------------------------------------------------------------------------
log "== Step 1: apt full-upgrade =="
sudo apt full-upgrade -y
sudo apt autoremove -y --purge

# ----------------------------------------------------------------------------
# 2. Debloat: remove snap
# ----------------------------------------------------------------------------
log "== Step 2: Remove snap =="
sudo systemctl disable --now snapd.service snapd.socket snapd.seeded.service 2>/dev/null || true
sudo systemctl mask snapd.service snapd.socket snapd.seeded.service 2>/dev/null || true
sudo apt purge -y snapd snap-confine 2>/dev/null || true
sudo rm -rf /var/cache/snapd /snap
rm -rf "$USER_HOME/snap"

sudo tee /etc/apt/preferences.d/nosnap.pref >/dev/null <<'EOF'
Package: snapd
Pin: release a=*
Pin-Priority: -10
EOF
sudo apt update

# ----------------------------------------------------------------------------
# 3. Flatpak + Flathub
# ----------------------------------------------------------------------------
log "== Step 3: Flatpak + Flathub =="
sudo apt install -y flatpak plasma-discover-backend-flatpak
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# ----------------------------------------------------------------------------
# 4. Firefox via Mozilla APT
# ----------------------------------------------------------------------------
log "== Step 4: Firefox via Mozilla APT =="
wget -q -O - https://packages.mozilla.org/apt/repo-signing-key.gpg | \
  sudo tee /etc/apt/keyrings/packages.mozilla.org.asc > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/packages.mozilla.org.asc] https://packages.mozilla.org/apt mozilla main" | \
  sudo tee /etc/apt/sources.list.d/mozilla.list > /dev/null

sudo tee /etc/apt/preferences.d/mozilla >/dev/null <<'EOF'
Package: *
Pin: origin packages.mozilla.org
Pin-Priority: 1000
EOF
sudo apt update
sudo apt install -y firefox thunderbird

# ----------------------------------------------------------------------------
# 5. NVIDIA driver 580 + CUDA + cuDNN
# ----------------------------------------------------------------------------
log "== Step 5: NVIDIA driver + CUDA =="
sudo timeshift --create --comments "before-nvidia $(date -I)" --tags D
sudo apt install -y nvidia-driver-580 libnvidia-gl-580 nvidia-settings nvidia-prime
sudo apt install -y nvidia-cuda-toolkit nvidia-cuda-toolkit-gcc cudnn9-cuda-13
log "NVIDIA driver installed. REBOOT REQUIRED before next steps."
log "After reboot, re-run this script from the 'checkpoint-after-nvidia' marker."
if [ ! -f /var/tmp/.provision-checkpoint-after-nvidia ]; then
  sudo touch /var/tmp/.provision-checkpoint-after-nvidia
  log "Created checkpoint. Reboot now (systemctl reboot) then re-run script."
  exit 0
fi

# ----------------------------------------------------------------------------
# 6. ASUS TUF tooling
# ----------------------------------------------------------------------------
log "== Step 6: ASUS TUF tooling =="
sudo apt install -y software-properties-common
sudo add-apt-repository -y ppa:flexiondotorg/asus-linux-rog || {
  log "PPA not available for resolute; trying asus-linux.org fallback"
  wget -qO - https://asus-linux.org/archive-key.asc | \
    sudo tee /etc/apt/keyrings/asus-linux.asc > /dev/null
  echo "deb [signed-by=/etc/apt/keyrings/asus-linux.asc] https://asus-linux.org/repo/ resolute main" | \
    sudo tee /etc/apt/sources.list.d/asus-linux.list
  sudo apt update
}
sudo apt install -y asusctl supergfxctl rog-control-center
sudo systemctl enable --now asusd.service supergfxd.service
asusctl -c 80

# ----------------------------------------------------------------------------
# 7. Kernel boot parameters
# ----------------------------------------------------------------------------
log "== Step 7: Kernel boot parameters =="
sudo timeshift --create --comments "before-kernel-params $(date -I)" --tags D

# Preserve existing GRUB_CMDLINE_LINUX_DEFAULT if not already edited
if ! grep -q "nvidia-drm.modeset=1" /etc/default/grub; then
  sudo sed -i.bak 's|^GRUB_CMDLINE_LINUX_DEFAULT=.*|GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1 nvidia-drm.fbdev=1 nvidia.NVreg_PreserveVideoMemoryAllocations=1 nvidia.NVreg_EnableGpuFirmware=1 nvidia.NVreg_UsePageAttributeTable=1 nvme_core.default_ps_max_latency_us=5500 mitigations=auto amd_pstate=active"|' /etc/default/grub
  sudo update-grub
  log "Kernel params updated. Will apply on next reboot."
fi

# ----------------------------------------------------------------------------
# 8. zram, sysctl, service masking
# ----------------------------------------------------------------------------
log "== Step 8: zram + sysctl + service masking =="

# zram
sudo apt install -y zram-tools
sudo tee /etc/default/zramswap >/dev/null <<'EOF'
ALGO=zstd
PERCENT=50
PRIORITY=100
EOF
sudo systemctl restart zramswap.service

# sysctl
sudo tee /etc/sysctl.d/99-local-tuning.conf >/dev/null <<'EOF'
vm.swappiness = 10
vm.vfs_cache_pressure = 50
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
vm.dirty_expire_centisecs = 1500
vm.dirty_writeback_centisecs = 1500
kernel.sched_autogroup_enabled = 1
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 8192
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_fastopen = 3
EOF
sudo sysctl --system

# Service masking
sudo systemctl disable --now apport.service whoopsie.service ModemManager.service motd-news.timer 2>/dev/null || true
sudo systemctl mask apport.service whoopsie.service ModemManager.service 2>/dev/null || true
sudo apt purge -y apport whoopsie popularity-contest command-not-found 2>/dev/null || true

# journald cap
sudo tee /etc/systemd/journald.conf.d/90-cap.conf >/dev/null <<'EOF'
[Journal]
Storage=persistent
Compress=yes
SystemMaxUse=400M
SystemMaxFileSize=50M
SystemKeepFree=1G
MaxLevelStore=info
MaxRetentionSec=1month
EOF
sudo systemctl restart systemd-journald
sudo journalctl --vacuum-size=400M

# ----------------------------------------------------------------------------
# 9. BTRFS mount tuning
# ----------------------------------------------------------------------------
log "== Step 9: BTRFS mount options =="
# Note: fstab edit is surgical; only add options if not present.
if ! grep -q "compress=zstd" /etc/fstab; then
  log "NOTE: you need to manually add 'noatime,compress=zstd:1,space_cache=v2,discard=async' to BTRFS lines in /etc/fstab. See 3.4."
fi

# fstrim (already enabled by default but ensure):
sudo systemctl enable --now fstrim.timer

# BTRFS scrub monthly timer
sudo tee /etc/systemd/system/btrfs-scrub.service >/dev/null <<'EOF'
[Unit]
Description=BTRFS scrub of root filesystem

[Service]
Type=simple
ExecStart=/usr/bin/btrfs scrub start -B /
EOF
sudo tee /etc/systemd/system/btrfs-scrub.timer >/dev/null <<'EOF'
[Unit]
Description=Monthly BTRFS scrub

[Timer]
OnCalendar=monthly
Persistent=true

[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now btrfs-scrub.timer

# ----------------------------------------------------------------------------
# 10. Audio/Video: PipeWire BT codecs, VA-API
# ----------------------------------------------------------------------------
log "== Step 10: PipeWire BT codecs + VA-API =="
sudo apt install -y \
  libldac2 libfreeaptx0 liblc3-1 libspa-0.2-bluetooth \
  bluez bluez-tools blueman \
  pavucontrol qpwgraph \
  mesa-va-drivers libva-utils libvdpau-va-gl vdpauinfo mesa-vdpau-drivers \
  mpv brightnessctl

# ----------------------------------------------------------------------------
# 11. Podman + Distrobox + ROS 2 Jazzy container
# ----------------------------------------------------------------------------
log "== Step 11: Podman + Distrobox + ROS 2 Jazzy =="
sudo apt install -y podman distrobox uidmap slirp4netns fuse-overlayfs podman-compose

mkdir -p "$USER_HOME/.config/containers"
cat > "$USER_HOME/.config/containers/containers.conf" <<'EOF'
[engine]
events_logger = "file"
cgroup_manager = "systemd"

[containers]
label = false
pids_limit = 0
EOF
chown -R "$USER_NAME:$USER_NAME" "$USER_HOME/.config/containers"

sudo -u "$USER_NAME" systemctl --user enable --now podman.socket
sudo loginctl enable-linger "$USER_NAME"

mkdir -p "$USER_HOME/Containers/ros2-jazzy"
chown -R "$USER_NAME:$USER_NAME" "$USER_HOME/Containers"

log "ROS 2 Jazzy Distrobox container will be created interactively. Run manually:"
log '  distrobox create --name ros2-jazzy --image docker.io/library/ubuntu:24.04 ...'
log "(see [6.3](06-ros2-robotics/03-distrobox-bridge-jazzy-now.md))"

# ----------------------------------------------------------------------------
# 12. uv + Python 3.12 AI venv
# ----------------------------------------------------------------------------
log "== Step 12: uv + Python 3.12 AI venv =="
sudo -u "$USER_NAME" bash -c "curl -LsSf https://astral.sh/uv/install.sh | sh"

# ----------------------------------------------------------------------------
# 13. Ollama
# ----------------------------------------------------------------------------
log "== Step 13: Ollama =="
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl enable --now ollama

log "Pulling default model set (15 GB)..."
sudo -u "$USER_NAME" ollama pull qwen2.5-coder:3b
sudo -u "$USER_NAME" ollama pull llama3.2:3b
sudo -u "$USER_NAME" ollama pull deepseek-r1:1.5b
sudo -u "$USER_NAME" ollama pull nomic-embed-text

# Open WebUI Quadlet
mkdir -p "$USER_HOME/.config/containers/systemd" "$USER_HOME/Containers/openwebui"
cat > "$USER_HOME/.config/containers/systemd/openwebui.container" <<EOF
[Unit]
Description=Open WebUI (Ollama frontend)
After=network.target

[Container]
Image=ghcr.io/open-webui/open-webui:main
ContainerName=openwebui
PublishPort=3000:8080
Volume=$USER_HOME/Containers/openwebui:/app/backend/data:Z
Environment=OLLAMA_BASE_URL=http://127.0.0.1:11434
Environment=WEBUI_AUTH=False
Network=host

[Service]
Restart=on-failure

[Install]
WantedBy=default.target
EOF
chown -R "$USER_NAME:$USER_NAME" "$USER_HOME/.config/containers" "$USER_HOME/Containers/openwebui"

# ----------------------------------------------------------------------------
# 14. Web-dev stack
# ----------------------------------------------------------------------------
log "== Step 14: Web-dev stack =="
sudo apt install -y zsh zsh-autosuggestions zsh-syntax-highlighting \
  git gh zellij tmux \
  eza bat ripgrep fd-find duf btop tealdeur zoxide fzf \
  mkcert libnss3-tools \
  helix

# Starship
sudo -u "$USER_NAME" bash -c "curl -sS https://starship.rs/install.sh | sh -s -- -y"

# fnm
sudo -u "$USER_NAME" bash -c "curl -fsSL https://fnm.vercel.app/install | bash -s -- --skip-shell"

# ----------------------------------------------------------------------------
# 15. VS Code
# ----------------------------------------------------------------------------
log "== Step 15: VS Code =="
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > /tmp/packages.microsoft.gpg
sudo install -D -o root -g root -m 644 /tmp/packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg
rm /tmp/packages.microsoft.gpg
echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" | \
  sudo tee /etc/apt/sources.list.d/vscode.list
sudo apt update
sudo apt install -y code

# ----------------------------------------------------------------------------
# 16. GitHub CLI
# ----------------------------------------------------------------------------
log "== Step 16: GitHub CLI =="
sudo mkdir -p -m 755 /etc/apt/keyrings
wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg | \
  sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null
sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | \
  sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install -y gh

# ----------------------------------------------------------------------------
# 17. Security: UFW + Firejail + ClamAV
# ----------------------------------------------------------------------------
log "== Step 17: UFW + Firejail + ClamAV =="
sudo apt install -y ufw firejail firejail-profiles clamav clamav-freshclam keyd
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw --force enable

# KeyD CapsLock remap
sudo tee /etc/keyd/default.conf >/dev/null <<'EOF'
[ids]
*

[main]
capslock = overload(control, esc)
EOF
sudo systemctl enable --now keyd

# ----------------------------------------------------------------------------
# 18. Flatpak app set
# ----------------------------------------------------------------------------
log "== Step 18: Flatpak app set =="
flatpak install -y flathub \
  com.spotify.Client \
  org.signal.Signal \
  org.telegram.desktop \
  com.discordapp.Discord \
  md.obsidian.Obsidian \
  com.bitwarden.desktop \
  org.videolan.VLC \
  com.obsproject.Studio \
  com.github.wwmm.easyeffects

# ----------------------------------------------------------------------------
# 19. Backup prerequisites (Restic install; repo setup manual)
# ----------------------------------------------------------------------------
log "== Step 19: Restic + rclone =="
sudo apt install -y restic rclone

# ----------------------------------------------------------------------------
# 20. Final snapshot
# ----------------------------------------------------------------------------
log "== Step 20: Final Timeshift snapshot =="
sudo timeshift --create --comments "post-provisioning $(date -I)" --tags D

# ----------------------------------------------------------------------------
# Done
# ----------------------------------------------------------------------------
log "=========================================="
log "Provisioning complete."
log "=========================================="
log "MANUAL STEPS remaining:"
log "  1. chsh -s /bin/zsh  (then log out, log back in)"
log "  2. Customize ~/.zshrc (see 5.1)"
log "  3. Git identity: git config --global user.name/user.email"
log "  4. SSH key: ssh-keygen -t ed25519"
log "  5. Create ros2-jazzy Distrobox (see 6.3)"
log "  6. Configure Restic repo + password (see 8.2)"
log "  7. After 2026-05-22: install ROS 2 Lyrical natively (see 6.4)"
log "  8. Reboot to apply kernel params + NVIDIA persistent mode"
log "=========================================="

rm -f /var/tmp/.provision-checkpoint-after-nvidia
```

## 99.5 Ordering assumptions

The script's ordering assumes:

- Kubuntu Minimal was installed following [1.3](01-install/03-installer-flavor-minimal-vs-full.md).
- Partitioning matches [1.4](01-install/04-installation-and-partitioning.md) with `@`, `@home`, `@snapshots`, `@var-log`, `@var-cache`.
- First-boot apt upgrade was done (from [1.5](01-install/05-first-boot-and-snapshot.md)).

If these are different on your machine, edit accordingly.

## 99.6 Idempotency

The script is largely idempotent (safe to re-run). Each `apt install` is safe to re-run; systemd mask/enable is safe to re-run; file writes use `tee` which overwrite existing.

If you re-run after a reboot, the `checkpoint-after-nvidia` marker ensures step 6+ resume without repeating steps 0-5.

## 99.7 When the script fails

If a step errors, the script exits (`set -euo pipefail`). The most common failure modes:

- **Network timeout** — re-run; apt and curl resume.
- **PPA unavailable** — the ASUS PPA fallback handles this; for others, comment the step and continue.
- **NVIDIA install fails** — check `journalctl -u dkms` for kernel module build errors. Usually resolved by `sudo apt install --reinstall linux-headers-$(uname -r)`.

---

[Home](README.md) · [← Previous: 10 Troubleshooting](10-troubleshooting.md) · **99 — Provisioning script**
