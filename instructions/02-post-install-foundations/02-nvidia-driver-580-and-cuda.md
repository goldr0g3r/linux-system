[Home](../README.md) · [↑ 02 Post-Install](README.md) · [← Previous: 2.1 Debloat](01-debloat-snap-flatpak-pim.md) · **2.2 NVIDIA & CUDA** · [Next: 2.3 ASUS TUF →](03-asus-tuf-hybrid-gpu.md)

---

# 2.2 NVIDIA Driver 580 + Native `nvidia-cuda-toolkit` (New in 26.04)

Proprietary NVIDIA driver, matching CUDA toolkit, cuDNN 9, and the PRIME offload setup for your RTX 3050 Mobile hybrid-GPU. This is the single biggest change between 24.04 and 26.04: **CUDA now ships in the Ubuntu archive**, so the NVIDIA-APT-repo dance from `docs/03-dev-environment/03-ai-ml.md` is no longer required.

## Version baseline (as of 2026-04-23)

- **NVIDIA driver**: `nvidia-driver-580` — the 580 production branch, recommended by `ubuntu-drivers devices` on Kubuntu 26.04 for the RTX 3050 Mobile (GA107, Ampere).
- **CUDA Toolkit**: 13.2 — shipped natively as `nvidia-cuda-toolkit` in the Ubuntu 26.04 archive.
- **cuDNN**: 9.21 — shipped natively as `cudnn9-cuda-13`.

## Snapshot first

```bash
sudo timeshift --create --comments "before-nvidia-580 $(date -I)" --tags D
```

## 2.2.1 Install the proprietary NVIDIA driver

```bash
sudo apt update
sudo ubuntu-drivers devices
```

Expect a table like:

```
== /sys/devices/pci0000:00/0000:00:01.1/0000:01:00.0 ==
modalias : pci:v000010DEd00002583sv00001043sd000018ABbc03sc00i00
vendor   : NVIDIA Corporation
model    : GA107M [GeForce RTX 3050 Mobile]
driver   : nvidia-driver-580 - distro non-free recommended
driver   : nvidia-driver-580-open - distro non-free
driver   : nvidia-driver-575-server - distro non-free
driver   : nvidia-driver-575-server-open - distro non-free
driver   : xserver-xorg-video-nouveau - distro free builtin
```

Install the recommended proprietary driver:

```bash
sudo apt install -y nvidia-driver-580 libnvidia-gl-580 nvidia-settings nvidia-prime
```

### Why 580 proprietary and not 580-open

- `nvidia-driver-580-open` uses NVIDIA's open-source kernel module. Supported on Turing+ (your Ampere works). Advantages: smaller, MIT/GPL licence on the kernel part, easier to debug.
- `nvidia-driver-580` is the fully proprietary kernel module. Advantages: widest third-party-tooling compatibility (`supergfxctl` reads its state reliably, suspend/resume is battle-tested, older CUDA code never hits open-module corner cases).
- **On the TUF A17 specifically, pick proprietary 580.** The hybrid-GPU pipeline with `supergfxctl` on AMD iGPU + NVIDIA Ampere dGPU has a long track record with the proprietary module. The `-open` variant works but has historically had suspend-resume quirks on Renoir.

If a future 26.04 point release (say 26.04.2) promotes `-open` to the "recommended" label, revisit. For now, stick to proprietary.

## 2.2.2 Reboot

```bash
systemctl reboot
```

NVIDIA's kernel module needs a clean load. Rebooting is the cleanest way — `modprobe -r` on a GPU that is driving the display does not work.

## 2.2.3 Verify the driver is alive

After reboot, log in, open Konsole:

```bash
nvidia-smi
```

Expect a table with:

- **Driver Version**: `580.126.xx` (or higher).
- **CUDA Version**: `13.2` (this is the driver-reported capability, not the toolkit version — the toolkit comes next).
- One GPU listed: `NVIDIA GeForce RTX 3050 Laptop GPU`.
- Memory: 4096 MiB total, ~150 MiB used (a few Plasma/KWin bits).
- Temperature: 40–50 °C idle, fan 0 %.

