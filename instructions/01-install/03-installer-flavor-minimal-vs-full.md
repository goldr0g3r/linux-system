[Home](../README.md) · [↑ 01 Install](README.md) · [← Previous: 1.2 Windows prep & BIOS](02-windows-prep-and-bios.md) · **1.3 Installer flavor** · [Next: 1.4 Partitioning →](04-installation-and-partitioning.md)

---

# 1.3 Installer Flavor — Minimal vs Normal vs Extended

The Calamares installer in Kubuntu 26.04 asks you, on the **Software selection** screen, to pick one of three package sets plus a couple of checkboxes. This is one of the most consequential five-minute decisions in the whole install, because it determines your on-disk footprint, first-boot service count, and debloat workload for the next 90 minutes.

## The three flavors, explicitly

Kubuntu 26.04's Calamares presents three radio buttons:

| Name                  | Rough installed-size | What it is                                                                                                                                          |
| --------------------- | -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Minimal**           | ~3.5 GB              | KDE Plasma Desktop, SDDM greeter, Firefox via Mozilla APT, Discover, Dolphin, Konsole, Spectacle, Kate, a text editor. No office, no games, no PIM. |
| **Normal** (default)  | ~7 GB                | Minimal + LibreOffice suite, Elisa music player, KMail+Korganizer+Kontact suite, Kdenlive video editor hooks, sample wallpapers, a couple of games. |
| **Extended** / **Full** | ~9 GB              | Normal + all KDE-Edu packages, Krita, Kontact, extra language packs, and most of the "looks good on a spec sheet" long tail.                        |

Plus two checkboxes that apply to any of the three:

- ☑ **Install third-party software for graphics and Wi-Fi hardware, and additional media formats** — installs patent-encumbered codecs (H.264, HEVC, MP3), NVIDIA's `ubuntu-drivers` metapackage, and `mesa-extra`/VA-API bits you will need in [4.2](../04-daily-driver-stack/02-video-audio-pipewire.md). **TURN THIS ON.**
- ☐ **Download and install updates while installing** — makes the installer download apt updates mid-install. **TURN THIS OFF.**

## The recommendation: Minimal + codecs ON + updates-during-install OFF

```
Installation type:          ● Minimal
                            ○ Normal
                            ○ Extended

☑ Install third-party software for graphics and Wi-Fi hardware, and additional media formats
☐ Download and install updates while installing
```

## Why Minimal (and not Normal or Extended)

1. **You want a curated app list, not Ubuntu's 2016-era opinions.** The Normal flavour's LibreOffice you can install in 20 seconds via apt or Flatpak when needed. The Full flavour's KDE-Edu suite is aimed at schools; it is dead weight here.
2. **KDE PIM (KMail/Kontact) is a support burden you didn't sign up for.** It auto-indexes your `~`, adds Akonadi to SDDM's session start, and runs `akonadictl` in the background. Minimal omits it entirely. Normal and Extended both include it and then [2.1 debloat](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md) has to rip it out.
3. **Boot time is directly proportional to installed service count.** Fewer apt-installed services = fewer SDDM-session `*.desktop` autostarts = fewer systemd units = lower `systemd-analyze` total. On a fresh install this delta is ~1.5 s to login prompt, but it compounds because fewer services also means fewer incompatibilities with future kernel and Plasma upgrades.
4. **Curated > opinionated.** You and I are going to pick the music player in [4.4](../04-daily-driver-stack/04-music-streaming-local.md), the comms stack in [4.5](../04-daily-driver-stack/05-comms-and-productivity.md), and the editor in [5.4](../05-web-development/04-editors-and-ides.md). Having the Ubuntu version of all three pre-installed just means you have to uninstall first.
5. **Disk footprint on a 512 GB SSD is not the issue; first-boot entropy is.** Saving 5 GB of disk doesn't matter. Saving 40 pre-installed apt packages that you didn't choose and that Ubuntu will security-update from now until 2031, does.

## Why codecs ON

Kubuntu is legally conservative about patented codecs — H.264 decode, HEVC, MP3 encoding, some AAC variants — so by default the installer ships only fully-free codecs. The third-party checkbox installs the Ubuntu Restricted Extras metapackage silently. Without it, Firefox/Chromium cannot play most web video (YouTube uses VP9, which is free, but the web is still 50 % H.264), mpv cannot play your phone's video export, and Discord's screenshare breaks.

There is no downside. Ubuntu's repos contain vendor-agreed codec packages that are legal in virtually all jurisdictions; the checkbox has been a net-positive default since 22.04.

## Why download-updates OFF

Three reasons:

