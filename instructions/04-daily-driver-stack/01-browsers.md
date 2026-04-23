[Home](../README.md) · [↑ 04 Daily Driver](README.md) · [← Previous: 04 Daily Driver (section)](README.md) · **4.1 Browsers** · [Next: 4.2 Video & audio →](02-video-audio-pipewire.md)

---

# 4.1 Browsers

The browsers you will actually use, installed from the right channels, configured for privacy and performance on Plasma 6.6 Wayland + NVIDIA.

## The stack

| Browser         | Role                                          | Channel                      | Install command                                   |
| --------------- | --------------------------------------------- | ---------------------------- | ------------------------------------------------- |
| **Firefox**     | Primary daily driver                          | APT (Mozilla upstream)       | already installed in [2.1](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md#213-firefox-via-mozillas-apt-repository) |
| **Brave**       | Secondary; Chromium tests, YouTube with built-in ad-block | APT (Brave upstream)  | `sudo apt install brave-browser`                  |
| **Zen Browser** | Tertiary; productivity-focused Firefox fork with tabs-as-trees, workspaces | Flatpak                | `flatpak install flathub app.zen_browser.zen`     |
| **Chromium**    | Tabula rasa for rendering tests / incognito contexts | Flatpak (no snap)     | `flatpak install flathub org.chromium.Chromium`   |

You do not need all four. Pick Firefox as primary, then one or two of the other three by taste.

## Snapshot first

```bash
sudo timeshift --create --comments "before-browsers $(date -I)" --tags D
```

## 4.1.1 Firefox (primary) — Wayland + hardware decode tuning

Firefox 148 on 26.04 is already in place from [2.1.3](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md#213-firefox-via-mozillas-apt-repository). Confirm:

```bash
firefox --version
# Expect: Mozilla Firefox 148.x

apt policy firefox | head -4
# Installed: 148.x from packages.mozilla.org
```

### Configure for Wayland + VA-API

URL: `about:config` → accept the "Proceed with Caution" dialog. Set these keys:

| Key                                   | Value   | Why                                                           |
| ------------------------------------- | ------- | ------------------------------------------------------------- |
| `media.ffmpeg.vaapi.enabled`          | `true`  | Hardware video decode via VA-API (AMD iGPU)                   |
| `media.rdd-ffmpeg.enabled`            | `true`  | Use the RDD process for ffmpeg decode (sandboxed)             |
| `gfx.webrender.all`                   | `true`  | GPU-accelerated compositor                                    |
| `widget.dmabuf.force-enabled`         | `true`  | Zero-copy buffer passing on Wayland                           |
| `widget.wayland.fractional-scale.enabled` | `true` | Honour fractional scale on HiDPI externals                 |
| `browser.download.manager.closeWhenDone` | `true` | Close the download panel when done                         |
| `browser.urlbar.suggest.topsites`     | `false` | Hide the top-sites list from the new tab page               |
| `browser.newtabpage.activity-stream.showSponsored` | `false` | Disable sponsored tiles                     |
| `browser.startup.homepage`            | `about:home` or your choice | |
| `privacy.resistFingerprinting`        | `true`  | Only if you can tolerate some sites breaking; significantly harder to fingerprint |

Restart Firefox.

Verify hardware decode is live:

```bash
# In a terminal while a YouTube 1080p video plays:
intel_gpu_top 2>/dev/null || radeontop 2>/dev/null
# Should see the iGPU's VCN engine active.
```

Or just open a 4K YouTube video and watch CPU usage — if it stays under 30 % total, VA-API is working. Without hw-decode, 4K pegs 1-2 cores of a Ryzen 4800H.

### Extensions (install once, configure)

- **uBlock Origin** — the only content blocker you need. Strongest rules list: stock default + `EasyList Cookie` + `uBlock filters - Annoyances`.
- **Bitwarden** — password manager. Install the extension AND the `bitwarden-cli` via apt for CLI integration.
- **SponsorBlock** — skip sponsors in YouTube videos.
- **Vimium** — keyboard navigation for sites that cooperate.
- **Tree Style Tab** — vertical tab tree in the sidebar; essential for >20 open tabs.
- **LanguageTool** — grammar checking in textareas.

### Profile backup

Firefox profile lives at `~/.mozilla/firefox/<random>.default-release/`. Back it up (part of the [8.2 Restic](../08-productivity-security/02-backup-strategy-3-2-1.md) backup). Syncing across machines via Mozilla Sync is convenient; for a single-machine setup, just ensure the profile dir is in your Restic include list.

### Multi-profile workflow

Running multiple Firefox profiles for context separation (personal / work / research) is a one-line convention:

```bash
# Launch profile manager:
firefox -ProfileManager -no-remote

# Or launch a specific profile directly:
firefox -P work -no-remote &
```

Create `~/.local/share/applications/firefox-work.desktop`:

```ini
[Desktop Entry]
Version=1.0
Name=Firefox (Work)
Exec=firefox -P work -no-remote
Icon=firefox
Terminal=false
Type=Application
Categories=Network;WebBrowser;
```

Refresh KRunner/Plasma menus (`kbuildsycoca6 --noincremental`). Now "Firefox (Work)" appears in the launcher as a separate app.

## 4.1.2 Brave (secondary Chromium)

Brave is Chromium with reasonable defaults (ad-block built-in, no Google telemetry, Tor windows). Install from Brave's APT repo:

```bash
sudo curl -fsSLo /etc/apt/keyrings/brave-browser-archive-keyring.gpg \
  https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/brave-browser-archive-keyring.gpg] https://brave-browser-apt-release.s3.brave.com/ stable main" | \
  sudo tee /etc/apt/sources.list.d/brave-browser-release.list

sudo apt update
sudo apt install -y brave-browser
```

### Wayland + VA-API for Brave

Edit `~/.config/brave-flags.conf`:

```
--ozone-platform-hint=auto
--enable-features=VaapiVideoDecoder,VaapiVideoEncoder,WebRTCPipeWireCapturer
--disable-features=UseChromeOSDirectVideoDecoder
```

Or launch with `brave-browser --ozone-platform-hint=auto` explicitly. On Wayland, `auto` resolves to `wayland` natively; on XWayland fallback it goes to `x11`.

Verify via `brave://gpu` — "Video Decode" should say "Hardware accelerated".

### Brave vs Firefox daily-use split

I use Brave specifically for:

- YouTube (built-in ad-block on mobile YouTube app)
- Google Workspace (better compat with Google Meet)
- Anything that wants "Chrome or Chromium"

Firefox for everything else.

## 4.1.3 Zen Browser (productivity Firefox fork)

Zen is a Firefox fork opinionated about tab management and workspaces. Think Arc Browser but open-source and Qt-native. Install via Flatpak:

```bash
flatpak install -y flathub app.zen_browser.zen
```

Launch:

```bash
flatpak run app.zen_browser.zen
```

Zen features that justify its existence:

- **Workspaces** — named tab groups you switch between, like Arc. Each workspace has its own tab bar.
- **Tabs as a tree** in the sidebar, built-in (no Tree Style Tab extension needed).
- **Essentials** — pinned tabs across workspaces.
- **Compact mode** — hides chrome, content fills the window.

Use Zen when your work benefits from workspace separation (e.g. "subsea-research" workspace with 20 tabs, separate from "daily" workspace). Skip it if Firefox's tab container extensions already scratch that itch.

## 4.1.4 Chromium (rare test-only)

If you need a clean Chromium (no Brave opinions, no Google Chrome telemetry), install via Flatpak:

```bash
flatpak install -y flathub org.chromium.Chromium
```

Note: this is the upstream Chromium, compiled by Flatpak build infrastructure from the Chromium source tree. Installs and updates are slower than Brave (Flatpak's build farm is smaller than Brave's CI), but it's a true zero-telemetry Chromium.

Do not install `chromium-browser` from `apt` — that package on Ubuntu-descended systems is a transition package that pulls in the snap. Use Flatpak.

## 4.1.5 Default browser selection

```bash
# List registered browsers:
update-alternatives --display x-www-browser

# Set default (Firefox):
sudo update-alternatives --config x-www-browser
sudo update-alternatives --config gnome-www-browser  # for apps looking for this alternative
```

Or via Plasma: System Settings → **Default Applications** → Web Browser → Firefox.

This propagates to file manager "Open link in browser" contextual actions, KMail's HTML rendering, IRC/chat clients, etc.

## 4.1.6 URL handlers and app-integration

Register Zoom / Signal / Telegram / Matrix links correctly:

```bash
# After installing Signal Desktop (Flatpak, covered in 4.5):
xdg-mime default org.signal.Signal.desktop x-scheme-handler/sgnl

# After installing Telegram:
xdg-mime default org.telegram.desktop.desktop x-scheme-handler/tg
```

Now clicking an `sgnl:` or `tg:` link in Firefox opens the native app instead of the browser's "what to do with this protocol?" prompt.

## 4.1.7 GPU process sanity

With any Chromium-based browser, on the "edge" of Wayland + NVIDIA bugs, you sometimes see the GPU process repeatedly restart. Symptoms: tabs turn black, page refresh recovers. Diagnose:

```bash
# For Brave:
brave-browser --ozone-platform-hint=auto --enable-logging --v=0 2>&1 | grep -i "gpu"
```

If you see repeated `GPU process exited unexpectedly`, the common fix is to add `--disable-gpu-memory-buffer-video-frames` to the flags, or fall back to X11 via `--ozone-platform=x11` temporarily while diagnosing. More in [10-troubleshooting.md](../10-troubleshooting.md).

## 4.1.8 Snapshot

```bash
sudo timeshift --create --comments "post-browsers $(date -I)" --tags D
```

Proceed to [4.2 Video + audio + PipeWire](02-video-audio-pipewire.md).

---

[Home](../README.md) · [↑ 04 Daily Driver](README.md) · [← Previous: 04 Daily Driver (section)](README.md) · **4.1 Browsers** · [Next: 4.2 Video & audio →](02-video-audio-pipewire.md)