If `nvidia-smi` says `No devices were found`, check `supergfxctl -g` (if installed — otherwise skip this; it's covered in [2.3](03-asus-tuf-hybrid-gpu.md)) to confirm you are in `Hybrid` mode, not `Integrated`. On a fresh install without `supergfxctl`, the dGPU defaults to being available, so `No devices` usually means a driver install issue. Check `dmesg | grep -i nvidia` and `journalctl -b -u nvidia-persistenced.service`.

### PRIME offload test

PRIME = "dGPU is idle by default, render-offload individual apps to it on demand." This saves 10+ W at idle on a TUF A17.

```bash
# Default renderer on the iGPU:
glxinfo | grep "OpenGL renderer"
# Expect: "AMD RENOIR (radeonsi, ...)"

# Explicitly render on the dGPU via prime-run:
prime-run glxinfo | grep "OpenGL renderer"
# Expect: "NVIDIA GeForce RTX 3050 Laptop GPU/PCIe/SSE2"
```

If `glxinfo: command not found`:

```bash
sudo apt install -y mesa-utils
```

### Wayland + NVIDIA sanity check

On the default Wayland Plasma session (26.04 default), `nvidia-smi` will NOT show Xorg in the process list because there is no X server. That is correct. KWin uses the NVIDIA driver via the EGL Wayland protocol directly.

```bash
# Confirm KWin is using the NVIDIA driver for Wayland:
journalctl -b -u plasma-kwin_wayland.service 2>/dev/null | grep -i nvidia | head -5
# Expect some "Loading EGL" or "Using nvidia" lines. No errors.
```

## 2.2.4 Install the CUDA toolkit (native, one line)

This is the big 26.04 improvement. No NVIDIA APT repo, no `cuda-keyring_1.1-1_all.deb` download.

```bash
sudo apt install -y nvidia-cuda-toolkit nvidia-cuda-toolkit-gcc

# Verify:
nvcc --version
# Expect: Cuda compilation tools, release 13.2, V13.2.xxx
```

- `nvidia-cuda-toolkit` installs to `/usr/lib/nvidia-cuda-toolkit/` with a wrapper that compiles against the Ubuntu-packaged gcc-13 (CUDA 13.2 supports gcc up to 13 cleanly; gcc-15 default needs a compat package).
- `nvidia-cuda-toolkit-gcc` installs the gcc-13 compatibility wrapper so `nvcc` finds a supported host compiler.

### Side-by-side with the NVIDIA APT repo (if you need CUDA 12)

If a project pins to PyTorch cu128 wheels (CUDA 12.x) — e.g. a legacy marine-robotics codebase or TensorFlow 2.15 — you can still add the NVIDIA APT repo:

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2604/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install -y cuda-toolkit-12-8
rm cuda-keyring_1.1-1_all.deb
```

Each toolkit lands in `/usr/local/cuda-<version>/`. Switch via `PATH` / `LD_LIBRARY_PATH`. Covered in [7.1](../07-ai-ml-workstation/01-cuda-13-cudnn-native.md).

## 2.2.5 Install cuDNN 9 for CUDA 13

```bash
sudo apt install -y cudnn9-cuda-13

# Verify:
dpkg -l | grep cudnn
# Expect: libcudnn9-cuda-13 9.21.x, libcudnn9-dev-cuda-13 9.21.x
```

The metapackage pulls runtime (`libcudnn9-cuda-13`) and dev headers (`libcudnn9-dev-cuda-13`). Only one cuDNN CUDA major can be installed at a time; if you later install `cudnn9-cuda-12` for a legacy project it will prompt to replace.

## 2.2.6 PATH additions for CUDA

The Ubuntu-packaged CUDA puts `nvcc` in `/usr/bin` directly (no PATH adjustment needed). However, if you also install the NVIDIA-APT-repo version for CUDA 12.x, add this to your shell rc file (`~/.zshrc` after [5.1](../05-web-development/01-shell-and-terminal.md), or `~/.bashrc` for now):

```bash
# Uncomment only if you have installed cuda-toolkit-12-X from the NVIDIA APT repo:
# export PATH="/usr/local/cuda-12.8/bin:$PATH"
# export LD_LIBRARY_PATH="/usr/local/cuda-12.8/lib64:${LD_LIBRARY_PATH:-}"
```

For the default path (native 26.04 `nvidia-cuda-toolkit`), nothing to add.

## 2.2.7 Verify CUDA with a toy matrix multiply

Quick sanity that the CUDA stack end-to-end works:

```bash
# Install the CUDA samples (small, ~40 MB):
sudo apt install -y nvidia-cuda-samples

# The deviceQuery sample is in /usr/share/nvidia-cuda-samples/samples/:
cd /tmp
cp -a /usr/share/nvidia-cuda-samples/samples/1_Utilities/deviceQuery .
cd deviceQuery
make
./deviceQuery
```

Expect output ending with:

```
deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 13.2, CUDA Runtime Version = 13.2, NumDevs = 1
Result = PASS
```

If `Result = PASS`, the driver + toolkit + cuDNN chain is live. PyTorch + CUDA comes in [7.2](../07-ai-ml-workstation/02-python-uv-pytorch-jax.md).

## 2.2.8 Known 26.04-specific NVIDIA quirks to be aware of

- **Wayland Explicit Sync is mandatory for flicker-free dGPU rendering.** Plasma 6.6 on 26.04 enables it by default when the driver reports support (580+). If Firefox or Chromium shows flicker during video playback, verify by opening KScreen settings → Compositor → ensure "Explicit sync" is ticked. More in [2.4](04-wayland-plasma-66-nvidia.md).
- **`nvidia-persistenced` can cause battery drain on idle.** 580 onwards is better than 550-era, but double-check after [2.3](03-asus-tuf-hybrid-gpu.md) that `powertop --html` shows the dGPU at `PC8` or deeper sleep state when idle. If not, a runtime PM fix is documented in [10-troubleshooting.md](../10-troubleshooting.md).
- **Kernel 7.0 + NVIDIA DKMS** uses the new `drm_panic` infrastructure; `journalctl -b` should be clean at boot. If you see `nvidia: loading out-of-tree module taints kernel`, that's informational, not an error — the proprietary module is always out-of-tree.
- **Secure Boot off** is assumed (from [1.2.6](../01-install/02-windows-prep-and-bios.md#126-secure-boot--turn-off-for-install)). If you turned Secure Boot back on at some point, the DKMS-built NVIDIA module needs MOK enrollment — documented in [10-troubleshooting.md](../10-troubleshooting.md) but not recommended for a laptop without a regulated-data threat model.

## 2.2.9 Take a snapshot

```bash
sudo timeshift --create --comments "post-nvidia-580-cuda $(date -I)" --tags D
```

Proceed to [2.3 ASUS TUF hybrid-GPU enablement](03-asus-tuf-hybrid-gpu.md).

---

[Home](../README.md) · [↑ 02 Post-Install](README.md) · [← Previous: 2.1 Debloat](01-debloat-snap-flatpak-pim.md) · **2.2 NVIDIA & CUDA** · [Next: 2.3 ASUS TUF →](03-asus-tuf-hybrid-gpu.md)
