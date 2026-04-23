[Home](../README.md) · [← Previous: 3.4 Filesystem & thermals](../03-speed-and-boot-optimizations/04-filesystem-thermals-power.md) · **04 — Daily-Driver Stack** · [Next: 4.1 Browsers →](01-browsers.md)

---

# 04 — Daily-Driver Stack

The applications and configuration for your stated daily workload: **browsing, video, Bluetooth headset, music, communication**, on top of a Plasma 6.6 desktop tuned for one-person productive use.

## Steps

| #   | Step                                                                           | Time    | Core daily driver? |
| ---:| ------------------------------------------------------------------------------ | ------- | ------------------ |
| 4.1 | [Browsers (Firefox via Mozilla APT, Brave, Zen, Vivaldi — no snap)](01-browsers.md) | 10 min  | Yes                |
| 4.2 | [Video + audio + PipeWire architecture + VA-API hardware decode](02-video-audio-pipewire.md) | 15 min  | Yes                |
| 4.3 | [Bluetooth headset (LDAC / AptX HD codec config, HSP/HFP mic switching, auto-reconnect)](03-bluetooth-headset-hifi.md) | 20 min  | Yes                |
| 4.4 | [Music: Spotify (Flatpak), local libraries, terminal players, podcast ripping](04-music-streaming-local.md) | 10 min  | Yes                |
| 4.5 | [Communication + productivity apps (Telegram, Signal, Discord, Obsidian)](05-comms-and-productivity.md) | 10 min  | Yes                |
| 4.6 | [Plasma 6.6 UX: per-desktop tiling, Activities, Konsole / Yakuake, fonts](06-plasma-66-ux-tiling-activities.md) | 25 min  | Yes                |

Total: ~90 minutes for the complete stack.

## Ordering rationale

- **Browsers first** because the post-install debloat ([2.1](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md)) already put Firefox in via Mozilla APT. Nothing in this section depends on a browser, but confirming your default-daily-browser is live is the fastest "the system is mine" signal.
- **Audio architecture before the Bluetooth headset.** LDAC codec priority only matters if PipeWire + WirePlumber are understood first. Page 4.2 is the theory; 4.3 is the headset-specific recipe that builds on it.
- **Communication and music after audio works.** No point pairing Slack or Discord if the output device is silent.
- **Plasma UX last.** The tiling rules, Activities, and Konsole profiles are most useful once you have actual workflows to tile (browser + editor + terminal + chat).

## Install channel philosophy

This section is opinionated about where each app comes from. Short version:

| Channel                 | Use for                                                                        | Avoid for                                        |
| ----------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------ |
| **APT (upstream repo)** | Firefox, Brave, VS Code, Cursor, `mpv`, `git`, `zsh`, anything with a trustworthy vendor-signed APT repo | Apps with a vendor Flatpak and no vendor APT     |
| **APT (Ubuntu archive)**| Kernel, drivers, `mpv`, most CLI tools, system services                        | Slow-moving Ubuntu versions of apps that matter (Firefox, LibreOffice) |
| **Flatpak (Flathub)**   | Spotify, Discord, Signal, Zen Browser, OBS Studio, Telegram Desktop            | Shell tools, CLI utilities, anything headless    |
| **.deb download (vendor)** | Cursor, Zoom, VSCode-Insiders when no repo exists                             | Anything that auto-updates via its own mechanism that might conflict with apt |
| **Container (Podman)**  | Postgres, Redis, MinIO, Ollama+WebUI, ROS 2 (Distrobox wraps this)             | Daily GUI apps                                   |
| **Snap**                | **Never, on this system.**                                                      | Literally everything — snap is removed in [2.1](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md) |

The rationale for each preference is on the individual pages.

---

[Home](../README.md) · [← Previous: 3.4 Filesystem & thermals](../03-speed-and-boot-optimizations/04-filesystem-thermals-power.md) · **04 — Daily-Driver Stack** · [Next: 4.1 Browsers →](01-browsers.md)
