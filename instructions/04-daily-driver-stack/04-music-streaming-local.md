[Home](../README.md) · [↑ 04 Daily Driver](README.md) · [← Previous: 4.3 Bluetooth headset](03-bluetooth-headset-hifi.md) · **4.4 Music** · [Next: 4.5 Comms & productivity →](05-comms-and-productivity.md)

---

# 4.4 Music — Streaming and Local Libraries

## The stack

| Role                        | Pick                     | Channel  | Why                                                                 |
| --------------------------- | ------------------------ | -------- | ------------------------------------------------------------------- |
| Primary streaming           | **Spotify**              | Flatpak  | The client people actually use; Flatpak is the maintained path      |
| Primary local library       | **Strawberry**           | APT      | Clementine fork; modern Qt, rock-solid                              |
| Plasma-native alternative   | **Elisa**                | APT      | KDE-native, minimal, great system tray and MPRIS integration        |
| Terminal player             | **ncspot** (Spotify) or **ncmpcpp** (local MPD) | APT/cargo | Lightweight, keyboard-driven          |
| YouTube / podcast / ripping | **yt-dlp** (+ `mpv` for playback) | APT | Scripted ripping and mirror-playback                           |
| High-fidelity local ripping | **Sound Juicer** OR **asunder** | APT | Rip CDs to FLAC                                                 |

## 4.4.1 Spotify via Flatpak

```bash
flatpak install -y flathub com.spotify.Client
```

Launch via KRunner → "Spotify" or from the Plasma menu. Log in.

### Why Flatpak and not the snap, the .deb, or the Docker image

- **Snap** — default on Ubuntu-flavour, but not installed on Kubuntu 26.04 Minimal and explicitly removed in [2.1](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md). Known for slow cold-start and ugly theming.
- **`.deb` from Spotify's repo** — works, but Spotify ships `.deb` infrequently; you'll lag behind the Windows client by 3-6 months on features.
- **Flatpak** — Spotify publishes to Flathub directly; always current. Best theming of the three.

### Spotify and LDAC — interaction verification

After [4.3](03-bluetooth-headset-hifi.md), playing Spotify to your paired headset should use LDAC. Verify while music is playing:

```bash
pactl list sinks | grep "api.bluez5.codec"
# Expect: "ldac" (or your headset's highest-supported)
```

If Spotify is using SBC while other apps use LDAC, it's a WirePlumber routing question; restart wireplumber or log out/in.

### Spotify Connect

Spotify Desktop acts as a Connect device. From your phone, you can pick "this computer" as playback target. Useful for curating a queue from the phone while the laptop's speakers/headset play. On by default; no config needed.

## 4.4.2 Strawberry for local libraries

```bash
sudo apt install -y strawberry
```

Launch. Point at `~/Music` (or wherever your FLAC/MP3 library lives). Let it scan.

Features you will use:

- **Library organiser** — File → Organise Files → pattern `%albumartist%/%year% - %album%/%track:.2% - %title%.%extension%`. Standardises file paths.
- **Tag editor** — ID3v2.4 for MP3, Vorbis for FLAC, M4A for AAC. Bulk-rename via "Edit Tags → Apply Pattern".
- **Transcoder** — FLAC → MP3 320 for a phone with limited storage, etc.
- **Last.fm scrobbling** — if you scrobble. Toggleable per-device.
- **MusicBrainz** integration — CoverArt + metadata lookup.

Configure output to PipeWire (default), set crossfade on / off as you like, disable all the `rain / fire / storm` ambiance effects it ships with.

## 4.4.3 Elisa for Plasma-native minimalism

```bash
sudo apt install -y elisa
```

Elisa is simpler than Strawberry — no library tagger, no transcoder, just a clean "Play the library" experience with perfect system tray / MPRIS / task-switcher integration. Pick Elisa OR Strawberry as your local player; having both is pointless.

Elisa is my daily pick for local listening because:

- System tray control hooks (right-click Elisa icon → play/pause/next/prev) work via MPRIS2.
- Plasma lock-screen shows the current track and controls.
- No settings to configure; just works.

## 4.4.4 ncspot — Spotify in a terminal

If you live in Konsole and want Spotify without the Electron overhead:

```bash
sudo apt install -y ncspot
ncspot
```

Spotify Premium required (Connect device only authenticates Premium accounts). Keyboard shortcuts:

- `s` — search
- `q` — queue view
- `p` — playlists
- `Enter` — play selected
- `Space` — pause/resume
- `n` — next, `N` — previous
- `Shift+Left/Right` — seek
- `Q` — quit

Connects via `librespot-rs`; uses your Spotify account as a Connect endpoint.

## 4.4.5 MPD + ncmpcpp (the UNIX-y local-library stack)

