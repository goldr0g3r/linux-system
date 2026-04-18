[Home](../../README.md) · [↑ Dev Environment](README.md) · [← Previous: 3.1 Containers](01-containers.md) · **3.2 Embedded** · [Next: 3.3 AI/ML →](03-ai-ml.md)

---

# 3.2 Embedded & Hardware Toolchain

KiCad 8 for PCB design, `arm-none-eabi-gcc` + OpenOCD for STM32 bare-metal, PlatformIO for higher-level workflows, ST-Link udev rules so you do not need `sudo` to flash a Nucleo-144, and VS Code from Microsoft's APT repo (not Snap).

Everything here goes on the **host** — these tools need `/dev/ttyACM*`, `/dev/ttyUSB*`, and udev events that are messy to forward into containers.

## KiCad 8 (primary PCB EDA)

```bash
sudo add-apt-repository --yes ppa:kicad/kicad-8.0-releases
sudo apt update
sudo apt install -y \
  kicad kicad-libraries kicad-footprints kicad-symbols kicad-templates kicad-packages3d \
  kicad-demos

# Optional: pull JLCPCB / LCSC libraries via kicad-cli plugin manager from inside KiCad.
```

If you want rolling KiCad (9.x nightlies) in parallel without breaking the stable 8.x install, use Flatpak:

```bash
flatpak install -y flathub org.kicad.KiCad
```

The Flatpak and APT versions coexist — the Flatpak gets its own config directory at `~/.var/app/org.kicad.KiCad/config/kicad/`, so your symbol/footprint libraries do not bleed across.

**Library sources you will want after install:**

- **Official KiCad libraries** — installed above.
- **JLCPCB/LCSC component library** — install the `kicad-jlcpcb-tools` plugin via KiCad's built-in plugin manager (Tools → Plugin Manager).
- **SnapEDA / Ultra Librarian** — on-demand component exports; download `.kicad_sym` / `.pretty` files into a `lib/` folder in each project.
- **KiCanvas** — browser-based KiCad file viewer; useful for sharing screenshots with hardware vendors.

## STM32 bare-metal toolchain

```bash
sudo apt install -y \
  gcc-arm-none-eabi gdb-multiarch binutils-arm-none-eabi \
  libnewlib-arm-none-eabi libnewlib-nano-arm-none-eabi \
  openocd stlink-tools \
  build-essential cmake ninja-build make \
  minicom picocom screen gtkterm cutecom \
  srecord
```

Verify:

```bash
arm-none-eabi-gcc --version    # expect 13.x for Ubuntu 24.04
openocd --version              # expect 0.12.x
st-info --probe                # expect: device probe (if Nucleo plugged in) or "no devices found"
```

## STM32CubeIDE (proprietary ST IDE)

