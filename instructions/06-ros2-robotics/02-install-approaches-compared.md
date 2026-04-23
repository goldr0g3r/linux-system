[Home](../README.md) · [↑ 06 ROS 2](README.md) · [← Previous: 6.1 ROS 2 landscape](01-ros2-landscape-2026.md) · **6.2 Approaches compared** · [Next: 6.3 Distrobox bridge →](03-distrobox-bridge-jazzy-now.md)

---

# 6.2 Install Approaches Compared — native, Distrobox, Podman Compose, rocker, VM

The researched comparison. Five candidate approaches, scored on five axes, with the verdict.

## The five approaches

1. **Native apt on host** — `apt install ros-<distro>-desktop`. The "factory" way.
2. **Distrobox** — rootless Podman container with an Ubuntu base matching a ROS distro, tightly integrated with host.
3. **Podman Compose (direct)** — official ROS Docker images driven by `podman-compose up`.
4. **rocker** — NVIDIA's Docker-wrapper specifically for ROS with GPU / X11 passthrough.
5. **Full VM (virt-manager / QEMU)** — Ubuntu 24.04 guest running as a virtual machine.

## The five axes

- **Integration**: does the ROS environment see your home directory, USB devices (STM32, serial), display, audio, and udev events as if it were running on bare metal?
- **GUI performance**: can RViz2, Gazebo, and other OpenGL/Vulkan-using apps run at native frame rates?
- **Isolation**: how well does this approach protect your host from a broken ROS install, and vice versa?
- **Reproducibility**: can you stand up an identical environment on another machine from a pinned recipe?
- **Switching cost / learning curve**: how much does the developer have to learn beyond "I know how to `apt install`"?

## Scoring

| Approach             | Integration | GUI perf | Isolation | Reproducibility | Switch cost | Total | Notes                                                                                    |
| -------------------- | ----------: | -------: | --------: | --------------: | ----------: | ----: | ---------------------------------------------------------------------------------------- |
| **Native apt**       |         5/5 |      5/5 |       2/5 |             3/5 |         5/5 | 20/25 | Only when the ROS distro is built for your host Ubuntu. Not possible for Jazzy-on-26.04. |
| **Distrobox**        |         5/5 |      5/5 |       3/5 |             4/5 |         4/5 | 21/25 | Best overall for bridge phase. Near-native integration, zero measurable GPU penalty.     |
| **Podman Compose**   |         3/5 |      4/5 |       5/5 |             5/5 |         3/5 | 20/25 | Excellent for CI, reproducible demos. Overkill for interactive day-to-day.              |
| **rocker**           |         4/5 |      5/5 |       3/5 |             3/5 |         3/5 | 18/25 | Strong GPU passthrough but Distrobox covers 90 % of use cases with better ergonomics.    |
| **Full VM**          |         2/5 |      2/5 |       5/5 |             4/5 |         2/5 | 15/25 | Only when you need a different kernel.                                                   |

## Breakdown by axis

### Integration: who sees the host?

- **Native**: 5. You are the host. The ROS install sees everything.
- **Distrobox**: 5. By design, Distrobox mounts `$HOME` directly, shares `/dev` with the host (USB, serial, audio, video, DRI nodes), shares X11/Wayland socket, shares `systemd-resolved` DNS. `ros2 run rviz2 rviz2` opens a native-feeling window. `colcon build` produces binaries in your `$HOME` that you can `git commit`.
- **Podman Compose**: 3. By default, `/dev` is opaque; you have to explicitly pass `--device /dev/ttyUSB0`, mount volumes, and plumb DISPLAY. Working but a lot of YAML for integration features Distrobox gets for free.
- **rocker**: 4. `rocker` flags `--nvidia --x11 --user --home` give most of Distrobox's integration. Less automatic; you specify what you want each time.
- **Full VM**: 2. USB passthrough requires libvirt config. Display is through SPICE/VNC or passthrough (PCIe). Not natural.

### GUI performance: RViz2, Gazebo, RQt

- **Native**: 5. Direct DRM/KMS to your iGPU/dGPU. Whatever your bare metal does.
- **Distrobox**: 5. Passes `/dev/dri` for DRI/OpenGL and optionally `/dev/nvidia*` for CUDA; uses the host's X11/Wayland socket. RViz2 and Gazebo run within measurement noise of native.
- **Podman Compose**: 4. Same mechanics as Distrobox but you have to configure them — `--device /dev/dri --env DISPLAY --volume /tmp/.X11-unix`. Once configured, same as Distrobox.
- **rocker**: 5. Its whole raison d'être; `rocker --x11 --nvidia` handles GPU passthrough cleanly.
- **Full VM**: 2. Without GPU passthrough (VFIO + second GPU) you get software-rendered OpenGL. Gazebo on software rendering is a slideshow.