1. **The installer cannot resume a partial apt download if Wi-Fi drops mid-install.** A fresh `sudo apt update && sudo apt full-upgrade` after first boot (in [1.5](05-first-boot-and-snapshot.md)) is resumable and logged.
2. **The installer applies updates without you seeing the changelog, and in particular without you seeing kernel/driver upgrades.** Better to take the ISO-version kernel on first boot, confirm the system boots, take a Timeshift snapshot, *then* run `apt full-upgrade` — so you have a known-good pre-update state.
3. **On a slow install-location Wi-Fi (coffee shop, hostel) it turns a 15-minute install into a 60-minute install with no benefit.** The same apt operation after first boot can happen on your home connection.

## What Minimal leaves out that you will install back anyway

These are the things you will re-add, but by your choice, from your preferred channels:

| Category                 | Package set                                                                            | Installed in               |
| ------------------------ | -------------------------------------------------------------------------------------- | -------------------------- |
| Office                   | LibreOffice (from apt) or OnlyOffice (Flatpak)                                         | [4.5](../04-daily-driver-stack/05-comms-and-productivity.md) |
| Music                    | Spotify (Flatpak) + Strawberry/Tauon (apt)                                             | [4.4](../04-daily-driver-stack/04-music-streaming-local.md)  |
| Video                    | mpv (apt) + VLC (Flatpak)                                                              | [4.2](../04-daily-driver-stack/02-video-audio-pipewire.md)  |
| Communication            | Signal / Telegram / Discord (Flatpak)                                                  | [4.5](../04-daily-driver-stack/05-comms-and-productivity.md) |
| Editor                   | VS Code + Cursor                                                                       | [5.4](../05-web-development/04-editors-and-ides.md)          |
| Mail (if you want KDE's) | `kmail`, `akonadi` ... explicitly via `sudo apt install`                               | optional                   |

## What Minimal keeps that you do actually want

- `firefox` via Mozilla team APT — set up on first boot in [2.1](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md) but the Minimal bundle already provides a working browser for the initial `apt full-upgrade` step.
- `konsole`, `dolphin`, `kate`, `spectacle`, `ksystemlog`, `partitionmanager` — the core KDE utility suite.
- `discover` — KDE's GUI app manager. Useful for confirming Flathub integration in [2.1](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md); you can remove it after if you prefer CLI apt/flatpak.
- `sddm`, `plasma-desktop`, `plasma-workspace`, `kwin-wayland` — the desktop itself.
- `network-manager`, `bluez`, `cups`, `pipewire`, `wireplumber` — the system-level services you need for networking, Bluetooth, printing, audio. Minimal keeps these.
- `linux-image-generic`, `linux-headers-generic`, `linux-firmware` — kernel + microcode. Minimal keeps these.

## If you really want Normal

You are not wrong to pick Normal. The case for Normal is "I want LibreOffice immediately, and I am fine with ripping out KDE PIM afterwards." Both are valid. If you pick Normal:

- The [2.1 debloat](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md) step will still work — it simply has more to remove.
- Expect ~2 additional minutes of apt removal on first debloat run.
- Expect ~3 GB more installed; irrelevant on 512 GB.

**Do not pick Extended / Full unless you have a specific reason (you teach from the KDE-Edu apps, or you want to ship Krita from the repos rather than Flatpak).** It adds ~2 GB of apps you didn't ask for and will uninstall.

## Other installer screens — quick notes

The rest of Calamares on 26.04 is uncontroversial, but two items to flag:

- **Keyboard layout:** pick yours and leave the "variant" at default. Any remapping (CapsLock → Ctrl/Esc) we do post-install with KeyD, not with keyboard layout changes — it's more portable and survives session/Wayland/X11 swaps.
- **Timezone:** set it correctly. This affects the `UTC=yes` default in `/etc/adjtime` and Windows's interpretation of the hardware clock. If you dual-boot Windows and see a time skew, the fix is in [10-troubleshooting.md § clock skew](../10-troubleshooting.md).
- **User account:** pick a short username (e.g. your first name). Pick a password you can type without looking. **Do NOT enable "Log in automatically"** on a laptop that leaves the house — the five-second password entry is worth the privacy.
- **Hostname:** pick something you will recognise in `ssh` logs. `tuf-a17` or `resolute` works. Stay away from anything containing spaces or emoji.

Proceed to [1.4 installation and partitioning](04-installation-and-partitioning.md) for the meaningful Calamares screens.

---

[Home](../README.md) · [↑ 01 Install](README.md) · [← Previous: 1.2 Windows prep & BIOS](02-windows-prep-and-bios.md) · **1.3 Installer flavor** · [Next: 1.4 Partitioning →](04-installation-and-partitioning.md)
