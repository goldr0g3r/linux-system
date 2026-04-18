[Home](../../README.md) · [← Previous: 2.7 UI/UX](../02-setup/07-ui-ux.md) · **Part 3: Dev Environment** · [Next: 3.1 Containers →](01-containers.md)

---

# Part 3 — The Developer Environment Blueprint

Reproducible developer toolchain for automotive, marine robotics, embedded, AI/ML, and web workflows. Every tool installs via APT / PPA / upstream repo / single binary — no Snap confinement around your compilers.

## Overview

| # | Topic | What you get | Time |
|---|-------|--------------|------|
| 3.1 | [Containerization: Podman (rootless) + Distrobox](01-containers.md) | Three standing containers (`ros2-humble`, `ros2-jazzy`, `embedded-fedora`) that share your `$HOME`, `/dev`, and display. | 20 min |
| 3.2 | [Embedded & hardware toolchain](02-embedded.md) | KiCad 9, `arm-none-eabi-gcc`, OpenOCD, ST-Link rules, PlatformIO, VS Code (native, not Snap). | 30 min |
| 3.3 | [AI/ML: NVIDIA 580 + CUDA 13.2 + cuDNN 9.21 + PyTorch via `uv`](03-ai-ml.md) | Working CUDA, PyTorch that reports the RTX 3050, Ollama + Open WebUI for local LLMs. | 30 min |
| 3.4 | [Next.js blog workflow](04-web.md) | `fnm` + `pnpm` + Vercel CLI, optional containerized Node workspace. | 10 min |
| 3.5 | [Learning roadmap for subsea robotics](05-learning-roadmap.md) | Ranked list of the subsystems you must master to transition credibly. | (ongoing) |

Total unattended time: ~90 minutes, excluding reboot for NVIDIA driver and the inevitable coffee/cable-rummaging breaks.

## Execution order

The install order matters in two specific places:

1. **NVIDIA driver before ASUS hardware enablement:** `supergfxctl` (covered in [2.6](../02-setup/06-asus-hardware.md)) needs the NVIDIA kernel module present to distinguish Hybrid vs Integrated. If you skipped 2.6 earlier, do it *after* 3.3.
2. **Podman before Ollama's Open WebUI (in 3.3):** the local-LLM front-end runs as a Podman container.

Everything else is independent — pick the order you care about.

## Design philosophy

- **Host stays minimal.** Toolchains that touch hardware (`arm-none-eabi-gcc`, `openocd`, `candump`, KiCad, PlatformIO) go on the host — they need `/dev/ttyUSB*`, `/dev/can*`, udev access.
- **Containers isolate language/framework versions.** ROS distributions, experimental Python stacks, weird Node versions live in Distroboxes so upgrades do not destabilize the host.
- **`uv` for Python, `fnm` for Node.** Both are single-binary Rust tools that are 10–100x faster than their predecessors (`pip + venv + poetry`; `nvm`). They replace a pile of stateful config.
- **No Snap ever for development tools.** VS Code, JetBrains tools, Node, Python, browsers — all via APT / upstream / Flatpak. Confined Snap VS Code cannot reach host `/dev/ttyACM0` without manual plug overrides; that is unacceptable.

## What Part 3 does NOT cover

- **Real-time kernel setup.** Covered as a learning-roadmap item in [3.5](05-learning-roadmap.md); not a day-zero requirement.
- **Yocto / Buildroot.** Same — learning-roadmap item.
- **Specific ROS 2 packages beyond `ros-*-desktop`.** You will install `nav2`, `moveit2`, `gz-*` per project.
- **Kubernetes / k3s.** Listed as an optional application in [Part 4](../04-applications.md); not needed for the core workflow.

Proceed to [3.1 Containerization](01-containers.md).

---

[Home](../../README.md) · [← Previous: 2.7 UI/UX](../02-setup/07-ui-ux.md) · **Part 3: Dev Environment** · [Next: 3.1 Containers →](01-containers.md)
