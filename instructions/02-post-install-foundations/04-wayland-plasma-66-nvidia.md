[Home](../README.md) · [↑ 02 Post-Install](README.md) · [← Previous: 2.3 ASUS TUF](03-asus-tuf-hybrid-gpu.md) · **2.4 Wayland + Plasma 6.6 + NVIDIA** · [Next: 2.5 Snapshot policy →](05-snapshot-policy-btrfs-timeshift.md)

---

# 2.4 Wayland + Plasma 6.6 + NVIDIA 580+ Tuning

The Kubuntu 26.04 default session is **Plasma 6.6 on Wayland** with the NVIDIA 580 proprietary driver running in Explicit Sync mode. This is the first Kubuntu LTS where that combination is production-quality; the [`docs/` v1 verdict](../../docs/01-verdict.md) explicitly steered away from it on 24.04 because Plasma 5.27 + Wayland + NVIDIA 550 was rough. Five kernel cycles and three NVIDIA branches later, the story is different.

## What "Explicit Sync" means and why you care

Historically, Wayland assumed a compositor-driven implicit sync: the compositor would wait for GPU rendering to complete before presenting a frame. NVIDIA's driver model didn't fit this assumption, resulting in flicker, tearing, and frame pacing issues under load. **Explicit Sync** is a Wayland protocol addition (merged in 2024) where the client (app) explicitly signals "my frame is ready" via a fence, and the compositor schedules presentation based on that fence.

On 26.04:

- **KWin 6.6**: Explicit Sync support, enabled when driver advertises it.
- **NVIDIA 580+**: Advertises Explicit Sync support.
- **Firefox 148, Chromium 131+, Electron 30+**: Use Explicit Sync when the compositor offers it.

Result: the flicker-on-video-playback, dropped-frames-under-load, and ghosting-during-scroll issues that plagued Wayland + NVIDIA are largely resolved on this combination. Not literally zero — a handful of edge cases still exist (see [10-troubleshooting.md](../10-troubleshooting.md) § NVIDIA Wayland) — but 90th-percentile daily use is smooth.

## 2.4.1 Verify Explicit Sync is active

```bash
# KWin's protocol list:
qdbus org.kde.KWin /KWin supportInformation 2>/dev/null | grep -i "sync" | head -10
# Expect: lines mentioning "explicit sync" as "supported" or "enabled".

# Alternative via the debug console (KRunner → "KWin Debug Console"):
# Go to "Wayland Clipboard" tab; check "Supported protocols" list; look for "wp_linux_drm_syncobj_v1".
```

If Explicit Sync is NOT listed as supported:

1. NVIDIA driver is older than 580 — re-check [2.2](02-nvidia-driver-580-and-cuda.md).
2. Kernel modeset parameter missing. Add to kernel cmdline (covered in [3.1](../03-speed-and-boot-optimizations/01-kernel-boot-parameters.md)):
   ```
   nvidia-drm.modeset=1 nvidia-drm.fbdev=1
   ```
