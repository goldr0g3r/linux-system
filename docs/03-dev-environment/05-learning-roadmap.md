[Home](../../README.md) · [↑ Dev Environment](README.md) · [← Previous: 3.4 Web](04-web.md) · **3.5 Learning Roadmap** · [Next: Part 4 →](../04-applications.md)

---

# 3.5 Learning Roadmap for Subsea Robotics

Ranked by transition leverage. Each item states *what* and *why it maps to subsea*. Your existing automotive HW/SW experience is the foundation — the items below are the delta that makes you credible for an ROV / AUV role.

**Order of attack:** items 1–5 are the hard skills a subsea employer will interview you on. Items 6–10 are adjacent depth. The **Adjacent foundations** at the bottom you pick up opportunistically.

---

## Primary skills (do these in order)

### 1. SocketCAN + `can-utils` + `can-isotp` kernel module

Your automotive CAN fluency transfers 1:1: many ROV/AUV vehicles use CANopen or a proprietary CAN dialect over the tether for low-bandwidth telemetry to thrusters and sensors.

```bash
sudo apt install can-utils
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set vcan0 up
```

**Master these tools:** `cansniffer`, `candump`, `cangen`, `cansend`, `isotpsend`, `isotprecv`.

**Reading:** [SocketCAN kernel docs](https://docs.kernel.org/networking/can.html), CANopen CiA 301 specification.

**Projects to build:**
- Spoof a thruster-controller telemetry feed on `vcan0` and write a ROS 2 node that parses it.
- Bridge a CAN dialect to ROS 2 messages using `ros2_socketcan`.

---

### 2. PREEMPT_RT real-time kernel

Subsea autopilot control loops (thruster PID, depth hold, station keeping) are hard-real-time; learning `chrt`, `cyclictest`, CPU isolation via `isolcpus`, and IRQ affinity is table stakes.

Ubuntu Pro (free for 5 personal machines) ships a real-time 6.8 kernel:

```bash
sudo pro attach <your-token>
sudo pro enable realtime-kernel
sudo reboot
uname -a  # should show -realtime suffix
```

**Or build from `linux-rt` sources** for newer mainline versions.

**Master these tools:** `chrt`, `taskset`, `cyclictest` (from `rt-tests` package), `perf`, `trace-cmd`, CPU isolation via `isolcpus=` kernel param.

**Verification target:** sub-100 µs worst-case scheduling latency under load, measured with `cyclictest -l 100000 -m -S -p 99`.

**Reading:** ["How fast is fast enough?" by the OSADL](https://www.osadl.org/), the `Documentation/rt-priority.txt` kernel doc.

---

### 3. ROS 2 Jazzy + Nav2 + Cyclone DDS tuning

ROS 2 is the standard for modern AUVs (MBARI, WHOI, BlueRobotics, most academic groups).

```bash
distrobox enter ros2-jazzy
# Install Cyclone DDS RMW implementation:
sudo apt install -y ros-jazzy-rmw-cyclonedds-cpp
echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
```

**Tune `CYCLONEDDS_URI` for lossy tether links** — increase fragment burst size, disable Nagle, pin the participant to the tether interface:

```xml
<!-- ~/cyclonedds.xml -->
<CycloneDDS>
  <Domain>
    <General>
      <NetworkInterfaceAddress>tether0</NetworkInterfaceAddress>
      <AllowMulticast>true</AllowMulticast>
    </General>
    <Internal>
      <MaxMessageSize>65500B</MaxMessageSize>
      <FragmentSize>4000B</FragmentSize>
    </Internal>
  </Domain>
</CycloneDDS>
```

```bash
export CYCLONEDDS_URI=file://$HOME/cyclonedds.xml
```

**Master:** Nav2 (path planning + recovery behaviours), `ros2 bag` (recording topic data for post-mission analysis), `tf2` (coordinate frames — critical for multi-sensor robots).

**Projects:**
- Run the Nav2 turtlebot3 demo inside `ros2-jazzy`.
- Record a bag of a Gazebo mission, replay it in RViz2.

---

### 4. MAVLink + Ardusub + QGroundControl

Ardusub (a fork of ArduPilot for ROVs) powers the BlueROV2 and many custom vehicles.

```bash
# Host-side Python MAVLink tools:
uv pip install pymavlink pymavlink-examples

# QGroundControl:
flatpak install -y flathub org.mavlink.qgroundcontrol
```

**Master:** the MAVLink XML message definitions, `pymavlink` for scripting, message-rate configuration, parameter protocol, mission protocol (waypoint upload/download).

**Reading:** [Ardusub documentation](https://www.ardusub.com/), [MAVLink Developer Guide](https://mavlink.io/en/).

**Projects:**
- SITL (software-in-the-loop) Ardusub simulation — fly a virtual BlueROV2 in Gazebo from QGroundControl.
- Write a Python script that commands a Lawnmower mission and logs telemetry.

---

### 5. Gazebo Harmonic + `dave_simulation` + Stonefish

DAVE (Dynamic Autonomous Virtual Environment) adds hydrodynamic plugins for buoyancy, drag, thruster models. Stonefish is a lighter-weight C++ simulator specifically for AUVs. Either lets you test mission code without getting wet.

```bash
distrobox enter ros2-jazzy
sudo apt install -y gz-harmonic
# Install dave_simulation from source:
mkdir -p ~/ros2_ws/src && cd ~/ros2_ws/src
git clone --recursive https://github.com/Field-Robotics-Lab/dave.git
cd ~/ros2_ws
colcon build
```

**Master:** SDF world files, thruster plugins, buoyancy plugins, sensor models (DVL, multibeam sonar, underwater camera).

**Projects:**
- Run the DAVE ocean_world with a BlueROV2 model.
- Simulate a transponder array and use it to pose-estimate an AUV.

---

## Depth skills (items 6–10)

### 6. Yocto Project + Buildroot

Vehicle computers are commonly custom Linux images on NVIDIA Jetson / NXP i.MX8 / Raspberry Pi CM4. Build a minimal image with a known-good kernel, your ROS packages, and nothing else.

```bash
# kas-based Yocto workflow runs cleanly inside a Distrobox.
distrobox create --name yocto --image docker.io/library/ubuntu:24.04 --home ~/Containers/yocto
distrobox enter yocto
sudo apt install -y python3-pip gawk wget git diffstat unzip texinfo gcc build-essential chrpath socat cpio file
uv pip install kas
# Start from an existing config, e.g. meta-ros Poky + meta-ros2-humble.
```

**Reading:** [Yocto Mega-Manual](https://docs.yoctoproject.org/).

---

### 7. GTSAM + `robot_localization` EKF + Kalibr

Underwater state estimation is sensor fusion: DVL (Doppler Velocity Log) + IMU + depth sensor + USBL/LBL acoustic position fixes.

- **GTSAM** does factor-graph SLAM (the modern, globally-consistent approach).
- **`robot_localization`** is the ROS 2 EKF (simpler, good for runtime navigation).
- **Kalibr** calibrates your sensor extrinsics (IMU-to-camera transforms, etc.).

This is the **intellectually hardest and highest-leverage** item on the list for marine robotics. The field is converging on factor-graph methods; being fluent here is a distinguishing skill.

**Reading:** Frank Dellaert's GTSAM tutorials, Tim Barfoot's "State Estimation for Robotics" (free PDF).

---

### 8. EtherCAT on Linux — IgH EtherLab master

Multi-thruster ROVs and work-class vehicles increasingly use EtherCAT for deterministic distributed I/O. Requires PREEMPT_RT (item 2). `IgH EtherCAT Master` is the canonical open-source master stack; Acontis EC-Master is commercial.

**Hardware needed:** a cheap EtherCAT slave (Beckhoff EL1008, or LAN9252-based module) + a PC with a compatible NIC (Intel i210, i219, i225).

**Reading:** [IgH EtherLab docs](https://www.etherlab.org/en/ethercat/).

---

### 9. `systemd-networkd` + VLANs + link aggregation

Vehicles have multiple networks (tether uplink, payload LAN, sensor LAN, onboard safety LAN). Drop NetworkManager on the vehicle and use `systemd-networkd` for reproducibility.

**Master:** `.network` / `.netdev` files, VLANs, bridges, bonding (LACP), multi-path routing.

**Reading:** `man systemd.network`, `man systemd.netdev`.

---

### 10. MOOS-IvP + OpenCPN

MOOS-IvP is MIT's autonomy architecture for marine vehicles — older than ROS 2, still widely used in academic AUV work (MIT LAMSS). OpenCPN is the open-source chart-plotter stack for the surface-vessel / topside console.

Good to have in the toolbelt for a marine robotics employer conversation. Not a priority — learn if you are interviewing with a MOOS-shop (MIT, Thayer School, some defence primes).

```bash
sudo apt install -y opencpn
# MOOS-IvP builds from source; follow https://oceanai.mit.edu/moos-ivp/
```

---

## Adjacent foundations (do these in parallel with 1–5)

- **`tmux` or `zellij`** — you will SSH into vehicles constantly. Persistent sessions are non-negotiable. See [Part 4](../04-applications.md).
- **`mosh`** — resilient SSH over lossy links; tether loss tolerance.
- **`nftables`** — replace `iptables`. Firewall the vehicle-side networks.
- **`Wireshark` + `tcpdump` + `tshark`** — CAN bus analysis, DDS traffic inspection, tether link capture.
- **`pyserial` + `pymodbus`** — most oceanographic sensors (CTDs, DVLs, altimeters) speak RS-232/RS-485 with proprietary ASCII or Modbus RTU dialects.
- **Docker Compose / `podman-compose` / Quadlet** — many vehicle stacks are composed images; know both the old Compose and the new systemd-native Quadlet.

---

## Suggested pace

**Six months to basic credibility:**
- Month 1: item 1 (CAN) + item 3 (ROS 2 Jazzy basics).
- Month 2: item 2 (real-time) + item 4 (MAVLink/Ardusub SITL).
- Month 3: item 5 (Gazebo + DAVE simulation of a BlueROV2 mission).
- Month 4: item 7 (sensor fusion — start with `robot_localization`, graduate to GTSAM).
- Month 5: item 6 (Yocto — build a minimal Jetson image for a mock vehicle).
- Month 6: item 8 or 9 (EtherCAT or systemd-networkd), plus a portfolio project pulling 3–5 items together.

**Portfolio-worthy capstone options:**
- A BlueROV2 simulated in Gazebo, flown autonomously by Nav2 over a MAVLink bridge, with DVL+IMU fusion by `robot_localization`, deployed as a Yocto image.
- A CAN-to-MAVLink bridge that lets a CANopen thruster controller respond to MAVLink setpoints.
- A fault-injection harness: PREEMPT_RT + `tc netem` to simulate tether loss and measure how Nav2 recovers.

Any of these on a public GitHub repo with a 5-minute YouTube demo beats any resume bullet. The hardware to do them physically costs <$500 (one Pixhawk 6C + Ardusub + a BlueROV2 reference frame).

Proceed to [Part 4 — Application Stack](../04-applications.md) for the tools that make all of this bearable.

---

[Home](../../README.md) · [↑ Dev Environment](README.md) · [← Previous: 3.4 Web](04-web.md) · **3.5 Learning Roadmap** · [Next: Part 4 →](../04-applications.md)
