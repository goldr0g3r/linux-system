[Home](../README.md) · [↑ 04 Daily Driver](README.md) · [← Previous: 4.5 Comms & productivity](05-comms-and-productivity.md) · **4.6 Plasma UX** · [Next: 05 Web dev →](../05-web-development/README.md)

---

# 4.6 Plasma 6.6 UX — Tiling, Activities, Konsole, Fonts

Plasma 6.6 brought native per-virtual-desktop tiling, a stronger Activities model, and a new Spectacle. This page turns the stock desktop into a productive, keyboard-driven workspace for "one laptop, many workflows."

## The four levers

1. **Virtual desktops** — simple "swipe between 4 screens" layout, best for a single workflow that spans multiple apps.
2. **Activities** — per-context session states (window lists, wallpapers, widgets, file-open-history). Best for "I have 3-4 unrelated workflows and want them separate."
3. **Tiling** — native KWin tile manager in 6.6; per-virtual-desktop layouts.
4. **Konsole / Yakuake** — terminal setup.

## 4.6.1 Virtual desktops — the baseline

System Settings → **Virtual Desktops** → set to **4 desktops** in a **1 row × 4 column** grid.

Naming:

1. **Daily** — browser, comms, music.
2. **Code** — editor, terminal, browser with docs.
3. **ROS / AI** — Distrobox terminals, RViz, ollama webui.
4. **Research** — second browser profile, reading apps, Obsidian.

Shortcuts:

- `Meta+Ctrl+Left/Right` — switch desktop.
- `Meta+Ctrl+Shift+Left/Right` — move current window to left/right desktop.
- `Meta+F1/F2/F3/F4` — jump directly to desktop 1/2/3/4 (remap in System Settings → Shortcuts → KWin).

## 4.6.2 Activities — the power tool

Activities differ from virtual desktops in three ways:

1. Each Activity has its own window list. Closing an Activity doesn't close windows; it hides them. Reopening the Activity brings them back.
2. Each Activity can have its own wallpaper, widgets, dock configuration.
3. File Open dialogs remember per-Activity history.

When to use Activities instead of virtual desktops:

- You have 3-4 wildly different contexts that rarely intermix (e.g. `WebDev`, `Robotics`, `Personal`, `Research`).
- You want a "pause" and "resume" model — work on a project, switch to another, come back to the first with everything still there.

### Setting up Activities

1. System Settings → **Activities** (or press `Meta+Q` for the Activity switcher).
2. Create activities:
   - **Daily** (default)
   - **WebDev**
   - **Robotics**
   - **AI-ML**
3. For each Activity: right-click desktop → Configure Desktop Activity → set a distinct wallpaper (so your eye recognises the Activity at a glance).
4. Optionally: pin different apps to the Task Manager per Activity.

Shortcuts:

- `Meta+Tab` — Activity switcher.
- `Meta+Q` — same.

### Pinning windows to an Activity

Right-click a window title bar → **More Actions → Activities → Select** → pick the Activities where this window should appear. "All activities" is the default. Pinning Konsole with a ROS Distrobox profile to the Robotics Activity only means it never clutters Daily.

## 4.6.3 KWin tiling (per-desktop)

Plasma 6.6's KWin ships a native tile manager. Activate:

- Press `Meta+T` to open the Tile Manager.
- The current virtual desktop's tile layout appears on-screen.
- Drag splits to create a 2-column-2-row layout or whatever you want.
- Save per-desktop (automatically).

Or launch a window into a tile:

- `Super+drag window to screen edge` — snaps to half.
- `Super+arrow` — move window between tiles.
- `Super+Shift+arrow` — resize tile.

Default layouts I use:

- **Daily desktop**: 2x2 — browser, comms, Konsole, music player.
- **Code desktop**: 3 columns — editor 50 %, terminal 25 %, browser 25 %.
- **ROS desktop**: 2 columns — RViz2 60 %, Konsole stack 40 %.

