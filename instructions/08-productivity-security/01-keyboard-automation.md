[Home](../README.md) · [↑ 08 Productivity & Security](README.md) · [← Previous: 08 (section)](README.md) · **8.1 Keyboard & automation** · [Next: 8.2 Backup →](02-backup-strategy-3-2-1.md)

---

# 8.1 Keyboard Automation — KRunner, KeyD, AutoKey, and Activities

Ergonomics compounds: a few minutes of one-time setup saves 10-20 minutes per day. This page sets up KeyD (system-wide key remapping), AutoKey (text expansion), and consolidates the KRunner / Activities patterns.

## 8.1.1 KeyD — system-wide key remapping (do this first)

If you haven't already set up KeyD from [5.1.9](../05-web-development/01-shell-and-terminal.md#519-global-key-remap--capslock--ctrlesc-via-keyd), do it now. The remap works in every app: Konsole, VS Code, Firefox, Wayland, X11, everywhere.

```bash
sudo apt install -y keyd

sudo tee /etc/keyd/default.conf >/dev/null <<'EOF'
[ids]
*

[main]
# Caps Lock: Esc when tapped, Ctrl when held.
# This is the single highest-ROI Linux remap.
capslock = overload(control, esc)

# Optional: Right Alt → Compose key (for easy ≈ ñ é emojis):
# rightalt = compose

# Optional: Swap Ctrl and Caps more explicitly (if above syntax doesn't work):
# leftcontrol = capslock
# capslock = leftcontrol
EOF

sudo systemctl enable --now keyd
sudo keyd reload
```

Test:

- Press and release CapsLock → should act as Esc.
- Hold CapsLock + A → should act as Ctrl+A.
- Hold CapsLock + C → Ctrl+C.

Works in KDE, terminal, browser, everywhere. This is the single biggest daily-productivity gain from this whole section.

### Per-application KeyD rules (if wanted)

KeyD can apply different rules to different apps. E.g. in Konsole, remap Ctrl+Tab to switch tabs:

Edit `/etc/keyd/default.conf`:

```ini
[ids]
*

[main]
capslock = overload(control, esc)

[konsole]
# Rules specific to windows with wm_class "Konsole":
control+tab = next_tab    # pseudocode — actual keyd keybind is sending the shortcut Konsole wants.
```

Check `keyd list-keys` for the keys/actions available. For typical desktops, the single `[main]` map is enough.

## 8.1.2 Virtual desktops vs Activities — re-pick

You set both up in [4.6](../04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md). Revisit after using the system for a week:

- Are your virtual desktops organised enough? 4 is the sweet spot for a 17" laptop.
- Are Activities reducing context switching? If you're not using the Activity switcher (`Meta+Tab`), Activities aren't helping and you can fold into virtual desktops only.

My steady-state setup:

- **4 virtual desktops** within each Activity.
- **3 Activities**: Daily, Dev (web + ROS + AI all), Research.

Don't over-engineer. 4 virtual desktops is fine without Activities.

## 8.1.3 KRunner shortcuts I use constantly

The six I use daily, memorise these:

1. **`Alt+Space`, type an app name** — launch. Replaces clicking around the menu.
2. **`Alt+Space`, type a math expression** — calculator.
3. **`Alt+Space`, `=100*USD`** — unit + currency conversion.
4. **`Alt+Space`, `systemctl status ollama.service`** — drop into a terminal with the command pre-filled.
5. **`Alt+Space`, `ddg hello`** — web search (DuckDuckGo).
6. **`Alt+Space`, `gh:cursor/ide`** — GitHub repo lookup.

