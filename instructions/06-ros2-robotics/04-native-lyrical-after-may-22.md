[Home](../README.md) · [↑ 06 ROS 2](README.md) · [← Previous: 6.3 Distrobox bridge](03-distrobox-bridge-jazzy-now.md) · **6.4 Native Lyrical** · [Next: 07 AI/ML →](../07-ai-ml-workstation/README.md)

---

# 6.4 Native Phase — ROS 2 Lyrical Luth on Kubuntu 26.04 (post-2026-05-22)

**Do NOT execute this page until at least 2026-05-22.** Before that date, Lyrical packages do not exist in the ROS 2 apt repo. Use [6.3 Distrobox + Jazzy](03-distrobox-bridge-jazzy-now.md) in the meantime.

---

## When to run this page

- **2026-05-22** — Lyrical Luth general availability. The ROS 2 apt repo publishes `ros-lyrical-*` packages for `resolute` (Ubuntu 26.04 codename). **OK to install, but expect initial bugs — first release.**
- **2026-05-29** (a week later) — first bug-fix point release. Safer default.
- **2026-08-XX** — matches the Kubuntu 26.04.1 point release. Conservative target if you were following the `docs/` v2 verdict.

## Snapshot first

**Mandatory before any ROS 2 install on the host.**

```bash
sudo timeshift --create --comments "before-ros-lyrical $(date -I)" --tags D
```

## 6.4.1 Verify the ROS 2 apt repo has Lyrical

```bash
# Check what packages exist for your codename (should be "resolute" on 26.04):
. /etc/os-release
echo "Ubuntu codename: $UBUNTU_CODENAME"

# Browse the ROS apt repo manually:
curl -s http://packages.ros.org/ros2/ubuntu/dists/ | grep -i "lyrical\|resolute"
# Once Lyrical ships for resolute, you'll see appropriate metadata here.

# Alternative: check the manifest file for your codename:
curl -s "http://packages.ros.org/ros2/ubuntu/dists/${UBUNTU_CODENAME}/main/binary-amd64/Packages" | grep "^Package: ros-lyrical" | head -5
# Expect lines like "Package: ros-lyrical-desktop", "Package: ros-lyrical-ros-base", etc.
```

If those greps return nothing, **Lyrical is not yet published for your codename. Wait.**

## 6.4.2 Add the ROS 2 apt repo to the host

```bash
# Install prerequisites (most already present from other sections):
sudo apt install -y software-properties-common curl gnupg

# Enable universe:
sudo add-apt-repository -y universe

# Add ROS 2 signing key:
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

# Add the repo for your Ubuntu codename (should be resolute on 26.04):
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | \
  sudo tee /etc/apt/sources.list.d/ros2.list

sudo apt update
```

Check for errors in the apt update — if the repo doesn't have packages for your codename, you'll see `Not Found` for `binary-amd64/Packages`. That means Lyrical isn't published yet for resolute; stop here.

## 6.4.3 Install ROS 2 Lyrical

```bash
sudo apt install -y ros-lyrical-desktop ros-dev-tools python3-colcon-common-extensions

# Verify:
dpkg -l | grep ros-lyrical- | head -10

# Source it in your shell:
echo "source /opt/ros/lyrical/setup.bash" >> ~/.zshrc
source /opt/ros/lyrical/setup.bash

# First test:
ros2 --version
# Expect: Lyrical version info.

ros2 topic list
# Expect: blank (no nodes running).

# Hello world:
ros2 run demo_nodes_cpp talker &
ros2 run demo_nodes_py listener
# Watch the listener print Hello World.
# Ctrl+C to stop listener, fg to talker, Ctrl+C to stop.
```

## 6.4.4 Install Gazebo Ionic (the Lyrical-paired simulator)

Lyrical ships with Gazebo Ionic (assuming the Gazebo release cadence holds — may be Harmonic+ with a new minor):

```bash
sudo apt install -y ros-lyrical-ros-gz

# Test:
gz --version
gz sim shapes.sdf
# Gazebo Ionic opens. Should be faster than Harmonic with improved physics.
```

## 6.4.5 rosdep

```bash
sudo apt install -y python3-rosdep
sudo rosdep init
rosdep update
```

`rosdep` resolves non-ROS dependencies (apt packages) for ROS packages. Once initialised:

