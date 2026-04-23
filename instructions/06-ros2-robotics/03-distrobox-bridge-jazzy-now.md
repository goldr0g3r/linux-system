[Home](../README.md) · [↑ 06 ROS 2](README.md) · [← Previous: 6.2 Approaches](02-install-approaches-compared.md) · **6.3 Distrobox bridge** · [Next: 6.4 Native Lyrical →](04-native-lyrical-after-may-22.md)

---

# 6.3 Bridge Phase — Distrobox + ROS 2 Jazzy on Ubuntu 24.04 Base

The recipe to use **today** (2026-04-23 through at least 2026-05-22 when Lyrical ships). Set up a rootless Podman + Distrobox environment with a Jazzy container backed by Ubuntu 24.04, tightly integrated with the 26.04 host.

## Prerequisite: Podman is installed and rootless-configured

Done in [5.5.1](../05-web-development/05-frameworks-and-databases.md#551-prerequisite-rootless-podman). If you skipped that page, do the rootless Podman setup now before continuing.

## Snapshot first

```bash
sudo timeshift --create --comments "before-distrobox-jazzy $(date -I)" --tags D
```

## 6.3.1 Install Distrobox

```bash
sudo apt install -y distrobox
distrobox --version
# Expect: 1.8.x or later.
```

## 6.3.2 Create the Jazzy container

```bash
mkdir -p ~/Containers/ros2-jazzy

distrobox create \
  --name ros2-jazzy \
  --image docker.io/library/ubuntu:24.04 \
  --home ~/Containers/ros2-jazzy \
  --additional-flags "\
    --device /dev/dri \
    --device /dev/bus/usb \
    --device /dev/nvidia0 \
    --device /dev/nvidiactl \
    --device /dev/nvidia-uvm \
    --device /dev/nvidia-modeset \
    --network=host" \
  --init-hooks "
    echo 'deb http://packages.ros.org/ros2/ubuntu noble main' > /etc/apt/sources.list.d/ros2.list
  "
```

Flags explained:

- `--name ros2-jazzy` — the identifier you'll use for `distrobox enter`.
- `--image docker.io/library/ubuntu:24.04` — base image. Ubuntu 24.04 "Noble" is where Jazzy targets.
- `--home ~/Containers/ros2-jazzy` — container's `$HOME` is isolated from your host's `$HOME`. The container sees a full, independent home directory; its `.bashrc`, `.config`, and installed Python venvs don't collide with the host or other containers.
- `--device /dev/dri` — DRI nodes for iGPU/dGPU. RViz2 and Gazebo need these for OpenGL.
- `--device /dev/bus/usb` — USB bus, for STM32 flashing, serial adapters, BlueRobotics hardware.
- `--device /dev/nvidia0 ... nvidia-modeset` — exposes the NVIDIA dGPU to the container for CUDA-in-ROS workflows.
- `--network=host` — shares the host's network namespace, so ROS 2 multicast DDS discovery works across host and containers. Required for ROS; breaks container isolation, but Distrobox's whole philosophy is "more integration, less isolation."

## 6.3.3 Enter the container and install ROS 2 Jazzy

```bash
distrobox enter ros2-jazzy
# You are now inside the container. Prompt changes subtly (username@ros2-jazzy).
```

Inside the container:

```bash
sudo apt update
sudo apt install -y software-properties-common curl locales

# Locale (ROS 2 requires UTF-8):
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8
export LANG=en_US.UTF-8

# Universe repo:
sudo add-apt-repository -y universe

# ROS 2 repo and key:
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | \
  sudo tee /etc/apt/sources.list.d/ros2.list

sudo apt update
sudo apt install -y ros-jazzy-desktop ros-dev-tools python3-colcon-common-extensions

# Source Jazzy on container shell:
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
source /opt/ros/jazzy/setup.bash

# Verify:
ros2 --version
ros2 topic list
# Expect: blank topic list (no nodes running yet).
```

## 6.3.4 Export ROS tools to the host

```bash
# Still inside the container:
distrobox-export --app rviz2
distrobox-export --bin /opt/ros/jazzy/bin/ros2 --export-path ~/.local/bin

exit  # leave the container
```

Now on the **host**:

- `rviz2` from KRunner or the Plasma menu launches the container's RViz2 seamlessly.
- `ros2 topic list` from a host terminal runs the container's `ros2` CLI.

Verify from host:

```bash
which ros2
# Expect: /home/<user>/.local/bin/ros2 (Distrobox-export shim)

ros2 --version
# Expect: Jazzy version info.
```

## 6.3.5 Install Gazebo Harmonic (or Ionic when available)

Gazebo is the simulator. Jazzy ships with Gazebo Harmonic by default.

```bash
distrobox enter ros2-jazzy

# Gazebo Harmonic binaries:
sudo apt install -y ros-jazzy-ros-gz

# Test:
gz --version
gz sim -v4 shapes.sdf
# Should open the Gazebo GUI with a scene of basic shapes.
```

If Gazebo opens but GPU-acceleration is off (software rendering, slideshow), verify:

```bash
# Inside the container:
glxinfo | grep "OpenGL renderer"
# Expect: the iGPU or dGPU renderer (not llvmpipe).

# If llvmpipe: the --device /dev/dri flag wasn't applied.
# Exit, delete the container (distrobox rm ros2-jazzy), re-create with explicit --device flags.
```

## 6.3.6 Set up a colcon workspace on the shared `~/work/`

```bash
# On the host:
mkdir -p ~/work/ros2/src
cd ~/work/ros2

# Enter the container:
distrobox enter ros2-jazzy
cd ~/work/ros2  # Same path works — $HOME is mounted differently but ~/work is at the same
                # path inside and outside the container because --home doesn't affect /home or
                # bind mounts. Note: the container's $HOME is ~/Containers/ros2-jazzy, BUT
                # Distrobox mounts your host $HOME at /var/home/<user> by default AND creates
                # symlinks so cd ~/work works.

# If ~/work doesn't resolve inside the container, bind-mount it:
exit
distrobox stop ros2-jazzy
distrobox create --replace \
  --name ros2-jazzy \
  --image docker.io/library/ubuntu:24.04 \
  --home ~/Containers/ros2-jazzy \
  --volume "$HOME/work:$HOME/work" \
  --additional-flags "\
    --device /dev/dri --device /dev/bus/usb \
    --device /dev/nvidia0 --device /dev/nvidiactl --device /dev/nvidia-uvm --device /dev/nvidia-modeset \
    --network=host"

distrobox enter ros2-jazzy

# Now ~/work is shared host<->container.

cd ~/work/ros2
ros2 pkg create --build-type ament_cmake hello_world
cd ..
colcon build
source install/setup.bash
ros2 run hello_world hello_world
# (will say "No executable found" because we haven't written one, but colcon ran — confirmed working.)
```

## 6.3.7 Konsole profile for auto-enter

In [4.6.6 Konsole profiles](../04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md#466-konsole-setup), we set up a "ROS-Jazzy" profile. Its Command should be:

```
/usr/bin/distrobox enter ros2-jazzy
```

Starting directory:

```
/home/<user>/work/ros2
```

Now opening Konsole → New Tab → "ROS-Jazzy" drops you directly into the container at the workspace.

## 6.3.8 Verify nvidia-smi works inside the container

```bash
distrobox enter ros2-jazzy
nvidia-smi
# Expect: same output as host — driver 580, RTX 3050 Laptop GPU, CUDA 13.2.
# If "No such file or directory", the --device /dev/nvidia* flags weren't applied.
# If "Failed to initialize NVML", you need to install the container toolkit; see below.
```

### NVIDIA Container Toolkit (for CUDA inside containers)

For CUDA inside the container to work, you need the NVIDIA Container Toolkit on the host:

```bash
# On the host (not inside the container):
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Configure Podman CDI:
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# Test from a throwaway container:
podman run --rm --device nvidia.com/gpu=all nvidia/cuda:13.2.0-base-ubuntu24.04 nvidia-smi
# Expect: nvidia-smi output from inside the container.
```

Now update your Distrobox creation to use CDI:

```bash
distrobox rm ros2-jazzy
distrobox create \
  --name ros2-jazzy \
  --image docker.io/library/ubuntu:24.04 \
  --home ~/Containers/ros2-jazzy \
  --volume "$HOME/work:$HOME/work" \
  --additional-flags "\
    --device nvidia.com/gpu=all \
    --device /dev/dri --device /dev/bus/usb \
    --network=host"
```

Then re-do the ROS install inside.

## 6.3.9 Multiple ROS distros side-by-side

If you also need Humble for legacy work:

```bash
distrobox create \
  --name ros2-humble \
  --image docker.io/library/ubuntu:22.04 \
  --home ~/Containers/ros2-humble \
  --volume "$HOME/work:$HOME/work" \
  --additional-flags "\
    --device nvidia.com/gpu=all \
    --device /dev/dri --device /dev/bus/usb \
    --network=host"

distrobox enter ros2-humble
# Install ros-humble-desktop following the same pattern as jazzy but with $UBUNTU_CODENAME=jammy.
exit
```

List all containers:

```bash
distrobox list
# ros2-jazzy, ros2-humble — both "Up"
```

Aliases in `~/.zshrc`:

```bash
alias ros2j='distrobox enter ros2-jazzy'
alias ros2h='distrobox enter ros2-humble'
alias dbe='distrobox enter'
alias dbl='distrobox list'
```

## 6.3.10 ROS 2 first hello-world

```bash
distrobox enter ros2-jazzy

# Teleop to verify ros-dev-tools:
ros2 run demo_nodes_cpp talker &
ros2 run demo_nodes_py listener
# Watch the listener print "I heard: Hello World" messages.

# Stop:
fg # to talker
Ctrl+C
```

## 6.3.11 Snapshot

```bash
# Back on the host:
sudo timeshift --create --comments "post-distrobox-jazzy $(date -I)" --tags D
```

You now have a working ROS 2 Jazzy development environment on Kubuntu 26.04. Use it for ~29 days while Lyrical bakes.

When Lyrical ships (2026-05-22), proceed to [6.4 Native Lyrical](04-native-lyrical-after-may-22.md).

---

[Home](../README.md) · [↑ 06 ROS 2](README.md) · [← Previous: 6.2 Approaches](02-install-approaches-compared.md) · **6.3 Distrobox bridge** · [Next: 6.4 Native Lyrical →](04-native-lyrical-after-may-22.md)
