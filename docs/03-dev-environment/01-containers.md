[Home](../../README.md) · [↑ Dev Environment](README.md) · [← Previous: Part 3 Intro](README.md) · **3.1 Containers** · [Next: 3.2 Embedded →](02-embedded.md)

---

# 3.1 Containerization: Podman (rootless) + Distrobox

Distrobox is a wrapper around Podman that gives you **tightly host-integrated** containers. Each container sees your `$HOME`, your `/dev`, your X/Wayland display, your audio, and your udev events. It is the correct abstraction for running multiple ROS versions, testing on other distros, or isolating a language toolchain without a full VM.

## Why rootless Podman over Docker

- **No daemon.** Each container is a normal user process; there is no `dockerd` running as root.
- **No sudo required for container ops.** `podman run` works as your user.
- **Systemd-compatible.** `systemctl --user` manages long-running containers via Quadlet.
- **Drop-in CLI.** `alias docker=podman` and 95% of tutorials work unchanged.
- **Rootless by default.** Even if a container process escapes, it runs as your user — not root.

## Install

```bash
sudo apt install -y podman distrobox uidmap slirp4netns fuse-overlayfs

# Enable rootless Podman (no daemon; each container is a normal user process).
systemctl --user enable --now podman.socket
podman info | grep -E 'rootless|graphDriver'
```

The output should say `rootless: true` and `graphDriver.Name: overlay` (or `fuse-overlayfs`).

## Useful defaults

```bash
mkdir -p ~/.config/containers
tee ~/.config/containers/containers.conf >/dev/null <<'EOF'
[engine]
events_logger = "file"
cgroup_manager = "systemd"
[containers]
label = false
pids_limit = 0
EOF
```

`events_logger = "file"` avoids journald spam from each container. `cgroup_manager = "systemd"` uses cgroup v2 which is the default on Ubuntu 24.04.

## Three standing containers

```bash
# ROS 2 Humble on Ubuntu 22.04 — for legacy automotive/robotics packages still pinned to 22.04.
distrobox create --name ros2-humble \
  --image docker.io/library/ubuntu:22.04 \
  --home ~/Containers/ros2-humble \
  --additional-flags "--device /dev/dri --device /dev/bus/usb --device /dev/ttyUSB0 --device /dev/ttyACM0 --network=host"

# ROS 2 Jazzy on Ubuntu 24.04 — the current LTS line, matches your host, cleanest option for new projects.
distrobox create --name ros2-jazzy \
  --image docker.io/library/ubuntu:24.04 \
  --home ~/Containers/ros2-jazzy \
  --additional-flags "--device /dev/dri --device /dev/bus/usb --network=host"

# Embedded Fedora 40 — cross-distro sanity check for your STM32/KiCad work, and to evaluate Fedora's newer toolchain.
distrobox create --name embedded-fedora \
  --image registry.fedoraproject.org/fedora-toolbox:40 \
  --home ~/Containers/embedded-fedora \
  --additional-flags "--device /dev/bus/usb"
```

### Flags explained

- `--home ~/Containers/<name>` — each container gets its own home dir so their `.bashrc`, `.config`, and toolchain state do not collide with each other or the host.
- `--device /dev/dri` — passes iGPU + dGPU card devices for OpenGL/Vulkan-using GUI apps (RViz, Gazebo).
- `--device /dev/bus/usb` — lets the container see USB devices (for STM32 flashing, CAN adapters, Arduino).
- `--device /dev/ttyUSB0` / `--device /dev/ttyACM0` — the two common serial device names; add more if you use multiple adapters.
- `--network=host` — shares the host network namespace. Needed for ROS 2 multicast DDS discovery to work across containers and the host.

## First-time setup inside `ros2-jazzy`

```bash
distrobox enter ros2-jazzy

sudo apt update
sudo apt install -y software-properties-common curl locales
sudo locale-gen en_US.UTF-8 && sudo update-locale LANG=en_US.UTF-8
sudo add-apt-repository universe
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
  http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | \
  sudo tee /etc/apt/sources.list.d/ros2.list
sudo apt update
sudo apt install -y ros-jazzy-desktop ros-dev-tools python3-colcon-common-extensions
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc

# Make RViz2, gazebo, and ros2 CLI callable from the host's application menu:
distrobox-export --app rviz2
distrobox-export --bin /opt/ros/jazzy/bin/ros2 --export-path ~/.local/bin
```

Exit the container. From the host, `rviz2` now opens the Jazzy RViz2 transparently — Plasma shows it in the application menu, KWin tiles it, screen sharing works.

## Repeat for ros2-humble

Substitute `jazzy` with `humble` and `noble` with `jammy`:

```bash
distrobox enter ros2-humble
# ... same apt setup ...
sudo apt install -y ros-humble-desktop ros-dev-tools python3-colcon-common-extensions
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
distrobox-export --bin /opt/ros/humble/bin/ros2 --export-path ~/.local/bin
# Note: humble and jazzy both provide `ros2`; export to different names if you want both on the host PATH.
```

## Workflow helpers

Add to `~/.bashrc` (host):

```bash
alias dbe='distrobox enter'
alias dbl='distrobox list'
alias ros2j='distrobox enter ros2-jazzy'
alias ros2h='distrobox enter ros2-humble'
alias efed='distrobox enter embedded-fedora'
```

Konsole profiles (from [2.7](../02-setup/07-ui-ux.md)) can auto-enter a container on tab open — much smoother than typing `ros2j` every time.

## Quadlet: long-running containers as systemd services

Sometimes you want a container to run 24/7 as a service (local LLM server, Jellyfin, a dev database). Quadlet is Podman's native systemd integration.

Example — run Open WebUI (see [3.3](03-ai-ml.md)) as a user service:

```bash
mkdir -p ~/.config/containers/systemd
tee ~/.config/containers/systemd/openwebui.container >/dev/null <<'EOF'
[Unit]
Description=Open WebUI (Ollama front-end)

[Container]
Image=ghcr.io/open-webui/open-webui:main
ContainerName=openwebui
Network=host
Volume=%h/.config/openwebui:/app/backend/data
Environment=OLLAMA_BASE_URL=http://127.0.0.1:11434

[Service]
Restart=unless-stopped

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user start openwebui
# Survives reboots if you run: `loginctl enable-linger $USER`
```

## Verification

```bash
distrobox list
# Expect: ros2-humble, ros2-jazzy, embedded-fedora — all with status Up

# From host:
ros2 --version || which ros2
# Expect: exported ros2 binary pointing into a container

# Inside ros2-jazzy:
distrobox enter ros2-jazzy
ros2 topic list
# Expect: clean list (no errors about missing daemons)
```

Proceed to [3.2 Embedded & hardware toolchain](02-embedded.md).

---

[Home](../../README.md) · [↑ Dev Environment](README.md) · [← Previous: Part 3 Intro](README.md) · **3.1 Containers** · [Next: 3.2 Embedded →](02-embedded.md)
