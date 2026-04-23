[Home](README.md) · [← Previous: 8.3 Firewall](08-productivity-security/03-firewall-sandboxing.md) · **09 — Apps catalog** · [Next: 10 Troubleshooting →](10-troubleshooting.md)

---

# 09 — Curated Applications Catalog

~100 applications, 10 categories, with install channel and one-line rationale. Install channel legend:

- **APT-U** — Ubuntu archive.
- **APT-M** — Mozilla / Brave / Microsoft / vendor APT repo.
- **APT-ASUS** — `asus-linux.org` PPA.
- **APT-GH** — GitHub CLI's APT repo.
- **FP** — Flatpak from Flathub.
- **DEB** — vendor `.deb` download.
- **Cont.** — Podman container (Quadlet or rootless run).
- **uv-tool** — `uv tool install`.
- **cargo** — Rust binary via `cargo install`.
- **Upstream** — curl-piped installer (Ollama, Starship, Rustup, uv, Bun, Deno).
- **Distrobox** — inside a Distrobox container (ROS 2).

---

## 1. Browsers (see [4.1](04-daily-driver-stack/01-browsers.md))

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| Firefox             | APT-M       | Primary daily. Mozilla's APT repo; never snap.                       |
| Brave               | APT-M       | Chromium alternative; built-in ad-block; VA-API works out-of-box.   |
| Zen Browser         | FP          | Productivity-focused Firefox fork: workspaces, tab-trees.            |
| Chromium            | FP          | Last-resort clean Chromium for rendering tests. Never snap.         |
| Tor Browser         | FP          | `com.github.micahflee.torbrowser-launcher`. Anonymous browsing.     |

## 2. Terminal, shell, CLI (see [5.1](05-web-development/01-shell-and-terminal.md))

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| Konsole             | APT-U       | KDE-native, best Wayland integration.                                |
| Yakuake             | APT-U       | Drop-down terminal on `Meta+\``.                                    |
| Zsh                 | APT-U       | Primary shell.                                                       |
| Starship            | Upstream    | Fast, modern prompt. One binary.                                     |
| zellij              | APT-U       | Modern terminal multiplexer; better defaults than tmux.             |
| tmux                | APT-U       | Classic multiplexer; universal on servers.                           |
| eza                 | APT-U       | Modern `ls` with git + icons.                                       |
| bat                 | APT-U       | Modern `cat` with syntax highlighting.                              |
| ripgrep             | APT-U       | Fast `grep` respecting `.gitignore`.                                |
| fd-find             | APT-U       | Modern `find`.                                                       |
| fzf                 | APT-U       | Fuzzy finder; `Ctrl+R` for history.                                  |
| zoxide              | APT-U       | `z foo` replaces `cd` by fuzzy history.                             |
| btop                | APT-U       | Top replacement with GPU/NIC graphs.                                 |
| nvtop               | APT-U       | Per-GPU process view.                                                |
| tealdeur (`tldr`)   | APT-U       | Cheat-sheet CLI.                                                     |
| duf                 | APT-U       | Readable `df`.                                                       |
| dust                | cargo       | Interactive `du`.                                                    |
| hyperfine           | cargo       | Benchmark CLI commands.                                              |
| tokei               | cargo       | Lines of code in a project.                                          |

## 3. Editors and IDEs (see [5.4](05-web-development/04-editors-and-ides.md))

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| VS Code             | APT-M       | Primary IDE. `packages.microsoft.com` APT.                          |
| Cursor              | DEB         | AI-first VS Code fork.                                               |
| Zed                 | Upstream    | Fast Rust-native editor with collab.                                |
| Helix               | APT-U       | Modal terminal editor; modern Vim alternative.                      |
| JetBrains Toolbox   | DEB         | WebStorm / PyCharm / RustRover manager.                              |
| Kate                | APT-U       | KDE-native light editor; bundled in Minimal.                        |
| Neovim              | APT-U       | For Vim purists.                                                     |
| Meld                | APT-U       | GUI diff/merge.                                                      |

## 4. Robotics / simulation (see [06 ROS 2](06-ros2-robotics/README.md))

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| ROS 2 Jazzy         | Distrobox   | Bridge phase: Ubuntu 24.04 base.                                    |
| ROS 2 Lyrical Luth  | APT-U       | Native on 26.04, from 2026-05-22.                                   |
| ROS 2 Humble        | Distrobox   | Legacy projects pre-2024.                                            |
| Gazebo Harmonic     | Distrobox   | Simulator paired with Jazzy.                                         |
| Gazebo Ionic        | APT-U       | With Lyrical, post-May 22.                                           |
| PlotJuggler         | FP          | ROS bag + timeseries visualiser.                                    |
| Foxglove Studio     | FP          | Modern ROS visualisation / debugging.                               |
| QGroundControl      | FP          | PX4 / ArduPilot ground station.                                     |
| Mission Planner     | Wine / FP   | ArduPilot legacy ground station.                                    |