If KWin's native tiling is too stiff, install Krohnkite (classic tiling scripts; works on 6.6 but rougher than native):

```bash
# Only if you want an i3-like tiling experience beyond KWin's 6.6 native tile manager:
git clone https://github.com/esjeon/krohnkite.git
cd krohnkite
plasmapkg2 -t kwinscript -i .
```

Then System Settings → Window Management → KWin Scripts → enable Krohnkite.

## 4.6.4 KRunner shortcuts

KRunner is the KDE quick-launcher. Default shortcut: `Alt+Space` or `Meta`. Configure:

System Settings → **Search** → **KRunner**:

- **Providers to enable**: Applications, Bookmarks, Calculator, Desktop Sessions, File Content, Services, Settings, Shell, System Settings, Unit Conversion, Web Search Keywords. **Disable** KDE Help Center, old file history, and anything not relevant — each disabled provider is one less query per keystroke.
- **Web shortcuts**: set `ddg` (DuckDuckGo), `gh` (GitHub repo search), `aw` (Arch Wiki), `mdn` (Mozilla MDN), `apt` (Ubuntu package search).

Type examples:

- `Alt+Space` then `5*6+2` → 32 in calculator.
- `Alt+Space` then `gh:cursor-ai/ide` → opens GitHub to cursor-ai org / ide repo.
- `Alt+Space` then `apt:distrobox` → searches Ubuntu packages.
- `Alt+Space` then `ssh myserver` → opens Konsole with `ssh myserver` pre-filled.

## 4.6.5 Global keyboard shortcuts

System Settings → **Keyboard** → **Shortcuts** → **Global Shortcuts** — edit as desired. My non-default bindings:

| Shortcut              | Action                                        |
| --------------------- | --------------------------------------------- |
| `Meta+Return`         | Open Konsole                                  |
| `Meta+E`              | Open Dolphin (file manager)                   |
| `Meta+L`              | Lock screen                                   |
| `Meta+Shift+L`        | Log out                                       |
| `Meta+Space`          | KRunner                                       |
| `Meta+F1..F4`         | Switch to virtual desktop N                   |
| `Meta+Ctrl+F1..F4`    | Move current window to virtual desktop N      |
| `Meta+Q`              | Activity switcher                             |
| `Meta+T`              | KWin Tile Manager                             |
| `Meta+Shift+S`        | Spectacle region screenshot                   |
| `Meta+Shift+R`        | Spectacle screen recording                    |
| `Meta+Tab`            | Task switcher (hold to preview)               |
| `Meta+\``             | Yakuake (drop-down terminal)                  |

## 4.6.6 Konsole setup

Konsole is the default terminal. Profiles are the best-kept productivity feature.

### Install a good monospaced font

```bash
sudo apt install -y fonts-firacode fonts-hack fonts-jetbrains-mono fonts-cascadia-code

# Nerd Font variants for icons (Starship, fzf, eza):
mkdir -p ~/.local/share/fonts
curl -sL -o /tmp/JetBrainsMonoNL.zip \
  https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip
unzip -o /tmp/JetBrainsMonoNL.zip -d ~/.local/share/fonts/JetBrainsMonoNerd
fc-cache -f
```

### Profiles

Konsole → Settings → Configure Konsole → **Profiles** → New. Create profiles for:

- **Default** — zsh on host, `JetBrainsMono Nerd Font 11`, Breeze colour scheme, history size 10 000 lines.
- **ROS-Jazzy** — Command: `distrobox enter ros2-jazzy`, starts in `~/work/ros2`.
- **Dev-Postgres** — Command: `psql -h localhost postgres`, connects to the rootless Podman Postgres from [5.5](../05-web-development/05-frameworks-and-databases.md).
- **Sys-Root** — `sudo -i`, different colour scheme (red tint) so you never forget you're root.

