[Home](../README.md) · [← Previous: 3.5 Learning Roadmap](03-dev-environment/05-learning-roadmap.md) · **Part 4: Applications** · [Next: Appendix A →](appendix-a-provisioning.md)

---

# Part 4 — Curated Application Stack

Every entry specifies **install channel**:

- **APT** — native `.deb` from Ubuntu/Kubuntu repos.
- **FP** — Flatpak from Flathub.
- **Repo** — upstream APT repository (add a third-party signing key and source).
- **BIN** — upstream-provided binary or AppImage.
- **Container** — Podman/Distrobox container (typically as a service).
- **PIP** / **uv** — Python package.
- **Cargo** — Rust binary via `cargo install`.
- **Go** — Go binary via `go install`.

**Rule of thumb:** prefer **APT** for toolchains that touch `/dev`, `/sys`, or other host interfaces (they need host kernel integration, not sandboxing). Prefer **Flatpak** for GUI consumer apps where confinement is acceptable. Use **BIN** / **Cargo** when the APT version is too old or missing.

## Jump to a category

- [4.1 Browsers & web](#41-browsers--web)
- [4.2 Terminal, shell, core CLI upgrades](#42-terminal-shell-core-cli-upgrades)
- [4.3 IDEs, editors, and dev tools](#43-ides-editors-and-dev-tools)
- [4.4 Robotics, embedded, EDA](#44-robotics-embedded-eda)
- [4.5 AI / ML / data](#45-ai--ml--data)
- [4.6 Productivity, notes, reading](#46-productivity-notes-reading)
- [4.7 Communication](#47-communication)
- [4.8 Media, creation, consumption](#48-media-creation-consumption)
- [4.9 System, backup, privacy, utilities](#49-system-backup-privacy-utilities)
- [4.10 Virtualization & containers](#410-virtualization--containers)

---

## 4.1 Browsers & web

| App | Channel | Install | Use case |
|-----|---------|---------|----------|
| Firefox | Repo | Kubuntu default (Mozilla APT repo) | Primary browser |
| Firefox (Flatpak, isolated profile) | FP | `flatpak install flathub org.mozilla.firefox` | Parallel isolated Firefox profile for web dev, kept separate from the APT Firefox |
| Chromium | FP | `flatpak install flathub org.chromium.Chromium` | Cross-browser testing for Next.js blog |
| Brave | Repo | `curl -fsSLo /usr/share/keyrings/brave-browser-archive-keyring.gpg https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg` | Privacy-first browsing, good mobile sync |
| LibreWolf | FP | `flatpak install flathub io.gitlab.librewolf-community` | Hardened Firefox fork for research |

---

## 4.2 Terminal, shell, core CLI upgrades

| App | Channel | Install | Replaces / adds |
|-----|---------|---------|-----------------|
| Konsole | APT | preinstalled | Keep as default — Plasma-native, fastest |
| Alacritty | APT | `sudo apt install alacritty` | GPU-accelerated alternative |
| WezTerm | Repo | `curl -L https://github.com/wez/wezterm/releases/download/...` | Lua-config, multiplexing, best for multi-SSH workflows |
| Starship | BIN | `curl -sS https://starship.rs/install.sh \| sh` | Unified shell prompt |
| fish | APT | `sudo apt install fish` | Alternative to bash/zsh with better defaults |
| zsh + zinit | APT | `sudo apt install zsh`; zinit-specific init | If you want heavy customization |
| tmux | APT | `sudo apt install tmux` | Terminal multiplexing (non-negotiable for ROS) |
| zellij | BIN | `cargo install zellij` or `.tar.gz` | Modern tmux alternative, floating panes |
| ripgrep (`rg`) | APT | `sudo apt install ripgrep` | Replace `grep` for code search |
| fd | APT | `sudo apt install fd-find` | Replace `find` |
| fzf | APT | `sudo apt install fzf` | Fuzzy finder, binds to Ctrl+R/T |
| bat | APT | `sudo apt install bat` | `cat` with syntax highlighting |
| eza | Cargo | `cargo install eza` | Replace `ls` |
| zoxide | APT | `sudo apt install zoxide` | Smart `cd` |
| btop | APT | `sudo apt install btop` | Replace `top`/`htop` |
| nvtop | APT | `sudo apt install nvtop` | GPU monitoring (NVIDIA + AMD) |
| duf | APT | `sudo apt install duf` | Replace `df` |
| dust | Cargo | `cargo install du-dust` | Replace `du` |
| procs | Cargo | `cargo install procs` | Replace `ps` |
| gping | Cargo | `cargo install gping` | `ping` with graphs |
| delta | APT | `sudo apt install git-delta` | Git diff viewer |
| lazygit | Go | `go install github.com/jesseduffield/lazygit@latest` | TUI git client |
| yt-dlp | APT | `sudo apt install yt-dlp` | Download lectures/talks |

---

## 4.3 IDEs, editors, and dev tools

| App | Channel | Install | Use case |
|-----|---------|---------|----------|
| VS Code | Repo | Microsoft APT repo (see [3.2](03-dev-environment/02-embedded.md)) | Primary IDE, PlatformIO, Next.js |
| VSCodium | FP | `flatpak install flathub com.vscodium.codium` | Telemetry-free VS Code fork (alt) |
| Neovim | APT | `sudo apt install neovim` | Lightweight edits, SSH sessions |
| LazyVim / NvChad | BIN | `git clone ... ~/.config/nvim` | Pre-configured Neovim distro |
| Helix | APT | `sudo apt install helix` | Modal editor without Vim baggage |
| Zed | Repo | `curl -f https://zed.dev/install.sh \| sh` | Fast, collaborative editor |
| JetBrains Toolbox | BIN | download from JetBrains | Manage CLion (C++), PyCharm Pro (AI), WebStorm (Next.js) — free as a student |
| Meld | APT | `sudo apt install meld` | GUI diff/merge |
| Kate | APT | preinstalled | KDE text editor, surprisingly capable |
| DevDocs (offline) | FP | `flatpak install flathub io.github.martinrotter.devdocsdesktop` | Offline API docs for flights/travel |
| Insomnia | FP | `flatpak install flathub rest.insomnia.Insomnia` | REST/GraphQL API testing |
| Bruno | FP | `flatpak install flathub com.usebruno.Bruno` | Git-friendly Postman alternative |
| DBeaver | FP | `flatpak install flathub io.dbeaver.DBeaverCommunity` | SQL client for blog DB / RAG DB |
| Beekeeper Studio | FP | `flatpak install flathub io.beekeeperstudio.Studio` | Alternative SQL client |

---

## 4.4 Robotics, embedded, EDA

| App | Channel | Install | Use case |
|-----|---------|---------|----------|
| KiCad 9 | Repo | `ppa:kicad/kicad-9.0-releases` (see [3.2](03-dev-environment/02-embedded.md)) | PCB layout (primary) — current stable 9.0.8 |
| KiCad (Flatpak) | FP | `flatpak install flathub org.kicad.KiCad` | Parallel install for testing pre-release builds |
| FreeCAD (stable) | APT | `sudo apt install freecad` | Mechanical CAD for watertight enclosures |
| FreeCAD (RealThunder weekly) | BIN | AppImage from GitHub | More capable assembly workbench |
| OpenSCAD | APT | `sudo apt install openscad` | Parametric 3D for brackets / mounts |
| MeshLab | APT | `sudo apt install meshlab` | STL/PLY clean-up |
| PrusaSlicer | FP | `flatpak install flathub com.prusa3d.PrusaSlicer` | 3D print prep |
| Cura | FP | `flatpak install flathub com.ultimaker.cura` | Alternative slicer |
| LibreCAD | APT | `sudo apt install librecad` | 2D drafting |
| QCAD | FP | `flatpak install flathub org.qcad.QCAD` | Alternative 2D CAD |
| Fritzing | FP | `flatpak install flathub org.fritzing.Fritzing` | Breadboard schematics (learning) |
| PulseView / sigrok | APT | see [3.2](03-dev-environment/02-embedded.md) | Logic analyser |
| Saleae Logic 2 | BIN | download `Logic-2.*.AppImage` | For Saleae hardware |
| GTKTerm / CuteCom / moserial | APT | `sudo apt install gtkterm cutecom moserial` | Serial monitors |
| Arduino IDE 2 | FP | `flatpak install flathub cc.arduino.IDE2` | Arduino / ESP |
| PlatformIO Core | PIP | see [3.2](03-dev-environment/02-embedded.md) | MCU meta-framework |
| STM32CubeIDE | BIN | st.com | Proprietary ST IDE |
| Wireshark | APT | `sudo apt install wireshark` | Network capture (CAN, DDS, tether) |
| Savvy CAN | BIN | GitHub release | CAN bus GUI analyser (automotive) |
| can-utils | APT | see [3.5](03-dev-environment/05-learning-roadmap.md) | CLI CAN tools |
| Gazebo Harmonic | APT (Jazzy container) | `sudo apt install gz-harmonic` in `ros2-jazzy` | Robot simulator |
| RViz2 | APT (Jazzy container) | `ros-jazzy-desktop` | ROS visualiser |
| CoppeliaSim Edu | BIN | coppeliarobotics.com | Alternative simulator, free for education |
| QGroundControl | FP | `flatpak install flathub org.mavlink.qgroundcontrol` | MAVLink ground station |
| Mission Planner | Wine / VM | last-resort | APM/Pixhawk configuration (Windows-only; use your Win11 partition for this) |
| QGIS | APT | `sudo apt install qgis` | Geospatial analysis for marine ops |
| OpenCPN | APT | `sudo apt install opencpn` | Marine chart plotter |

---

## 4.5 AI / ML / data

| App | Channel | Install | Use case |
|-----|---------|---------|----------|
| JupyterLab | uv | see [3.3](03-dev-environment/03-ai-ml.md) | Notebooks, M.Tech coursework |
| marimo | uv | `uv pip install marimo` | Reactive, git-friendly notebooks |
| Ollama | BIN | see [3.3](03-dev-environment/03-ai-ml.md) | Local LLM server |
| Open WebUI | Container | see [3.3](03-dev-environment/03-ai-ml.md) | ChatGPT-style UI on top of Ollama |
| LM Studio | BIN | AppImage from lmstudio.ai | GUI LLM explorer with model browser |
| Stable Diffusion (ComfyUI) | Git | `git clone https://github.com/comfyanonymous/ComfyUI` + `uv pip install -r requirements.txt` | Generative image work |
| Netron | FP | `flatpak install flathub com.github.lutzroeder.Netron` | Neural net architecture visualiser |
| Label Studio | Container | `podman run -p 8080:8080 heartexlabs/label-studio` | Dataset annotation |
| Roboflow Inference | Container | `docker.io/roboflow/roboflow-inference-server-cpu` | CV pipelines |

---

## 4.6 Productivity, notes, reading

| App | Channel | Install | Use case |
|-----|---------|---------|----------|
| Obsidian | FP | `flatpak install flathub md.obsidian.Obsidian` | Second brain, M.Tech notes, research |
| Logseq | FP | `flatpak install flathub com.logseq.Logseq` | Alternative to Obsidian (outliner) |
| Joplin | FP | `flatpak install flathub net.cozic.joplin_desktop` | Markdown notes with E2E sync |
| Zotero | FP | `flatpak install flathub org.zotero.Zotero` | Research paper management (essential for M.Tech) |
| Anki | FP | `flatpak install flathub net.ankiweb.Anki` | Spaced-repetition flashcards |
| Foliate | FP | `flatpak install flathub com.github.johnfactotum.Foliate` | EPUB reader |
| Calibre | APT | `sudo apt install calibre` | E-book library + conversion |
| Okular | APT | preinstalled | PDF / DjVu, KDE-native |
| Xournal++ | APT | `sudo apt install xournalpp` | PDF annotation, handwriting |
| LibreOffice | APT | preinstalled | Office docs |
| OnlyOffice | FP | `flatpak install flathub org.onlyoffice.desktopeditors` | Better MS Office compatibility |
| Marktext / Ghostwriter / Apostrophe | FP | Flathub | Distraction-free Markdown for the blog |
| Standard Notes | FP | `flatpak install flathub org.standardnotes.standardnotes` | Encrypted cloud notes (alt) |

---

## 4.7 Communication

| App | Channel | Install | Use case |
|-----|---------|---------|----------|
| Thunderbird | FP | `flatpak install flathub org.mozilla.Thunderbird` | Email (Kubuntu ships no default client) |
| Signal | FP | `flatpak install flathub org.signal.Signal` | Secure messaging |
| Telegram | APT | `sudo apt install telegram-desktop` | Messaging |
| Element | FP | `flatpak install flathub im.riot.Riot` | Matrix client (open source) |
| Slack | FP | `flatpak install flathub com.slack.Slack` | Work |
| Discord | FP | `flatpak install flathub com.discordapp.Discord` | Communities |
| Zoom | FP | `flatpak install flathub us.zoom.Zoom` | Meetings |
| Teams-for-Linux | FP | `flatpak install flathub com.github.IsmaelMartinez.teams_for_linux` | MS Teams (unofficial) |
| Jitsi Meet Electron | FP | `flatpak install flathub org.jitsi.jitsi-meet` | Open-source video calls |

---

## 4.8 Media, creation, consumption

| App | Channel | Install | Use case |
|-----|---------|---------|----------|
| VLC | APT | `sudo apt install vlc` | Media playback |
| mpv | APT | `sudo apt install mpv` | Minimalist player, scriptable |
| OBS Studio | FP | `flatpak install flathub com.obsproject.Studio` | Screen recording, project demos |
| Kdenlive | APT | preinstalled | Video editing (KDE-native) |
| DaVinci Resolve | BIN | blackmagicdesign.com | Pro video (free tier) |
| Krita | APT | `sudo apt install krita` | Digital painting, diagrams |
| GIMP | FP | `flatpak install flathub org.gimp.GIMP` | Raster editing |
| Inkscape | APT | `sudo apt install inkscape` | Vector graphics, schematics |
| Blender | FP | `flatpak install flathub org.blender.Blender` | 3D — useful for ROV CAD + simulation assets |
| darktable | APT | `sudo apt install darktable` | RAW photo processing |
| Audacity | FP | `flatpak install flathub org.audacityteam.Audacity` | Audio (hydrophone capture) |
| HandBrake | FP | `flatpak install flathub fr.handbrake.ghb` | Video transcoding |
| Spotify | FP | `flatpak install flathub com.spotify.Client` | Music |
| Tidal-hifi | FP | `flatpak install flathub com.mastermindzh.tidal-hifi` | Tidal |
| Flameshot | APT | `sudo apt install flameshot` | Screenshots with annotations |
| Spectacle | APT | preinstalled | KDE-native screenshot tool (good enough) |
| Peek / Blue Recorder | FP | Flathub | GIF / short screen recordings |

---

## 4.9 System, backup, privacy, utilities

| App | Channel | Install | Use case |
|-----|---------|---------|----------|
| Timeshift | APT | see [2.5](02-setup/05-kernel-boot.md) | BTRFS snapshot manager |
| Snapper + snapper-gui | APT | `sudo apt install snapper` | BTRFS snapshots with pre/post-apt hooks (alt to Timeshift) |
| Vorta | FP | `flatpak install flathub com.borgbase.Vorta` | BorgBackup GUI (remote encrypted backups) |
| BleachBit | APT | `sudo apt install bleachbit` | Cache/log cleaner |
| Stacer | FP | `flatpak install flathub com.oguzhaninan.Stacer` | System monitor + cleaner |
| Filelight | APT | preinstalled | KDE disk-usage visualiser |
| KDiskMark | FP | `flatpak install flathub io.github.jonmagon.kdiskmark` | SSD benchmarks |
| Hardware Probe | APT | `sudo apt install hw-probe` | Submit hardware reports to linux-hardware.org |
| Bottles | FP | `flatpak install flathub com.usebottles.bottles` | Wine prefix manager for Windows-only tools |
| Lutris | FP | `flatpak install flathub net.lutris.Lutris` | Games (when you need a break) |
| Steam | FP | `flatpak install flathub com.valvesoftware.Steam` | Games |
| KeePassXC | FP | `flatpak install flathub org.keepassxc.KeePassXC` | Offline password manager |
| Bitwarden | FP | `flatpak install flathub com.bitwarden.desktop` | Cloud password manager |
| Proton VPN / Mullvad | BIN | respective repos | VPN when on hotel / airport Wi-Fi |
| WireGuard | APT | `sudo apt install wireguard-tools` | Self-host VPN to home lab |
| syncthing | APT | `sudo apt install syncthing` | P2P sync of blog drafts, configs |
| Nextcloud Desktop | APT | `sudo apt install nextcloud-desktop` | Self-hosted cloud sync |
| rclone | APT | `sudo apt install rclone` | CLI sync to 40+ cloud providers |
| KDE Connect | APT | preinstalled | Phone integration (SMS, clipboard, file sharing) |
| Barrier / Input Leap | FP | `flatpak install flathub io.github.input_leap.input-leap` | Software KVM between laptops |
| Ferdium | FP | `flatpak install flathub org.ferdium.Ferdium` | Multi-service messenger hub |
| Rustdesk | BIN | GitHub release | Self-hostable remote desktop |

---

## 4.10 Virtualization & containers

| App | Channel | Install | Use case |
|-----|---------|---------|----------|
| Podman | APT | see [3.1](03-dev-environment/01-containers.md) | Primary container engine (rootless, daemonless) |
| Distrobox | APT | see [3.1](03-dev-environment/01-containers.md) | Host-integrated containers for dev envs |
| Docker Engine | Repo | docker.com APT repo | Only if a vendor tool hard-requires Docker |
| QEMU / KVM / libvirt | APT | `sudo apt install qemu-kvm libvirt-daemon-system virt-manager` | Full VMs (faster than VirtualBox, native, free) |
| virt-manager | APT | (above) | GUI for KVM |
| VirtualBox | Repo | Oracle APT repo | Keep if you have existing VMs; prefer KVM for new work |
| GNOME Boxes | FP | `flatpak install flathub org.gnome.Boxes` | Simpler VM GUI |
| Vagrant | APT | `sudo apt install vagrant` | Reproducible VM provisioning |
| k3s | BIN | `curl -sfL https://get.k3s.io \| sh` | Lightweight Kubernetes when you want to learn k8s for marine-robotics cloud backends |

---

## Minimum recommended starter set (the "first 20")

If you want to install the **smallest useful subset** and build from there, these are the 20 applications that deliver the highest value and are cited most often in the rest of this guide:

1. `code` — VS Code ([Part 3.2](03-dev-environment/02-embedded.md))
2. `podman` + `distrobox` ([Part 3.1](03-dev-environment/01-containers.md))
3. `kicad` ([Part 3.2](03-dev-environment/02-embedded.md))
4. `timeshift` ([Part 2.5](02-setup/05-kernel-boot.md))
5. `kwin-bismuth` ([Part 2.7](02-setup/07-ui-ux.md))
6. `ripgrep`, `fd-find`, `fzf`, `bat`, `btop` ([4.2](#42-terminal-shell-core-cli-upgrades))
7. `git-delta`
8. `flatpak` + Flathub ([Part 2.4](02-setup/04-debloat.md))
9. Firefox (APT from Mozilla)
10. Thunderbird (Flatpak)
11. Obsidian (Flatpak)
12. Zotero (Flatpak)
13. KeePassXC (Flatpak)
14. Signal (Flatpak)
15. OBS Studio (Flatpak)
16. QGroundControl (Flatpak)
17. `can-utils`, `wireshark`
18. `nvtop`, `btop`
19. `uv` ([Part 3.3](03-dev-environment/03-ai-ml.md))
20. `fnm` ([Part 3.4](03-dev-environment/04-web.md))

Everything else in this guide installs on-demand as your workflow evolves.

Proceed to [Appendix A — Provisioning Script](appendix-a-provisioning.md) for the one-shot install.

---

[Home](../README.md) · [← Previous: 3.5 Learning Roadmap](03-dev-environment/05-learning-roadmap.md) · **Part 4: Applications** · [Next: Appendix A →](appendix-a-provisioning.md)
