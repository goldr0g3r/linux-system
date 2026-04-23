[Home](../README.md) · [↑ 02 Post-Install](README.md) · [← Previous: 2.2 NVIDIA & CUDA](02-nvidia-driver-580-and-cuda.md) · **2.3 ASUS TUF hybrid-GPU** · [Next: 2.4 Wayland & Plasma →](04-wayland-plasma-66-nvidia.md)

---

# 2.3 ASUS TUF Hybrid-GPU Enablement — `asusctl`, `supergfxctl`

TUF-specific tooling from the `asus-linux.org` community. `asusctl` controls fan curves, keyboard RGB, battery charge limits, and power profiles via the firmware ACPI WMI. `supergfxctl` arbitrates between the Ryzen iGPU and the RTX 3050 dGPU (Integrated / Hybrid / Vfio / AsusMuxDgpu modes). Both are optional to strictly boot, but without them you're leaving hardware features unused and battery life on the table.

## What you get out of this page

- **Fn+F5 (fan profile cycle)** starts working.
- **Fn+F7/F8 (brightness)** starts working.
- **Fn+F10 (airplane mode / Wi-Fi toggle)** starts working.
- **Battery charge cap at 80 %** (preserves battery health for a 5-year LTS).
- **`supergfxctl -m Integrated/Hybrid/AsusMuxDgpu`** mode switching.
- **`asusctl profile -P Quiet/Balanced/Performance`** — cycles the Windows Armoury Crate equivalent.
- Plasma tray applet showing power profile and GPU mode.

## Snapshot first

```bash
sudo timeshift --create --comments "before-asus-tooling $(date -I)" --tags D
```

## 2.3.1 Add the `asus-linux.org` PPA