3. KWin built without the feature (shouldn't happen on 26.04; check `apt policy kwin-wayland`).

## 2.4.2 Fractional scaling on HiDPI displays

The TUF A17's 17.3" 1080p display at 144 Hz does not need fractional scaling (the DPI is ~128, which is fine at 1x). But if you dock to an external 4K monitor:

- **System Settings → Display Configuration → Display Scale** → drag the per-monitor slider to 125 % / 150 % / 200 % as needed.
- On Plasma 6.6 this uses the `wp_fractional_scale_v1` protocol, which KWin + most Qt apps + Firefox + Chromium now honor. GTK apps will use the nearest integer (100 %, 200 %) unless `GDK_SCALE=2` is set via `.config/environment.d/`.

## 2.4.3 XWayland — for legacy X11 apps

Most apps are natively Wayland on 26.04. The exceptions:

- Older Electron apps (pre-v30): fall back to X11, run through XWayland.
- Some Qt 5 apps (if you install any — rare in a 26.04-native install): run through XWayland.
- Games under Steam/Proton: vast majority still X11 inside Proton; XWayland is the default path.
- ROS 2 tools (RViz2, Gazebo-Classic): run on X11 inside the Distrobox; XWayland surfaces them to the host.

**You cannot fully escape XWayland on this Plasma session** — and that's fine. It's a well-maintained bridge on 26.04 with full GPU acceleration via the NVIDIA driver's `EGLStream-on-XWayland` compatibility layer.

Verify XWayland is running:

```bash
ps aux | grep -i xwayland | head -2
# Expect: /usr/bin/Xwayland :0 ... (started by KWin as needed)

# A legacy X11 app to verify:
xeyes &
# The two-eye GUI appears. It's an XWayland client. Close it:
pkill xeyes
```

## 2.4.4 Screen sharing via PipeWire portal

Wayland's security model disallows apps from directly reading the framebuffer. Instead, screen sharing (OBS, Discord, Zoom, browser WebRTC) goes through `xdg-desktop-portal-kde` → PipeWire → the compositor. Install and enable:

```bash
sudo apt install -y xdg-desktop-portal-kde pipewire-pulse wireplumber
sudo systemctl --user enable --now pipewire.service pipewire-pulse.service wireplumber.service

# Restart the portal after install:
systemctl --user restart xdg-desktop-portal.service xdg-desktop-portal-kde.service
```

### Test with OBS Studio (Flatpak)

```bash
flatpak install -y flathub com.obsproject.Studio
flatpak run com.obsproject.Studio &
# Add source → "Screen Capture (PipeWire)" → pick your display or a window.
# A KDE permission dialog should appear asking "Allow OBS to share the screen?"
# Approve.
```

If the PipeWire option is missing from OBS's source picker, the portal isn't discoverable. Check:

```bash
systemctl --user status xdg-desktop-portal-kde.service
journalctl --user -u xdg-desktop-portal-kde.service -n 50
```

## 2.4.5 Browser hardware acceleration

Firefox and Chromium on 26.04 Wayland: by default, GPU acceleration is enabled but video decode falls back to software on some setups. Explicit fix:

### Firefox

`about:config`, set:

```
media.ffmpeg.vaapi.enabled = true
gfx.webrender.all = true
widget.dmabuf.force-enabled = true          (Wayland dmabuf)
media.rdd-ffmpeg.enabled = true
```

Verify via `about:support` → "Compositing" line should read `WebRender (Hardware)` and "Hardware H264 Decoding" should say `Yes, by default`.

### Chromium / Brave / Cursor (Chromium-based)

Launch with flags (persistent via `.desktop` file or wrapper script):

```
--ozone-platform-hint=auto --enable-features=VaapiVideoDecoder,VaapiVideoEncoder,WebRTCPipeWireCapturer
```

Or, for Brave specifically, go to `brave://flags` and enable:

- `#ozone-platform-hint` → "Auto"
- `#enable-accelerated-video-decode` → "Enabled"

The daily-driver browser setups in [4.1](../04-daily-driver-stack/01-browsers.md) cover each in detail.

## 2.4.6 Fontconfig tweaks

Plasma 6.6 defaults are fine but anti-aliasing and hinting can be tuned to taste.

System Settings → Fonts → Force font DPI to match your monitor (usually leave on auto).

For terminal/editor fonts, use a modern programming font and size it appropriately:

```bash
sudo apt install -y fonts-firacode fonts-hack fonts-jetbrains-mono fonts-cascadia-code fonts-noto-color-emoji
fc-cache -f
```

My pick for Konsole and VS Code: **JetBrains Mono Nerd Font** (install via [nerdfonts.com](https://www.nerdfonts.com/font-downloads)). Covered in [4.6](../04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md).

## 2.4.7 Plasma 6.6 session quality-of-life

Changes from 5.27 that you want to know about:

- **Per-virtual-desktop tiling** — each virtual desktop can have its own default tile layout. Tile management in [4.6](../04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md).
- **New Spectacle** — the screenshot tool has a redesigned UI with better area/window/region selection. Prefix keys unchanged (`PrintScreen` for whole, `Meta+Shift+S` for region).
- **HDR wizard** (System Settings → Display → HDR) — irrelevant for the TUF A17's SDR panel, but present for external HDR displays.
- **Color-profile-per-display** — if you have an external calibrated monitor, colour profiles now follow the display (not the output port).
- **KRunner has per-provider toggle** — prune providers that waste cycles (old file history, KDE Help Center) in Settings → Search.

## 2.4.8 The break-glass: X11 fallback session

If Plasma 6.6 Wayland breaks badly (login loop, cursor vanishes, dGPU refuses to initialise), you can fall back to X11. **This is a troubleshooting escape, not a recommendation — prefer fixing Wayland first via [10-troubleshooting.md](../10-troubleshooting.md).**

```bash
# Install the X11 session assets (not installed by default on 26.04):
sudo apt install -y plasma-workspace-x11 xserver-xorg-video-nvidia-580
```

After install, SDDM's session picker will show a "Plasma (X11)" option.

Limitations of the X11 session on 26.04:

- **No upstream testing.** Kubuntu 26.04 QA tested Wayland only. X11 is a community-maintained side channel.
- **Plasma 6.6 X11 has known bugs** — e.g. window decoration blur, compositor stutter under load. These are not going to be fixed.
- **Fractional scaling doesn't work on X11.**

Use for a day to unblock yourself; fix Wayland and switch back.

## 2.4.9 Wake/resume behaviour

Close the lid, wait 10 seconds, open. Does the display come back or stay black? On the TUF A17 with 580 + Hybrid + `asusd`:

- **Expected:** display wakes in ~1 second, Wi-Fi reconnects in ~5 seconds, battery icon refreshes.
- **If stuck black screen:** known Ampere + Wayland issue on some firmware revisions. Kernel parameters to fix in [3.1](../03-speed-and-boot-optimizations/01-kernel-boot-parameters.md) — specifically `nvidia.NVreg_PreserveVideoMemoryAllocations=1` and `nvidia.NVreg_EnableGpuFirmware=1`.

## 2.4.10 Snapshot

```bash
sudo timeshift --create --comments "post-wayland-tuning $(date -I)" --tags D
```

Proceed to [2.5 Snapshot policy: BTRFS + Timeshift](05-snapshot-policy-btrfs-timeshift.md).

---

[Home](../README.md) · [↑ 02 Post-Install](README.md) · [← Previous: 2.3 ASUS TUF](03-asus-tuf-hybrid-gpu.md) · **2.4 Wayland + Plasma 6.6 + NVIDIA** · [Next: 2.5 Snapshot policy →](05-snapshot-policy-btrfs-timeshift.md)
