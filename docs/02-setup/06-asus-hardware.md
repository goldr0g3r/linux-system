[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.5 Kernel/Boot](05-kernel-boot.md) · **2.6 ASUS Hardware** · [Next: 2.7 UI/UX →](07-ui-ux.md)

---

# 2.6 ASUS TUF A17 Hardware Enablement

The community-maintained [`asus-linux.org`](https://asus-linux.org/) project ships the two tools that turn a TUF / ROG laptop from "works in Linux" to "works *well* in Linux":

- **`asusctl`** — fan curves, keyboard RGB, battery charge limit, MUX switching, thermal profiles tied to the ASUS ACPI interface.
- **`supergfxctl`** — clean `Hybrid` / `Integrated` / `Vfio` / `AsusMuxDgpu` GPU-mode switching.

> **Important:** run this step **after** NVIDIA drivers are installed (see [Part 3.3](../03-dev-environment/03-ai-ml.md)). `supergfxctl` needs the NVIDIA module present to identify Hybrid vs Integrated states. If you have not done Part 3.3 yet, skip ahead, install NVIDIA, then return here.

## Option A (recommended): install via PPA

The community-maintained `ppa:mitchellaugustin/asusctl` packages recent `asusctl` and `supergfxctl` for Ubuntu 24.04. Easier and safer than a source build.

```bash
sudo add-apt-repository -y ppa:mitchellaugustin/asusctl
sudo apt update
sudo apt install -y asusctl supergfxctl

sudo systemctl enable --now asusd supergfxd
```

If the PPA lags behind a specific feature you need (common if the upstream project tags a release and it hasn't been packaged yet), use Option B below.

## Option B: install from upstream source

```bash
# Dependencies for Rust build + D-Bus / udev / system-tray plumbing.
sudo apt install -y curl cargo rustc pkg-config \
  libudev-dev libdbus-1-dev libclang-dev \
  libasound2-dev libayatana-appindicator3-dev \
  make git

mkdir -p ~/src
```

### asusctl

```bash
cd ~/src
git clone https://gitlab.com/asus-linux/asusctl.git
cd asusctl
git checkout $(git tag --sort=-v:refname | head -n1)   # latest tagged release
make
sudo make install
```

### supergfxctl

```bash
cd ~/src
git clone https://gitlab.com/asus-linux/supergfxctl.git
cd supergfxctl
git checkout $(git tag --sort=-v:refname | head -n1)
make
sudo make install

sudo systemctl enable --now asusd supergfxd
```

## Post-install: verify and configure (both options)

```bash
asusctl --version
supergfxctl --version

# Set a sane fan profile and battery charge limit:
asusctl profile -P Balanced        # or Quiet / Performance
asusctl -c 80                      # stop charging at 80% to preserve battery longevity
supergfxctl -m Hybrid              # PRIME render-offload, dGPU idles at 0W when unused
supergfxctl -g                     # show current mode
```

Optional GUI control panel:

```bash
flatpak install -y flathub org.asuslinux.rog-control-center
```

## What each tool gives you

### `asusctl` — daily knobs

| Command | Effect |
|---------|--------|
| `asusctl profile -P Quiet` | Lowest fan curve; quiet, throttles sustained loads |
| `asusctl profile -P Balanced` | Default; fans ramp with temp |
| `asusctl profile -P Performance` | Max fans, max sustained clock; use when compiling / training |
| `asusctl profile -l` | Show current profile |
| `asusctl -c 60` / `-c 80` / `-c 100` | Battery charge ceiling (60/80/100%). Use 60% when plugged in 24/7, 80% for daily, 100% for travel day. |
| `asusctl led-mode static -c 0x00ff00` | Keyboard backlight solid green |
| `asusctl led-mode breathing -c 0x00ff00` | Breathing pattern |
| `asusctl led-mode rainbow` | Rainbow wave |
| `asusctl led-bright 0-3` | Keyboard backlight brightness |

The GUI ROG Control Center exposes all of this as sliders and pickers.

### `supergfxctl` — GPU mode switching

| Command | Effect |
|---------|--------|
| `supergfxctl -m Hybrid` | Default. iGPU drives the display, dGPU idles at 0 W unless a process is prefixed with `prime-run`. |
| `supergfxctl -m Integrated` | Disables the NVIDIA driver entirely. Best battery life (1.5–2x). Requires logout/login. |
| `supergfxctl -m AsusMuxDgpu` | MUX switch to dGPU-only (if your TUF supports it; check `supergfxctl -s`). Max GPU performance for games/training. Requires reboot. |
| `supergfxctl -m Vfio` | Binds the dGPU to `vfio-pci` for PCI passthrough into a Windows VM. Advanced. |
| `supergfxctl -g` | Show current mode |
| `supergfxctl -s` | Show all supported modes for your specific laptop |

**Recommended daily pattern:**

- Plugged in at desk: `Hybrid` mode, `asusctl profile -P Balanced`.
- Compiling / training: `Hybrid`, `asusctl profile -P Performance`, prefix CUDA processes with `prime-run`.
- On battery, lecture / travel: `Integrated` + `Quiet`, charge limit 80%.

## Configuration files

After install, `asusctl` writes `/etc/asusd/asusd.ron` (custom fan curves, LED profiles) and `supergfxctl` writes `/etc/supergfxd.conf`. Both are readable, editable, and covered by the respective projects' GitLab READMEs.

## Verify

```bash
systemctl status asusd
systemctl status supergfxd
asusctl profile -l
supergfxctl -g
```

All four should report healthy and reasonable values.

## Troubleshooting

See [Appendix B](../appendix-b-troubleshooting.md) for:

- Fans never ramp down / stuck loud
- Keyboard RGB does not work
- MUX switch changes ignored
- `nvidia-smi` reports "No devices were found" (means supergfxctl is in Integrated mode — switch to Hybrid and reboot)

Proceed to [2.7 UI/UX tweaks for engineers](07-ui-ux.md).

---

[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.5 Kernel/Boot](05-kernel-boot.md) · **2.6 ASUS Hardware** · [Next: 2.7 UI/UX →](07-ui-ux.md)