Each profile becomes a menu entry in Konsole's "File → New Tab →" submenu. Right-click any tab to switch profile mid-session.

### Colour scheme

Breeze is fine. If you want a darker feel, **Nord** and **Catppuccin Mocha** look great on Wayland high-DPI:

Konsole → Manage Profiles → Appearance → New → paste a .colorscheme file from https://github.com/catppuccin/konsole.

### Tmux-like tabs and splits

Konsole supports splits natively: `Ctrl+Shift+(` for horizontal split, `Ctrl+Shift+)` for vertical. Use instead of tmux if you don't need persistence across SSH.

## 4.6.7 Yakuake — drop-down terminal

Yakuake is a Konsole variant that drops from the top of the screen on a hotkey. Great for quick one-liners.

```bash
sudo apt install -y yakuake
```

Launch, then Settings → Configure Yakuake → **General**:

- Shortcut: `Meta+``` (backtick).
- Screen: monitor you work on most.
- Height: 60 %.
- Auto-hide when losing focus: optional (I leave it off; hotkey toggles).

Now anywhere, anytime, a quick terminal is one keypress away.

## 4.6.8 Dolphin tuning

The default file manager is fine out of the box. A few tweaks:

- **Show hidden files by default**: Ctrl+H to toggle.
- **Detail view** for programmers — shows Size / Modified / Type columns.
- **Sort by Date Modified (descending)** — I keep this on for ~/Downloads.
- **Git integration**: right-click a git repo → "Show Git menu" → commit / pull / push from file manager.

Plugins:

```bash
sudo apt install -y dolphin-plugins
```

## 4.6.9 Fonts — the system-wide polish

Set consistent fonts:

System Settings → **Fonts**:

- **General**: `Noto Sans` (default) or `Inter` (install via `sudo apt install fonts-inter`) → 10 pt.
- **Fixed width**: `JetBrains Mono Nerd Font` → 10 pt.
- **Small**: `Noto Sans` → 8 pt.
- **Toolbar / Menu**: `Noto Sans` → 10 pt.
- **Window title**: `Noto Sans Bold` → 10 pt.

Anti-aliasing: enabled. Hinting: slight. Sub-pixel rendering: RGB.

## 4.6.10 Spectacle (screenshots) — 26.04 changes

The new Spectacle in 26.04 has:

- **Rectangular region** (default, `Meta+Shift+S`).
- **Window** (click a window to capture it).
- **Screen** (whole display).
- **All screens** (multi-monitor).
- **Timer** (5s delay).
- **Video recording** (new in 6.6 — `Meta+Shift+R`).

Output default: `~/Pictures/Screenshots/Screenshot_YYYY-MM-DD_HH-MM-SS.png`.

Right-click → Copy to clipboard, or directly upload to an image host (Imgur built-in).

## 4.6.11 Wallpaper and desktop aesthetics

Right-click desktop → Configure Desktop → choose wallpaper. The stock Kubuntu 26.04 "Resolute Raccoon" wallpaper is the raccoon silhouette against a twilight sky; fine as a default.

If you want per-Activity wallpapers, set each under its respective Activity.

Icon theme: **Breeze** (default). **Papirus** (via `sudo apt install papirus-icon-theme`) is a popular alternative.

## 4.6.12 Night-mode / redshift

Plasma 6.6 has native Night Color: System Settings → Display → Night Color → Always on or scheduled. Gentle 4500 K on evenings, 3500 K for late-night coding. Lower than 3000 K is excessive.

## 4.6.13 Snapshot

```bash
sudo timeshift --create --comments "post-plasma-ux $(date -I)" --tags D
```

Section 04 complete. Proceed to [05 Web development](../05-web-development/README.md).

---

[Home](../README.md) · [↑ 04 Daily Driver](README.md) · [← Previous: 4.5 Comms & productivity](05-comms-and-productivity.md) · **4.6 Plasma UX** · [Next: 05 Web dev →](../05-web-development/README.md)