## 5. AI / ML (see [07 AI/ML](07-ai-ml-workstation/README.md))

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| NVIDIA driver 580   | APT-U       | Proprietary driver.                                                 |
| `nvidia-cuda-toolkit` | APT-U     | CUDA 13.2 native in 26.04 archive.                                   |
| `cudnn9-cuda-13`    | APT-U       | cuDNN 9.21.                                                          |
| uv                  | Upstream    | Rust-based Python manager; replaces pip/venv/poetry/pyenv.          |
| JupyterLab          | uv pip      | Inside the AI venv.                                                 |
| marimo              | uv pip      | Reactive notebook alternative to Jupyter.                           |
| Ollama              | Upstream    | Local LLM inference.                                                 |
| Open WebUI          | Cont.       | ChatGPT-like UI on Ollama.                                          |
| Continue.dev        | VSCode ext. | Local LLM autocompletion in VS Code.                                |
| Unsloth             | uv pip      | 2x-faster LoRA fine-tuning.                                         |
| Hugging Face CLI    | uv tool     | `hf`. Model/dataset download and upload.                            |
| LM Studio           | FP          | Desktop LLM GUI alternative to Ollama+WebUI. `io.lmstudio.LMStudio`. |
| Perplexica          | Cont.       | Self-hosted Perplexity-like search (uses SearXNG + LLM).            |

## 6. Productivity (see [4.5](04-daily-driver-stack/05-comms-and-productivity.md), [4.6](04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md))

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| Obsidian            | FP          | Markdown notes with graph view.                                      |
| Joplin              | FP          | E2EE sync notes.                                                     |
| LibreOffice         | APT-U       | Office suite.                                                        |
| OnlyOffice          | FP          | MS-Office-compat alternative.                                        |
| KeePassXC           | APT-U       | Local password manager; TOTP codes.                                 |
| Bitwarden Desktop   | FP          | Cloud-synced password manager.                                      |
| Flameshot           | FP          | Annotated screenshots.                                               |
| Spectacle           | APT-U       | Plasma screenshots (built-in).                                      |
| AutoKey             | APT-U       | Text expansion.                                                      |
| KeyD                | APT-U       | System-wide key remap.                                              |
| Syncthing           | APT-U       | P2P file sync.                                                       |
| rclone              | APT-U       | Mount cloud storage as fs.                                          |
| Restic              | APT-U       | Encrypted deduplicated backup.                                      |
| Timeshift           | APT-U       | BTRFS snapshots / rollback.                                          |
| btrfs-assistant     | APT-U       | GUI for BTRFS + Snapper/Timeshift.                                  |
| chezmoi             | APT-U       | Dotfiles manager.                                                   |
| Calibre             | FP          | E-book library manager.                                              |

## 7. Communication (see [4.5](04-daily-driver-stack/05-comms-and-productivity.md))

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| Signal Desktop      | FP          | E2EE personal messaging.                                            |
| Telegram Desktop    | FP          | Groups, channels, fast.                                             |
| Discord             | FP          | Gaming / community voice.                                           |
| Element             | FP          | Matrix client. `im.riot.Riot`.                                      |
| Thunderbird         | APT-M       | Email + calendar + newsgroups.                                      |
| Zoom                | FP or DEB   | Corporate video calls.                                              |
| Jitsi Meet (browser)| —           | WebRTC video calls.                                                 |
| Slack               | FP          | `com.slack.Slack`. Work chat.                                       |
| Zulip               | FP          | `org.zulip.Zulip`. Alternative work chat.                           |
| Dino                | FP          | Modern XMPP client.                                                 |
| BlueMail / Geary    | FP          | Alternative mail clients.                                            |

## 8. Media (see [4.2](04-daily-driver-stack/02-video-audio-pipewire.md), [4.4](04-daily-driver-stack/04-music-streaming-local.md))

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| mpv                 | APT-U       | Primary video player.                                               |
| VLC                 | FP          | Secondary / DRM compat.                                             |
| Jellyfin Media Player | FP        | Client for self-hosted Jellyfin servers.                            |
| Celluloid           | FP          | GTK front-end for mpv (optional).                                   |
| Spotify             | FP          | Music streaming.                                                    |
| ncspot              | APT-U       | Terminal Spotify client.                                            |
| Strawberry          | APT-U       | Local music library manager.                                        |
| Elisa               | APT-U       | Plasma-native music player.                                         |
| Tauon Music Box     | FP          | Modern local music player.                                           |
| MPD + ncmpcpp       | APT-U       | Headless music daemon + TUI client.                                  |
| yt-dlp              | APT-U / uv-tool | Video/audio downloader.                                          |
| OBS Studio          | FP          | Screencast / recording with PipeWire.                               |
| Kdenlive            | FP          | KDE-native video editor.                                            |
| DaVinci Resolve     | DEB (free)  | Professional video editor.                                          |
| Audacity            | FP          | Audio editor.                                                       |
| Pavucontrol         | APT-U       | PipeWire/PulseAudio volume control.                                 |
| EasyEffects         | FP          | Per-app audio DSP.                                                  |
| qpwgraph            | APT-U       | PipeWire node graph editor.                                         |
| GIMP                | FP          | Image editor.                                                       |
| Inkscape            | FP          | Vector graphics.                                                    |
| Krita               | FP          | Digital painting.                                                   |