Setup (if not already done in [4.6.4](../04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md#464-krunner-shortcuts)):

System Settings → Search → **Web Search Keywords** → add:

- `ddg` → https://duckduckgo.com/?q=\\{@}
- `gh` → https://github.com/\\{@}
- `mdn` → https://developer.mozilla.org/en-US/search?q=\\{@}
- `apt` → https://packages.ubuntu.com/search?keywords=\\{@}&searchon=names
- `aw` → https://wiki.archlinux.org/index.php?search=\\{@}
- `pp` → https://pypi.org/search/?q=\\{@}
- `npm` → https://www.npmjs.com/search?q=\\{@}

## 8.1.4 AutoKey — text expansion

"I type my email and home address 10 times a week. Never again."

```bash
sudo apt install -y autokey-gtk

# Or flatpak if apt package is missing:
# flatpak install flathub com.github.autokey.AutoKey
```

Launch AutoKey. Create phrases under the right-pane:

- `mmm` → your full email.
- `addr` → your address.
- `sig` → your email signature.
- `;date` → `$(date -I)` (e.g. `2026-04-23`).
- `;iso` → `$(date -Iseconds)` (e.g. `2026-04-23T14:32:17+00:00`).
- `lgtm` → "Looks good to me. Approved."
- `ptal` → "Please take a look."

Trigger mode: abbreviation (defaults), send with `Tab` or `Space`.

### AutoKey on Wayland — caveat

AutoKey's abbreviation detection relies on X11's keyboard event polling. On Wayland, it works via XWayland for X11-ish apps (most things that aren't fully Wayland-native), but some text fields in fully-Wayland Qt6 apps may not trigger.

Workaround: use Plasma's own **Custom Shortcuts** (System Settings → Shortcuts → Custom Shortcuts → New → Global Shortcut → Command/URL) to bind text insertions:

```bash
# Example: bind Meta+E to type your email:
# System Settings → Shortcuts → Custom Shortcuts → New Global Shortcut → Command/URL:
# echo "you@example.com" | xdotool type --clearmodifiers --file -
# Trigger: Meta+E
```

Works everywhere on Wayland because KDE's global-shortcut system runs at the compositor level.

## 8.1.5 Clipboard manager — Klipper

Plasma's Klipper (system tray clipboard icon) is enabled by default and does what 99 % of clipboard managers do:

- Stores the last 20 clipboard items.
- `Meta+V` shows the history.
- Right-click an entry → copy back to clipboard.

Configure System Settings → **Clipboard**:

- **History size**: 50 (more is bloat).
- **Synchronise Selection & Clipboard** — yes.
- **Clear history on logout** — if you copy sensitive things, yes.

No third-party clipboard manager needed.

## 8.1.6 Screenshot workflow

You set up Spectacle in [4.6.10](../04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md#4610-spectacle-screenshots--2604-changes) and Flameshot in [4.5.3](../04-daily-driver-stack/05-comms-and-productivity.md#flameshot-screenshots-with-annotations). Decide which is your default:

- **Spectacle** for quick rectangle/window/screen shots, directly to clipboard or file.
- **Flameshot** for annotated screenshots (arrows, boxes, redaction) before saving or sharing.

My defaults:

- `PrintScreen` → Spectacle whole-screen → copy to clipboard.
- `Meta+Shift+S` → Spectacle region.
- `Meta+Shift+A` → Flameshot region with annotation toolbar.

Set via System Settings → Shortcuts → Global Shortcuts → Spectacle / Flameshot custom shortcut.

## 8.1.7 KWin scripting — per-virtual-desktop apps

KWin can auto-assign windows to specific virtual desktops when they open. E.g. always open Firefox on desktop 1, Konsole on 2.

System Settings → Window Management → **Window Rules** → New:

- **Match**: Window Class → Regular Expression → `firefox|Navigator`.
- **Rules**: Desktop → Force → 1.

Repeat for:

- Konsole → 2.
- Code / VS Code → 2.
- Discord / Signal / Telegram → 1.
- RViz2 → 3.

Apply; new windows auto-snap to their assigned desktop.

## 8.1.8 Kwin scripts — workspace layouts

Advanced: save and restore window layouts. Install **Quick Tile Screen Edge Maximise** (System Settings → KWin Scripts) for easy half/quarter tiling.

For more, `kdialog` and `kwin-client-list` + a shell script can save/restore layouts, but it's overkill for most uses.

## 8.1.9 Global shortcuts — my preferred bindings

Restating from [4.6.5](../04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md#465-global-keyboard-shortcuts) with additions specific to daily productivity:

| Shortcut              | Action                                        |
| --------------------- | --------------------------------------------- |
| `Meta+Return`         | Konsole                                       |
| `Meta+E`              | Dolphin                                       |
| `Meta+B`              | Firefox                                       |
| `Meta+C`              | VS Code                                       |
| `Meta+M`              | Elisa / music player toggle play              |
| `Meta+Space`          | KRunner                                       |
| `Meta+\``             | Yakuake                                       |
| `Meta+L`              | Lock screen                                   |
| `Meta+Ctrl+L`         | Logout (no shutdown confirmation)             |
| `Meta+Shift+S`        | Spectacle region                              |
| `Meta+Shift+A`        | Flameshot annotated region                    |
| `Meta+Shift+R`        | Spectacle screen record                       |
| `Meta+1/2/3/4`        | Switch to virtual desktop N                   |
| `Meta+Ctrl+1..4`      | Move window to virtual desktop N              |
| `Meta+Tab`            | Activity switcher                             |
| `Meta+T`              | Tile manager                                  |
| `Meta+V`              | Clipboard history                             |
| `Meta+N`              | Create new note in Obsidian (see below)       |
| `Meta+Ctrl+N`         | Toggle Do Not Disturb                         |

The `Meta+N` → new Obsidian note requires an Obsidian-side URI handler:

```bash
# In Obsidian → Settings → Community plugins → enable "Advanced URI".
# Then in System Settings → Shortcuts → Custom Shortcuts → New → Command/URL:
# xdg-open "obsidian://new?vault=main&name=quick-note-$(date +%Y%m%d-%H%M%S)"
# Trigger: Meta+N
```

## 8.1.10 dotfiles — commit everything

After you've tweaked all of the above, commit your dotfiles to a private repo. Essential files to track:

- `~/.zshrc`
- `~/.config/starship.toml`
- `~/.config/kitty/kitty.conf` (if you use Kitty)
- `~/.config/wireplumber/`
- `~/.config/pipewire/pipewire.conf`
- `~/.config/containers/`
- `~/.config/continue/config.json`
- `~/.config/kwinrulesrc`
- `~/.config/kglobalshortcutsrc`
- `~/.config/plasma-org.kde.plasma.desktop-appletsrc`
- `~/.config/krunnerrc`
- `/etc/keyd/default.conf` (needs sudo — copy to `~/dotfiles/system/etc/keyd/default.conf`)
- `/etc/systemd/journald.conf`
- `/etc/sysctl.d/99-local-tuning.conf`
- `/etc/default/grub`

### Using `chezmoi` for portable dotfiles

```bash
# Install (apt has it; or uv tool install chezmoi):
sudo apt install -y chezmoi

# Init:
chezmoi init --apply $GITHUB_USER/dotfiles
```

chezmoi handles OS-specific conditionals, secrets (via gpg or age encryption), and templates (so your ssh key's comment is per-machine).

Alternative: `stow`, `yadm`, raw git with a bare repo. Pick one; chezmoi is the most powerful.

## 8.1.11 Snapshot

```bash
sudo timeshift --create --comments "post-keyboard-automation $(date -I)" --tags D
```

Proceed to [8.2 Backup strategy — Restic + 3-2-1](02-backup-strategy-3-2-1.md).

---

[Home](../README.md) · [↑ 08 Productivity & Security](README.md) · [← Previous: 08 (section)](README.md) · **8.1 Keyboard & automation** · [Next: 8.2 Backup →](02-backup-strategy-3-2-1.md)