Luke Jones (`flukejones`) maintains [`ppa:flexiondotorg/asus-linux-rog`](https://launchpad.net/~flexiondotorg/+archive/ubuntu/asus-linux-rog) (the Mint Project-Launchpad-style PPA that also serves Ubuntu 26.04 from day zero). The main `asus-linux.org` website still points to the ROG-oriented repo, but for TUF laptops the same packages work.

```bash
# Install the PPA add-apt-repository tool if not present (it is, in Minimal, but defensive):
sudo apt install -y software-properties-common

# Add the PPA (the "ppa:flexiondotorg/asus-linux-rog" name is canonical on 26.04):
sudo add-apt-repository -y ppa:flexiondotorg/asus-linux-rog

# Update:
sudo apt update
```

If `add-apt-repository` complains the PPA is not yet published for `resolute` (the 26.04 codename) — which can happen for a day or two on a brand-new LTS release — fall back to the `asus-linux.org` official repo:

```bash
# Fallback: asus-linux.org-maintained repo that explicitly publishes for 26.04:
wget -qO - https://asus-linux.org/archive-key.asc | \
  sudo tee /etc/apt/keyrings/asus-linux.asc > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/asus-linux.asc] https://asus-linux.org/repo/ resolute main" | \
  sudo tee /etc/apt/sources.list.d/asus-linux.list
sudo apt update
```

## 2.3.2 Install `asusctl` and `supergfxctl`

```bash
sudo apt install -y asusctl supergfxctl rog-control-center
```

- **`asusctl`** — CLI + daemon (`asusd.service`) for fan / keyboard / battery.
- **`supergfxctl`** — CLI + daemon (`supergfxd.service`) for GPU mode.
- **`rog-control-center`** — GUI frontend; optional but nicer than the CLI for day-to-day "set fan to quiet" flicks.

Enable the daemons:

```bash
sudo systemctl enable --now asusd.service supergfxd.service
```

Verify:

```bash
asusctl --version
supergfxctl --version

# State inspection:
asusctl
# Expect: summary block of current fan profile, battery limit, LED state.
supergfxctl -g
# Expect: "Hybrid" (the default mode after installing supergfxctl on a system with an NVIDIA driver already loaded).
```

## 2.3.3 Set the battery charge limit

A 5-year LTS cycle on a lithium-ion battery that is kept charged to 100 % loses 40–60 % of usable capacity by year 3. Cap at 80 % and the battery still has >80 % of original capacity by year 5 for normal use.

```bash
asusctl -c 80
# Expect: "Charge limit set to 80"

# Verify (asusctl stores this in /sys/class/power_supply/BAT*/charge_control_end_threshold):
cat /sys/class/power_supply/BAT*/charge_control_end_threshold
# Expect: 80
```

The setting is persistent (asusd restores it at boot). To change, just run `asusctl -c N` for any N from 20 to 100.

## 2.3.4 Fan profile and power profile

`asusctl` exposes three profiles matching the BIOS-level ACPI WMI profiles:

- **Quiet** — throttles CPU, low fan curve, longest battery life.
- **Balanced** — default, reasonable.
- **Performance** — uncaps CPU, aggressive fan curve, loud but fast.

```bash
# Cycle (like Fn+F5 does):
asusctl profile --next

# Set explicitly:
asusctl profile -P Quiet      # on battery, reading / meeting
asusctl profile -P Balanced   # plugged in, normal work
asusctl profile -P Performance # compiling, training

# Inspect:
asusctl profile -p
# Expect: current profile.
```

These also wire to `power-profiles-daemon` on 26.04 (as documented in [3.4](../03-speed-and-boot-optimizations/04-filesystem-thermals-power.md)). The Plasma battery applet shows three modes (Power Save / Balanced / Performance); selecting there also changes `asusctl profile`.

### Fan curves

If you want to tune past the stock 3 curves (e.g. start ramping at 55 °C instead of 65 °C):

```bash
# List available curves for the current profile:
asusctl fan-curve -m Balanced -g
# Shows "CPU" and "GPU" fan curve arrays (x=temp, y=duty%).

# Set a CPU fan curve (comma-separated temp:duty pairs, 4 pairs):
asusctl fan-curve -m Balanced -f CPU -D "30c:0%,50c:20%,65c:45%,75c:70%,85c:100%"

# Set a GPU fan curve:
asusctl fan-curve -m Balanced -f GPU -D "30c:0%,60c:30%,75c:55%,85c:80%,95c:100%"

# Enable the custom curve (vs stock):
asusctl fan-curve -m Balanced -e true
```

These persist across reboots. To revert: `asusctl fan-curve -m Balanced -e false`.

## 2.3.5 `supergfxctl` modes — understand before you switch

| Mode             | What it does                                             | When to use                                      | Logout/reboot required? |
| ---------------- | -------------------------------------------------------- | ------------------------------------------------ | ----------------------- |
| **Integrated**   | Disables the NVIDIA driver entirely; only iGPU is active | Travel, lecture, 1.5–2x battery life             | Logout                  |
| **Hybrid**       | iGPU drives the display; dGPU is idle, PRIME offload     | **Default for 90 % of day-to-day.** CUDA, Gazebo, RViz, occasional gaming | None                    |
| **Vfio**         | dGPU bound to vfio-pci for VM passthrough                | VFIO a Windows VM for games                      | Reboot                  |
| **AsusMuxDgpu**  | dGPU drives the display directly (MUX switch)            | Max gaming perf plugged in                       | Reboot                  |
| **Egpu**         | eGPU support (not used on TUF; no TB port)               | Not applicable                                   | N/A                     |

On a TUF A17 for this workload, **you live in Hybrid** and occasionally switch to Integrated for travel.

```bash
# Switch to Integrated for a flight:
supergfxctl -m Integrated
# Log out of Plasma, log back in. nvidia-smi now says "No devices were found" — expected.

# Switch back to Hybrid:
supergfxctl -m Hybrid
# Log out, log back in. nvidia-smi works again.

# What mode am I in?
supergfxctl -g
```

### Why not just always stay in Hybrid?

Hybrid with the dGPU fully in runtime-PM (deep sleep) costs ~0.5 W idle — negligible vs Integrated. BUT if something wakes the dGPU (a rogue app that probes `/dev/nvidia*`, or a Flatpak that pulled a GL context on startup), the dGPU stays at ~10 W. Integrated mode is the nuclear option that makes the "stays woke" problem impossible.

**Rule of thumb:** on battery for more than ~2 hours = Integrated. Otherwise Hybrid.

## 2.3.6 Plasma applet for power profile and GPU mode

Install the KDE applet that surfaces asusctl / supergfxctl state in the system tray:

```bash
sudo apt install -y plasma-applet-asusctl 2>/dev/null || \
  flatpak install -y flathub com.github.jurplel.plasma-asusctl 2>/dev/null || \
  true
```

If neither is available in your 26.04 archive snapshot on day 0, the functionality is fine without the applet — you can always `asusctl profile -P Quiet` from Konsole.

## 2.3.7 Keyboard backlight and RGB

```bash
# Current backlight level (0-3 on TUF, 0=off):
asusctl -k

# Set to level 1 (dim):
asusctl -k 1

# The TUF A17 has a 4-zone RGB keyboard. asusctl can set colors, though GUI via rog-control-center is easier.
```

## 2.3.8 USB and charging

```bash
# Enable USB charging while laptop is off (if supported by BIOS — usually is on TUF A17):
# This is a BIOS setting on the A17 generation rather than asusctl-configurable.

# Enable "slow charging" (25W when it detects laptop is plugged into a non-ASUS 120W charger that
# can't sustain peak load — prevents the battery-drain-while-plugged phenomenon):
# Also BIOS-side, not asusctl.
```

## 2.3.9 First post-install reboot and verify

```bash
systemctl reboot
```

After reboot:

```bash
# State check:
asusctl
# Should show Profile=Balanced, Charge limit=80, correct fan curves.

supergfxctl -g
# Should show Hybrid.

# Hot-key test (these should now respond):
# - Fn+F5 cycles profile
# - Fn+F7/F8 adjusts brightness
# - Fn+F10 toggles Wi-Fi
# - Fn+F9 toggles touchpad

# Battery cap persists:
cat /sys/class/power_supply/BAT*/charge_control_end_threshold
# Expect: 80
```

Take a snapshot:

```bash
sudo timeshift --create --comments "post-asus-tuf-tooling $(date -I)" --tags D
```

Proceed to [2.4 Wayland + Plasma 6.6 + NVIDIA tuning](04-wayland-plasma-66-nvidia.md).

---

[Home](../README.md) · [↑ 02 Post-Install](README.md) · [← Previous: 2.2 NVIDIA & CUDA](02-nvidia-driver-580-and-cuda.md) · **2.3 ASUS TUF hybrid-GPU** · [Next: 2.4 Wayland & Plasma →](04-wayland-plasma-66-nvidia.md)