## 9. System utilities

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| asusctl             | APT-ASUS    | TUF laptop control (fan, battery, profile).                         |
| supergfxctl         | APT-ASUS    | Hybrid-GPU mode arbitration.                                        |
| `power-profiles-daemon` | APT-U   | Power profile manager.                                              |
| fwupd               | APT-U       | Firmware update via LVFS.                                           |
| lm-sensors          | APT-U       | Temperature monitoring.                                             |
| smartmontools       | APT-U       | SMART disk health.                                                  |
| partitionmanager    | APT-U       | KDE partition manager.                                              |
| gparted             | APT-U       | GTK partition manager (alternative).                                |
| KSystemLog          | APT-U       | Plasma log viewer.                                                  |
| Dolphin             | APT-U       | File manager (KDE-native).                                          |
| Ark                 | APT-U       | Archive manager.                                                    |
| KRecorder           | APT-U       | Audio recording (KDE).                                              |
| KCalc               | APT-U       | Calculator.                                                         |
| Bluetooth Manager (blueman) | APT-U | Bluetooth tray applet.                                             |
| Blueberry           | APT-U       | Alternative BT GUI.                                                 |
| UFW                 | APT-U       | Simple firewall.                                                    |
| Firejail            | APT-U       | App sandbox.                                                        |
| ClamAV              | APT-U       | On-demand malware scan.                                             |

## 10. Virtualization / containers / emulation

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| Podman              | APT-U       | Rootless container runtime.                                          |
| Distrobox           | APT-U       | Host-integrated Podman containers.                                   |
| podman-compose      | APT-U       | Docker-compose compatibility layer.                                  |
| virt-manager        | APT-U       | libvirt VM GUI.                                                      |
| QEMU                | APT-U       | Hardware emulation.                                                  |
| Waydroid            | APT-U       | Android container runtime (for Android-on-Linux).                    |
| Bottles             | FP          | Wine configuration manager for Windows apps.                        |
| Lutris              | FP          | Games launcher (Steam + Epic + GOG + Wine).                         |
| Steam               | APT-U (non-free) or FP | Games platform. Prefer Flatpak for Steam on Kubuntu.     |

## 11. Gaming (bonus — single entry in the plan)

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| Steam               | FP          | `com.valvesoftware.Steam`. Flatpak is the cleanest on Kubuntu.      |
| Heroic Games Launcher | FP        | Epic / GOG launcher.                                                |
| GameHub             | FP          | Unified game library.                                               |
| Minecraft           | FP          | `com.mojang.Minecraft`.                                              |
| MangoHud            | APT-U       | FPS/CPU/GPU overlay.                                                |

For serious gaming, switch to `supergfxctl -m AsusMuxDgpu` before launching (reboot required); revert to Hybrid afterwards.

## 12. Web dev specific tooling (see [5.3–5.5](05-web-development/README.md))

| App                 | Channel     | One-liner                                                           |
| ------------------- | ----------- | ------------------------------------------------------------------- |
| fnm                 | Upstream    | Node version manager.                                                |
| Corepack            | bundled     | pnpm/yarn/npm shims.                                                 |
| Bun                 | Upstream    | Fast runtime / bundler / test runner / PM.                          |
| Deno                | Upstream    | TS-first secure runtime.                                             |
| Vercel CLI          | npm global  | Next.js deploy.                                                     |
| Cloudflare Wrangler | npm global  | Cloudflare Pages / Workers deploy.                                  |
| GitHub CLI (gh)     | APT-GH      | PRs and issues from terminal.                                        |
| DBeaver             | FP          | SQL GUI (`io.dbeaver.DBeaverCommunity`).                            |
| pgcli               | uv-tool     | Postgres CLI with autocomplete.                                     |
| httpie              | uv-tool     | cURL replacement.                                                    |
| mkcert              | APT-U       | Local HTTPS cert generator.                                         |

---

## Install channel summary

- **APT (Ubuntu + vendor repos)** — ~50 apps: core CLI, KDE utilities, drivers, vendor-maintained browsers/editors.
- **Flatpak (Flathub)** — ~30 apps: Spotify, Discord, Signal, Zen, OBS, DBeaver, most "consumer" apps.
- **Podman containers** — 4-6: Postgres, Redis, MinIO, Open WebUI, Ollama (wrapper), Perplexica.
- **Distrobox** — 2: Jazzy, Humble.
- **Upstream installer scripts** — 5: Ollama, Starship, Rustup, uv, Bun, Deno.
- **`uv tool`** — 3-8: Python CLIs like yt-dlp, httpie, pgcli.
- **`cargo install`** — 5: Rust-native tools not in apt.

---

[Home](README.md) · [← Previous: 8.3 Firewall](08-productivity-security/03-firewall-sandboxing.md) · **09 — Apps catalog** · [Next: 10 Troubleshooting →](10-troubleshooting.md)
