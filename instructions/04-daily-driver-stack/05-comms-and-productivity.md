[Home](../README.md) · [↑ 04 Daily Driver](README.md) · [← Previous: 4.4 Music](04-music-streaming-local.md) · **4.5 Comms & productivity** · [Next: 4.6 Plasma UX →](06-plasma-66-ux-tiling-activities.md)

---

# 4.5 Communication and Productivity Apps

Chat, video calls, email, notes, and the password manager. All via Flatpak unless noted otherwise, because Flathub is where the active maintainers of these apps publish.

## The stack

| Role                     | App                      | Channel                           | Rationale                                  |
| ------------------------ | ------------------------ | --------------------------------- | ------------------------------------------ |
| 1:1 secure messaging     | **Signal Desktop**       | Flatpak (`org.signal.Signal`)     | Native Electron; best official build       |
| Group chat + channels    | **Telegram Desktop**     | Flatpak (`org.telegram.desktop`)  | Native Qt, best Linux client               |
| Voice / gaming           | **Discord**              | Flatpak (`com.discordapp.Discord`)| Same as above                              |
| Video calls (generic)    | **Zoom**                 | Flatpak (`us.zoom.Zoom`) or `.deb` | Zoom-the-protocol; `.deb` if you need corporate profiles |
| Video calls (WebRTC)     | **Browser (Firefox / Brave)** | see [4.1](01-browsers.md)    | Meet, Jitsi, BBB, WhereBy — all work in-browser |
| Email                    | **Thunderbird**          | APT (`thunderbird`) — Mozilla team | Or KMail via APT if you prefer KDE-native |
| Email CLI                | **aerc** or **mutt**     | APT                               | For script-driven email / mailing-list triage |
| Calendar                 | **Thunderbird + Lightning** | bundled                         | Or **Kalendar** if KDE-native              |
| Password manager         | **Bitwarden Desktop**    | Flatpak (`com.bitwarden.desktop`) | CLI via `bitwarden-cli` from apt           |
| 2FA (TOTP) offline       | **KeePassXC**            | APT (`keepassxc`)                 | Native Qt; best KDE integration            |
| Notes                    | **Obsidian**             | Flatpak (`md.obsidian.Obsidian`)  | Markdown + graph + plugins                 |
| Notes alt / research     | **Joplin**               | Flatpak (`net.cozic.joplin_desktop`) | E2EE sync, markdown, better for long-form |
| Note-taking for code     | **Zed / VS Code + Markdown** | apt / .deb                    | If Obsidian is too heavyweight             |
| Office                   | **LibreOffice**          | APT                               | Excellent native Linux office              |
| Screenshots              | **Spectacle** (built-in) | —                                 | Plus **Flameshot** (Flatpak) for annotations |
| Screen recording         | **OBS Studio**           | Flatpak                           | Covered in [4.2.8](02-video-audio-pipewire.md#428-obs-studio-screencast--recording) |

## Snapshot first

```bash
sudo timeshift --create --comments "before-comms-and-productivity $(date -I)" --tags D
```

## 4.5.1 Install the Flatpak set (one command)

```bash
flatpak install -y flathub \
  org.signal.Signal \
  org.telegram.desktop \
  com.discordapp.Discord \
  us.zoom.Zoom \
  com.bitwarden.desktop \
  md.obsidian.Obsidian \
  net.cozic.joplin_desktop \
  org.flameshot.Flameshot
```

## 4.5.2 Install the APT set

```bash
sudo apt install -y \
  thunderbird \
  keepassxc \
  libreoffice \
  libreoffice-style-breeze \
  libreoffice-kf6
```

The last two are the KDE-style icon set and KF6 plugin respectively — make LibreOffice look native in Plasma 6.6 instead of vaguely alien.

## 4.5.3 Per-app configuration notes

### Signal Desktop

- Link to your phone (Settings → Linked devices → Pair). First sync takes 1-2 minutes on a large history.
- Incoming notifications on Plasma 6.6 Wayland: use the KDE notification portal. If notifications don't appear, in Signal Settings → Notifications → toggle "Enable notifications" off/on once to re-register.

### Telegram Desktop

- Log in with SMS code or existing session. Desktop can't initiate Secret Chats (mobile-only), but otherwise full-featured.
- Two useful settings: **Settings → Chat Settings → Disable "Allow Animated Interactive Emoji"** (CPU hog on long chats) and **Settings → Advanced → Experimental → "Use a separate window for sticker/emoji picker"**.

### Discord

- The Flatpak's notification sound loves to bypass PipeWire default sink mixing. In Discord → User Settings → Voice & Video → Output Device → set explicitly (don't leave on "Default"), or Sound Effects → disable.
- For screen sharing on Wayland, Discord uses PipeWire via xdg-desktop-portal-kde (auto-configured from [2.4.4](../02-post-install-foundations/04-wayland-plasma-66-nvidia.md#244-screen-sharing-via-pipewire-portal)).
- Push-to-talk key binding works on Wayland in 26.04 (it didn't pre-Plasma 6.3). Set via Discord → Voice Settings → Key Binds.

### Zoom

Flatpak version works for 90 % of uses. If your employer requires the corporate-signed `.deb` (for SAML/SSO policies):

```bash
# Download the latest .deb from https://zoom.us/download?os=linux
# Then:
sudo apt install -y ./zoom_amd64.deb
```

.deb Zoom installs to `/opt/zoom` and puts shims in `/usr/bin/zoom`. Peacefully coexists with the Flatpak if you install both; pick one as default via `xdg-mime default`.

### Bitwarden Desktop

- Log in with your master password.
- Enable **TouchID-style unlock** — Settings → Security → Unlock with PIN / biometrics. TUF A17 doesn't have a fingerprint reader but you can set a short PIN to unlock within-session.
- Integrate with browser: the Bitwarden browser extension talks to the desktop via native messaging.

### KeePassXC

For offline-only 2FA codes and anything you don't want synced to a cloud:

- File → Database → New database.
- Save the `.kdbx` to `~/Documents/` — it's a single encrypted file; backed up via [8.2 Restic](../08-productivity-security/02-backup-strategy-3-2-1.md).
- Add entries: Name, Username, Password, URL, Notes. For TOTP: right-click entry → "TOTP → Setup TOTP" → paste the base32 secret.
- System tray icon → unlocks database with single password, quick-copy of TOTP codes.
- Browser extension: KeePassXC-Browser — install in Firefox, connects via a local socket to autofill.

### Obsidian

- Pick a vault location — I use `~/Documents/obsidian/main/`.
- Git plugin: Settings → Community plugins → "Obsidian Git" — enables auto-commit + push to a private GitHub repo as backup.
- Syncthing (alternative to Obsidian Sync): free, P2P. Install `syncthing` via apt, sync the vault to phone via the Android Syncthing app.
- Key plugins I use: **Dataview**, **Templater**, **Excalidraw**, **Tasks**, **Periodic Notes**.

### Joplin vs Obsidian

Pick one or both based on workflow:

- **Obsidian**: graph view, plugins, block-references, better for knowledge-base / Zettelkasten.
- **Joplin**: E2EE sync built-in, better clipper for web-article capture, simpler day-to-day notes.

Using both is fine — they don't conflict; use Obsidian for structured knowledge, Joplin for ephemeral clippings.

### Thunderbird

```bash
# Mozilla-signed Thunderbird from their APT repo (same setup as Firefox in 2.1.3):
# The Mozilla apt repo also carries Thunderbird 148.

sudo apt install -y thunderbird
```

Set up accounts via "Existing email account" wizard. For Gmail, use OAuth2 (default). For self-hosted, Fastmail, Proton Mail Bridge — IMAP + SMTP with appropriate authentication.

Add the Lightning calendar plugin via Thunderbird → Add-ons.

### Flameshot (screenshots with annotations)

Spectacle ships with Plasma and is perfectly fine for "take a screenshot". Flameshot is better when you need to annotate with arrows/boxes/text on the screenshot:

```bash
flatpak install -y flathub org.flameshot.Flameshot
```

Bind a shortcut (System Settings → Shortcuts → Custom Shortcuts → New → `flatpak run org.flameshot.Flameshot gui`) to `PrintScreen` or `Meta+Shift+S`.

Note: on Wayland + NVIDIA, Flameshot uses the `xdg-desktop-portal-kde` screenshot portal. Approve on first invocation.

### LibreOffice

Out of the box on 26.04: LibreOffice 26.2.2. Theme: Tools → Options → Personalization → "Choose preinstalled theme" → pick something other than the default (Colibre is good).

Install fonts for interop:

```bash
sudo apt install -y fonts-crosextra-carlito fonts-crosextra-caladea
# Carlito → Calibri substitute
# Caladea → Cambria substitute
```

## 4.5.4 System tray clutter — hide what you don't need

With all of the above installed, the system tray fills up: Signal, Telegram, Discord, Bitwarden, KeePassXC, Elisa, Blueman, etc. Hide rarely-used ones:

System Settings → Quick Settings → "System Tray" module → uncheck visibility for apps you want in "Status & Notifications" collapsible sub-panel rather than always visible.

Recommended always-visible: Battery, Network, Bluetooth, Audio, Clipboard. Everything else goes to collapsed.

## 4.5.5 Notifications policy

Plasma 6.6 has a powerful notification filter UI. Configure:

System Settings → **Notifications** → **Application Settings** → per-app:

- **Do not interrupt** for Signal / Telegram / Discord group chats you don't need live.
- **Allow critical notifications** for Thunderbird (email).
- **Show in Do Not Disturb mode** — allow only KeePassXC (security) and the system.
- Set a default Do Not Disturb hotkey (Meta+Ctrl+N).

## 4.5.6 Snapshot

```bash
sudo timeshift --create --comments "post-comms-and-productivity $(date -I)" --tags D
```

Proceed to [4.6 Plasma 6.6 UX — tiling, Activities, Konsole](06-plasma-66-ux-tiling-activities.md).

---

[Home](../README.md) · [↑ 04 Daily Driver](README.md) · [← Previous: 4.4 Music](04-music-streaming-local.md) · **4.5 Comms & productivity** · [Next: 4.6 Plasma UX →](06-plasma-66-ux-tiling-activities.md)