```bash
cd ~/work/ros2
rosdep install --from-paths src --ignore-src -r -y
# Installs apt deps for all packages in ~/work/ros2/src.
```

## 6.4.6 colcon workspace on the 26.04 host

Same workspace you used inside the Jazzy container — `~/work/ros2/` — is reusable. But colcon build artefacts (`build/`, `install/`, `log/`) are distro-specific. Segregate:

```bash
cd ~/work/ros2
mkdir -p build-lyrical install-lyrical log-lyrical

colcon build --build-base build-lyrical --install-base install-lyrical
source install-lyrical/setup.bash
ros2 run ...
```

Or use a separate workspace entirely:

```bash
mkdir -p ~/work/ros2-lyrical/src
cd ~/work/ros2-lyrical
# git clone or cp packages into src/, then:
colcon build
source install/setup.bash
```

Pick one; be consistent. I keep separate dirs per distro: `~/work/ros2-jazzy/` (inside the container) and `~/work/ros2-lyrical/` (on the host).

## 6.4.7 Deprecate the Jazzy Distrobox (gradually)

For the first week after installing Lyrical, keep the Jazzy Distrobox alive. Use it to:

- Confirm your existing Jazzy code still works (reproducibility check).
- Port packages to Lyrical as time permits.

After 2-4 weeks, if everything has ported, you can stop the container:

```bash
distrobox stop ros2-jazzy
```

Or remove entirely:

```bash
distrobox rm ros2-jazzy
rm -rf ~/Containers/ros2-jazzy
```

Keep `ros2-humble` around longer if you still have Humble codebases (EOL 2027).

## 6.4.8 Comparing install states

After installing native Lyrical, you may have four ROS environments:

| Where                                    | Distro   | Base OS     | Purpose                                                   |
| ---------------------------------------- | -------- | ----------- | --------------------------------------------------------- |
| `/opt/ros/lyrical` (host)                | Lyrical  | Kubuntu 26.04 | Primary development, new projects                       |
| `distrobox ros2-jazzy` (container)       | Jazzy    | Ubuntu 24.04| Legacy Jazzy projects, EOL 2029                           |
| `distrobox ros2-humble` (container)      | Humble   | Ubuntu 22.04| Legacy Humble projects, EOL 2027                          |
| (optional) `distrobox ros2-kilted`       | Kilted   | Ubuntu 24.04| Only if a specific vendor SDK needs it                    |

Switch between by:

- **Native**: `source /opt/ros/lyrical/setup.bash` (already in `.zshrc`).
- **Distrobox**: `distrobox enter ros2-<name>` — each container's shell sources its own ROS setup.

## 6.4.9 Verify LTS alignment

```bash
# Kubuntu version:
lsb_release -a | grep Description
# Expect: Kubuntu 26.04 LTS

# ROS distro:
printenv ROS_DISTRO
# Expect: lyrical

# Support calendars:
# Kubuntu 26.04: April 2031 (standard) / 2036 (Pro) / 2038 (Legacy)
# ROS 2 Lyrical: May 2031

echo "LTS alignment: both EOL 2031. Five-year stable support window."
```

## 6.4.10 Sanity checklist

- [ ] `nvidia-smi` works on the host.
- [ ] `ros2 topic list` returns silently (no error) on the host.
- [ ] `rviz2` opens a window on the host (may complain about no robot_description; that's normal).
- [ ] `gz sim` runs and renders hardware-accelerated.
- [ ] Your Jazzy Distrobox still works (`distrobox enter ros2-jazzy && ros2 topic list`).
- [ ] `rosdep` is initialised.
- [ ] `colcon build` works in a test workspace.

## 6.4.11 Snapshot

```bash
sudo timeshift --create --comments "post-ros-lyrical-native $(date -I)" --tags D
```

Proceed to [07 AI/ML Workstation](../07-ai-ml-workstation/README.md) if you haven't already set that up — CUDA was configured in [2.2](../02-post-install-foundations/02-nvidia-driver-580-and-cuda.md), so the AI/ML section is mostly venv + Ollama from here.

---

[Home](../README.md) · [↑ 06 ROS 2](README.md) · [← Previous: 6.3 Distrobox bridge](03-distrobox-bridge-jazzy-now.md) · **6.4 Native Lyrical** · [Next: 07 AI/ML →](../07-ai-ml-workstation/README.md)
