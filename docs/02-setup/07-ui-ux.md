[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.6 ASUS Hardware](06-asus-hardware.md) · **2.7 UI/UX** · [Next: Part 3 →](../03-dev-environment/README.md)

---

# 2.7 UI/UX Tweaks for Engineers

The out-of-box Plasma 5.27 is already good. These tweaks turn it into an **engineering cockpit**: dynamic tiling, multi-context Activities for "Automotive / ROV / M.Tech / Blog", per-project Konsole profiles, Dolphin with embedded git + terminal, KRunner as your universal launcher.

## Install packages

```bash
# KWin dynamic tiling. Bismuth is maintained; Krohnkite is the fork.
sudo apt install -y kwin-bismuth

# Optional second panel / dock dedicated to engineering apps.
sudo apt install -y latte-dock
```

Enable bismuth after install: **System Settings → Window Management → Window Tiling → Enable Bismuth**.

## Konsole power profile

**Settings → Edit Current Profile:**

- **General:** untick "Show menu bar" (Meta+M toggles it), set **scrollback to 100000 lines**.
- **Appearance:** install the "Dracula" or "Gruvbox Dark" color scheme from the KDE Store (the "Get new..." button on the dropdown).
- **Keyboard:** set `$TERM` to `xterm-256color`.
- **Scrolling:** enable "Smooth scrolling", set scrolling speed to max.

**Create per-project / per-Distrobox profiles:**

1. **Profiles → Manage Profiles → New.** Name: `ros2-humble`.
2. **Command:** set to `distrobox enter ros2-humble`.
3. **Tabs → Tab title format:** `%n` → `ROS2-Humble`.

Now a new Konsole tab of profile `ros2-humble` drops you directly into the container. Repeat for `ros2-jazzy`, `embedded-fedora`, `ai` (Python venv), `web` (the blog repo with `fnm` env).

## Dolphin tweaks

Open Dolphin:

- **View → Panels → Terminal** (F4 opens an inline shell synced with the current directory).
- **View → Panels → Information, Folders, Places.** Turn all three on.
- **Configure Dolphin → Services → enable "Git" version control plugin.** You now see git status icons on tracked files and a right-click "Git" menu.
- **View → Split** (F3) for two-pane file management.

Bonus: **Configure Dolphin → Context Menu → Services → Download New Services** — install `Open Terminal Here`, `Extract and Compress`, `Open in VS Code`.

## KDE Activities — your biggest productivity lever

Activities are Plasma's way of storing **entire desktop contexts**: different wallpaper, different docked apps, different widgets, different "recent files". Create four:

1. **Automotive** — wallpaper of a canbus trace; dock: Konsole, CANalyzer (Windows VM), VS Code, SavvyCAN, Wireshark, Slack.
2. **ROV** — wallpaper of subsea environment; dock: Konsole (Distrobox-Jazzy profile), RViz2, Gazebo, QGroundControl, KiCad, Firefox with BlueROV docs pinned.
3. **M.Tech AI** — wallpaper calm; dock: Konsole (AI venv profile), JupyterLab, Obsidian, Zotero, Firefox with arXiv + Hugging Face pinned, Anki.
4. **Blog** — wallpaper inspiring; dock: Konsole (web profile), VS Code, Firefox with localhost:3000, Vercel dashboard, Obsidian (for drafts).

**Create an Activity:** right-click desktop → Activities → + → pick a name and icon.
**Switch:** Meta+Tab cycles; or add the Activity Bar widget to a panel for one-click switches.
**Per-window Activity assignment:** right-click window titlebar → More Actions → Move to Activity → pin to a single Activity so RViz2 always opens in "ROV" and never "Blog".

## KRunner plugins (Meta key — your universal launcher)

System Settings → Search → KRunner. Enable these plugins:

- **Calculator** — `Meta + 2*3.14*5` returns 31.4
- **Unit Conversion** — `Meta + 10 inch in mm`
- **Shell** — `Meta + =ping google.com`
- **SSH Sessions** — reads `~/.ssh/config`, you type `Meta + rov-topside` to SSH in.
- **Krunner-translator** — `Meta + translate Hello to Japanese`
- **Places**, **Recent Documents**, **Bookmarks** — obvious.
- **PowerDevil** — `Meta + suspend`, `Meta + reboot`.
- **Konsole Profiles** — `Meta + ros2-humble` launches that Konsole profile directly.

KRunner replaces every "launcher" tool you have on macOS (Alfred, Raycast) or Windows (PowerToys Run).

## HiDPI / font rendering for the 17" 1080p panel

At 1920x1080 on 17.3", default 96 DPI is too small for reading KiCad footprints at 1:1. Fix:

**System Settings → Fonts →** set **Force Font DPI to 108** (or 120 if eyes prefer). Enable:

- Anti-aliasing: **Enabled**
- Sub-pixel rendering: **RGB**
- Hinting style: **Slight**

Do **not** enable "Scale display" fractional scaling in Plasma 5.27 on X11 — it uses Xft.dpi and works fine; the fractional-scaling slider is for Wayland which we are not on.

## Night Color

**System Settings → Display → Night Color →** enable, set target 4500 K, activation at sunset.

## Panel customization

Default Plasma panel at bottom. Right-click → Edit Panel.

**Add widgets:**

- **Application Menu** (Kickoff) — standard launcher.
- **Task Manager** — pinned apps + running windows.
- **System Tray** — status icons.
- **Digital Clock** — show date.
- **Desktop Pager** — virtual-desktop indicator.
- **Activity Bar** — one-click Activity switch.
- **KeyboardLayoutSwitcher** — if you type multilingual.
- **Network** — connection status.

**Recommended layout:** Left → Application Menu, Activity Bar. Middle → Task Manager. Right → System Tray, Pager, Clock.

## Virtual desktops

**System Settings → Workspace Behavior → Virtual Desktops →** 4 desktops, 1 row. Name them: `Main`, `Code`, `Sim`, `Docs`.

**Shortcut:** Ctrl+F1..F4 to jump, Meta+Ctrl+Arrow to navigate, Meta+Ctrl+Shift+Arrow to move a window between desktops.

Combined with Activities (4 contexts * 4 desktops each = 16 logical panes), you have more window-arrangement real estate than any Windows/macOS user.

## Optional: global Meta key to KRunner (if not default)

**System Settings → Shortcuts → Global Shortcuts → KWin:**

- `Activate Window Demanding Attention`: Alt+Tab
- `Walk Through Windows`: Alt+Tab
- **Ensure Meta (Super key) opens KRunner** (default) — if it opens Application Menu instead, change it in System Settings → Shortcuts → KRunner.

## Done

Reboot once to let all the shortcut/tiling changes fully settle.

```bash
sudo reboot
```

On next login, test:

- **Meta** opens KRunner.
- **Meta+Tab** cycles Activities.
- **Ctrl+F1..F4** jumps desktops.
- **Dolphin** shows git overlays on tracked directories.
- **Konsole → new tab → ros2-humble profile** drops into the container.

You are now ready for [Part 3 — The Developer Environment Blueprint](../03-dev-environment/README.md).

---

[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.6 ASUS Hardware](06-asus-hardware.md) · **2.7 UI/UX** · [Next: Part 3 →](../03-dev-environment/README.md)