### Isolation: how well does approach X protect against approach Y?

- **Native**: 2. If a ROS install breaks your Python 3.12 apt state, or a ROS apt source conflicts with an Ubuntu apt source, your host is impacted. Happened to us on 22.04; likely to happen on 26.04 too.
- **Distrobox**: 3. Container's apt state is separate. Home is shared — so if a ROS script `rm -rf`'s your home, it's gone. Systemd services stay on the host. Middling isolation.
- **Podman Compose**: 5. Full container isolation. Volumes are explicit. A broken container doesn't touch the host.
- **rocker**: 3. Same as Distrobox roughly.
- **Full VM**: 5. Kernel-level isolation. Nothing gets through without explicit configuration.

### Reproducibility: pin it and share it

- **Native**: 3. Your `/etc/apt/sources.list.d/ros*.list` + installed packages list is reproducible if you document it. But drift is easy.
- **Distrobox**: 4. `distrobox create --image X --init-hooks` scripts are reproducible. Less "formal" than Compose.
- **Podman Compose**: 5. `docker-compose.yml` is a single commitable artefact. CI-friendly.
- **rocker**: 3. Flags-on-command-line are less reproducible than a YAML or script.
- **Full VM**: 4. VM image can be exported and imported. Clumsy for sharing over git.

### Switching cost: what the dev needs to learn

- **Native**: 5. `apt install` is universal.
- **Distrobox**: 4. One new command: `distrobox enter`. Rest is stock apt-inside-container.
- **Podman Compose**: 3. You learn Compose YAML, volumes, network modes, services.
- **rocker**: 3. One more tool layered on Docker.
- **Full VM**: 2. You learn libvirt, virt-manager, network bridges, SPICE, snapshots of the VM state.

## Verdict

Ordered by fitness for our situation:

1. **Distrobox with Ubuntu 24.04 base + Jazzy** for the bridge phase (today → 2026-05-22). 21/25.
2. **Native apt** once Lyrical ships on 2026-05-22 — the 2/5 isolation score is the price we pay for the 5/5 on everything else. Take Timeshift snapshots before every `apt install ros-*`.
3. **Podman Compose** as a secondary tool for CI / shipping demos / reproducing a setup on a colleague's machine.
4. **rocker** — not installed. Distrobox supersedes.
5. **Full VM** — not installed. Use only if you need a different kernel.

## Edge case: when a VM IS the right answer

- You are developing a kernel driver for a specific hardware that requires a different Linux version.
- You need to reproduce a customer's exact old-distro setup (e.g. Ubuntu 18.04 for a vehicle SDK that was last released 2019).
- You're testing a Windows-native ROS workflow.

In all three, `virt-manager` + QEMU + a UEFI guest is the tool. Documented in the [Appendix B troubleshooting matrix](../../docs/appendix-b-troubleshooting.md) if you ever need it; out of scope here.

## Edge case: Kubernetes / k3s / k8s for ROS

Some teams run ROS 2 nodes in Kubernetes pods on a cluster. Legitimate if you're deploying to a fleet of robots with centralized command. For a single dev laptop, it's an order of magnitude more complexity than you need — ignore.

## A note on Docker vs Podman

You may have noticed this guide never installs Docker — only Podman. Reasons:

- **Rootless by default.** Docker requires a root daemon; Podman runs containers as your user.
- **Systemd-integrated.** Podman Quadlet vs Docker Compose — Quadlet is native-systemd, better logs, better status.
- **`docker`-compatible CLI.** `alias docker=podman` works for 95 % of tutorials you'll read.
- **No daemon to maintain.** Podman containers are user processes; no `systemctl status dockerd` debugging.
- **Better for laptop use.** When you close the lid, Podman containers get paused with the kernel; Docker daemon can get confused.

If a particular tool requires Docker (rare in 2026 — most have added Podman support), install Docker alongside; they don't conflict.

## Conclusion

Proceed to [6.3 Distrobox bridge — Jazzy on Ubuntu 24.04 base](03-distrobox-bridge-jazzy-now.md) to set up the bridge phase ROS 2 environment today.

After 2026-05-22, proceed to [6.4 Native Lyrical](04-native-lyrical-after-may-22.md) for the host-level install.

---

[Home](../README.md) · [↑ 06 ROS 2](README.md) · [← Previous: 6.1 ROS 2 landscape](01-ros2-landscape-2026.md) · **6.2 Approaches compared** · [Next: 6.3 Distrobox bridge →](03-distrobox-bridge-jazzy-now.md)