If you want a headless music daemon that any client can control (so the mobile-to-desktop queue interaction works without Spotify), set up MPD.

```bash
sudo apt install -y mpd mpc ncmpcpp

# Configure MPD for PipeWire output:
sudo tee -a /etc/mpd.conf >/dev/null <<'EOF'

audio_output {
    type        "pipewire"
    name        "PipeWire"
    mixer_type  "software"
}

music_directory    "/home/YOURUSERNAME/Music"
playlist_directory "/home/YOURUSERNAME/.config/mpd/playlists"
db_file            "/home/YOURUSERNAME/.config/mpd/mpd.db"
log_file           "/home/YOURUSERNAME/.config/mpd/mpd.log"
pid_file           "/home/YOURUSERNAME/.config/mpd/mpd.pid"
state_file         "/home/YOURUSERNAME/.config/mpd/state"
sticker_file       "/home/YOURUSERNAME/.config/mpd/sticker.sql"
EOF

# Or user-level MPD (cleaner):
mkdir -p ~/.config/mpd/playlists
cp /etc/mpd.conf ~/.config/mpd/mpd.conf
# Edit to match user paths, then:
systemctl --user enable --now mpd.service
```

Use `ncmpcpp` as the TUI client, or M.A.L.P / Cantata as Android clients that control MPD over the LAN.

## 4.4.6 yt-dlp for ripping / podcast download

```bash
sudo apt install -y yt-dlp
```

Common recipes:

```bash
# Download a YouTube video's audio as the best-available audio-only stream:
yt-dlp -x --audio-format opus -o '~/Music/yt/%(uploader)s - %(title)s.%(ext)s' 'https://www.youtube.com/watch?v=...'

# Download a whole podcast RSS feed as mp3:
yt-dlp -x --audio-format mp3 --yes-playlist --download-archive ~/Podcasts/.archive 'https://podcast.rss.feed.url'

# Play a YouTube URL directly in mpv without downloading:
mpv 'https://www.youtube.com/watch?v=...'
# (mpv detects the URL and invokes yt-dlp internally to get the stream URL)
```

Update `yt-dlp` monthly — the repos lag behind YouTube's frontend changes. Either `sudo apt update && sudo apt upgrade yt-dlp` (Ubuntu's packaging cadence is OK), or keep a per-user install via `uv tool install yt-dlp`:

```bash
# After installing uv in [5.1]:
uv tool install yt-dlp
# Now ~/.local/bin/yt-dlp is the newest version.
```

## 4.4.7 Ripping CDs to FLAC

If you still have a CD library:

```bash
sudo apt install -y asunder flac lame
# asunder GUI: File → Select CD → Rip. Flac + MP3 simultaneously, MusicBrainz lookup, gapless rip.
```

Output goes to `~/Music/Artist/Album/Track.flac` by default. Edit `asunder` preferences → output template to match Strawberry's organiser.

## 4.4.8 Lossless audio over Bluetooth — the real story

Even with LDAC 990, Bluetooth compresses. Nothing compresses as little as wired 3.5 mm or USB-C to a decent DAC. If you care about actual hi-fi music listening:

- Use wired headphones or a wired DAC + amp.
- Use USB Audio Class 2.0 gear (Topping, Fiio, Schiit) for bit-perfect 24/96 or 24/192.
- Bluetooth is for mobility and convenience, not for audiophile listening.

For most listeners with a $100-400 BT headset, LDAC 990 on your TUF A17 via PipeWire is indistinguishable from wired. Below $100 it's the headset that's the limit, not the codec.

## 4.4.9 MPRIS integration

All major players (Spotify, Strawberry, Elisa, mpv, VLC, ncspot, MPD via `mpDris2`) expose the MPRIS2 D-Bus interface. Plasma's media controls (system tray media-control widget, lock screen media-controls, global shortcuts `Meta+Play/Pause/Next/Prev`) all talk to MPRIS2 — so once any supported player is playing, pressing the hardware media keys just works.

Verify:

```bash
playerctl -l
# Lists currently-running MPRIS players.

playerctl play-pause
playerctl next
```

Add global shortcuts in System Settings → **Shortcuts** → **Global Shortcuts** → **Media Controls**. Map to Meta+P, Meta+[, Meta+].

## 4.4.10 Snapshot

```bash
sudo timeshift --create --comments "post-music $(date -I)" --tags D
```

Proceed to [4.5 Comms and productivity apps](05-comms-and-productivity.md).

---

[Home](../README.md) · [↑ 04 Daily Driver](README.md) · [← Previous: 4.3 Bluetooth headset](03-bluetooth-headset-hifi.md) · **4.4 Music** · [Next: 4.5 Comms & productivity →](05-comms-and-productivity.md)