Download from [st.com](https://www.st.com/en/development-tools/stm32cubeide.html) (requires free ST account). Pick the Linux generic installer (`.sh`).

```bash
# Assuming you downloaded to ~/Downloads/en.st-stm32cubeide_1.15.1_...lin.sh.zip
mkdir -p ~/st && cd ~/st
unzip ~/Downloads/en.st-stm32cubeide_*.sh.zip
chmod +x st-stm32cubeide_*.sh
sudo ./st-stm32cubeide_*.sh
# Follow the installer; accept license; install to /opt/st/stm32cubeide_<version>
```

Create a KDE .desktop entry so it appears in KRunner and the application menu:

```bash
mkdir -p ~/.local/share/applications
tee ~/.local/share/applications/stm32cubeide.desktop >/dev/null <<EOF
[Desktop Entry]
Type=Application
Name=STM32CubeIDE
Comment=Integrated development environment for STM32
Exec=/opt/st/stm32cubeide_1.15.1/stm32cubeide
Icon=/opt/st/stm32cubeide_1.15.1/icon.xpm
Terminal=false
Categories=Development;IDE;Embedded;
EOF
# Adjust the version in the path if different.
```

## ST-Link V2 udev rules

So you do not need `sudo` to flash the Nucleo-144:

```bash
sudo tee /etc/udev/rules.d/49-stlinkv2.rules >/dev/null <<'EOF'
# ST-Link V2
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="3748", \
  MODE="0660", GROUP="plugdev", TAG+="uaccess", SYMLINK+="stlinkv2_%n"

# ST-Link V2-1 (on Nucleo-144 boards)
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374b", \
  MODE="0660", GROUP="plugdev", TAG+="uaccess", SYMLINK+="stlinkv2-1_%n"

# ST-Link V3
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374e", \
  MODE="0660", GROUP="plugdev", TAG+="uaccess", SYMLINK+="stlinkv3_%n"
SUBSYSTEMS=="usb", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="374f", \
  MODE="0660", GROUP="plugdev", TAG+="uaccess", SYMLINK+="stlinkv3_%n"
EOF

sudo usermod -aG plugdev,dialout $USER
sudo udevadm control --reload-rules && sudo udevadm trigger
# Log out + log in (or reboot) for group membership to take effect.
```

Verify after re-login:

```bash
groups | grep -E 'plugdev|dialout'   # both should appear

# Plug in Nucleo, then:
lsusb | grep ST-LINK
st-info --probe
# Expect: the Nucleo description + its serial
```

## PlatformIO (inside VS Code; covers STM32, ESP32, Arduino, Teensy)

### Install VS Code natively via the Microsoft APT repo (not Snap)

```bash
sudo apt install -y wget gpg
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | \
  gpg --dearmor | sudo tee /usr/share/keyrings/vscode.gpg >/dev/null
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/vscode.gpg] \
  https://packages.microsoft.com/repos/vscode stable main" | \
  sudo tee /etc/apt/sources.list.d/vscode.list
sudo apt update && sudo apt install -y code
```

### Install PlatformIO CLI on the host (VS Code extension will auto-detect)

```bash
curl -fsSL https://raw.githubusercontent.com/platformio/platformio-core-installer/master/get-platformio.py -o /tmp/get-platformio.py
python3 /tmp/get-platformio.py
echo 'export PATH="$HOME/.platformio/penv/bin:$PATH"' >> ~/.bashrc

# udev rules for PlatformIO's wider board set:
curl -fsSL https://raw.githubusercontent.com/platformio/platformio-core/master/platformio/assets/system/99-platformio-udev.rules | \
  sudo tee /etc/udev/rules.d/99-platformio-udev.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Then in VS Code: **Extensions → search "PlatformIO IDE" → install.** First launch downloads the platform cores lazily; pick the STM32 and ESP32 platforms from the PIO Home tab.

## Signal analysis / logic analyzer

You will need this for the high-frequency driver circuits you mentioned:

```bash
sudo apt install -y sigrok pulseview sigrok-cli sigrok-firmware-fx2lafw
# PulseView drives any fx2-based 8-ch 24 MHz analyser (~$10 on AliExpress) as a first logic probe.
# For higher-speed work, Saleae Logic software is available as a Flatpak (see Part 4).
```

Common adapters supported out of the box:

- **Generic 8-channel 24 MHz (fx2lafw)** — the $10 blue/yellow AliExpress box. Fine for I2C, SPI, UART debug.
- **Saleae Logic 8 / Pro** — use the vendor software instead (see [Part 4](../04-applications.md)).
- **DreamSourceLab DSLogic** — use their fork of PulseView (different package).
- **Rigol DS1000Z / DS1074Z scopes** — `sigrok-cli --driver rigol-ds --scan` for LAN/USB control.

## CAN bus (automotive → subsea telemetry bridge)

From [Part 3.5](05-learning-roadmap.md) this is one of your most transferable skills. Install host-side:

```bash
sudo apt install -y can-utils

# Virtual CAN for practice without hardware:
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set vcan0 up
# Persist across reboot:
echo vcan | sudo tee /etc/modules-load.d/vcan.conf

# Quick smoke test:
candump vcan0 &
cangen vcan0 -I 123 -L 8 -g 100
# Expect: vcan0 123 [8] ... rows scrolling
```

For real hardware (PCAN-USB, PEAK, innomaker 2xCANFD, Kvaser Leaf, etc.), SocketCAN handles most USB-CAN adapters via kernel modules; check `dmesg` after plugging in and map the interface (`can0`) with `sudo ip link set can0 up type can bitrate 500000`.

## VS Code extensions you want installed

Once VS Code is up (APT version, not Snap), install these from the Extensions panel:

**Core:**
- `ms-vscode.cpptools` — C/C++
- `ms-python.python` + `ms-python.vscode-pylance` — Python
- `rust-lang.rust-analyzer` — Rust
- `platformio.platformio-ide` — PlatformIO (above)
- `marus25.cortex-debug` — STM32 debugging via OpenOCD

**Robotics:**
- `ms-iot.vscode-ros` — ROS 2 launch files, message introspection
- `deepblu.vscode-stm32-for-vscode` — STM32CubeMX project integration
- `zephyr-project.vscode-zephyr-ide` — Zephyr RTOS (if you go that route)

**Quality of life:**
- `ms-vscode-remote.remote-containers` — attach to Distrobox containers
- `github.copilot` + `github.copilot-chat` — if you use Copilot (free for students with GitHub Education)
- `continue.continue` — alternative: local-LLM assistant backed by Ollama (see [3.3](03-ai-ml.md))
- `eamodio.gitlens` — git blame + history inline
- `usernamehw.errorlens` — inline error messages
- `gruntfuggly.todo-tree` — TODO comment aggregator

## Verification

```bash
# Host toolchains:
arm-none-eabi-gcc --version
openocd --version
kicad-cli version
code --version

# Hardware access (plug in Nucleo-144 first):
st-info --probe
platformio device list --serial

# CAN:
ip link show vcan0
```

If all five succeed, the embedded stack is ready. Proceed to [3.3 AI/ML](03-ai-ml.md).

---

[Home](../../README.md) · [↑ Dev Environment](README.md) · [← Previous: 3.1 Containers](01-containers.md) · **3.2 Embedded** · [Next: 3.3 AI/ML →](03-ai-ml.md)
