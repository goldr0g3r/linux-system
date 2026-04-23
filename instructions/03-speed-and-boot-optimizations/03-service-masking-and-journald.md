[Home](../README.md) · [↑ 03 Speed & Boot](README.md) · [← Previous: 3.2 zram & sysctl](02-zram-swap-sysctl.md) · **3.3 Service masking & journald** · [Next: 3.4 Filesystem & thermals →](04-filesystem-thermals-power.md)

---

# 3.3 Service Masking and journald Capping

Kubuntu 26.04 ships with Ubuntu's historical accretion of crash-report, error-report, telemetry-adjacent, and "maybe you'll need this" services enabled by default. On a personal dev laptop, most of them are pure overhead. This page disables them safely.

## The principle

`systemd` has three service states:

- **Enabled** — starts at boot.
- **Disabled** — does not start at boot; can still be started manually (`systemctl start`) or pulled in by a dependency.
- **Masked** — symlinked to `/dev/null`; cannot be started by any mechanism. This is the "permanent off" state.

For services we truly want gone, we `mask`. For services that might legitimately be pulled in later (e.g. by a user choosing to run a specific app), we `disable`.

## Snapshot first

```bash
sudo timeshift --create --comments "before-service-masking $(date -I)" --tags D
```

## 3.3.1 Baseline: what's running now

```bash
# What's running on a fresh 26.04 install, sorted by boot-time impact:
systemd-analyze blame | head -30

# Failed units (should be none):
systemctl --failed

# Enabled services:
systemctl list-unit-files --state=enabled --type=service | wc -l
# On Minimal Kubuntu 26.04 this is typically 45-55.
```

Save the baseline for comparison:

```bash
systemctl list-unit-files --state=enabled > ~/baseline-services-$(date -I).txt
systemd-analyze blame > ~/baseline-blame-$(date -I).txt
```

## 3.3.2 Mask the safely-removable services

These are all removable on a personal dev laptop with no impact on daily work.

```bash
# ApportApport — Ubuntu's crash-report daemon. It uploads crashes to Canonical
# with minimal user benefit for a personal machine. Also popular as a malware
# vector pre-2020 (pwnkit-style local privesc bugs).
sudo systemctl disable --now apport.service 2>/dev/null || true
sudo systemctl mask apport.service
sudo apt purge -y apport apport-symptoms 2>/dev/null || true

# Whoopsie — sends aggregated crash metadata to Canonical. See Apport.
sudo systemctl disable --now whoopsie.service 2>/dev/null || true
sudo systemctl mask whoopsie.service

# ModemManager — matters for 3G/4G/LTE modems. Your TUF A17 doesn't have one.
# If you ever plug a USB LTE dongle in, unmask it then.
sudo systemctl disable --now ModemManager.service 2>/dev/null || true
sudo systemctl mask ModemManager.service

# Ubuntu Advantage tools — the Pro subscription client. Useful if you're
# activating Pro (free tier for 5 personal machines); noisy if you're not.
# Alternative: keep installed but silence the motd nags:
#   sudo pro config set apt_news=false
# If you want it gone:
sudo systemctl disable --now ua-timer.timer 2>/dev/null || true
sudo apt purge -y ubuntu-advantage-tools 2>/dev/null || true

# Livepatch — kernel live-patching; requires Pro for livepatches.
# Disable unless using Pro:
sudo systemctl disable --now canonical-livepatch.service 2>/dev/null || true

# MOTD news — daily "wisdom" messages on SSH login (not visible on GUI, but active).
sudo systemctl disable --now motd-news.service motd-news.timer 2>/dev/null || true

# Popularity contest (Debian Popcon) — weekly report of installed packages. Useless
# for a personal machine; already purged in 2.1 but double-check:
sudo apt purge -y popularity-contest 2>/dev/null || true

# cups-browsed — network printer discovery. Useful on a LAN with shared CUPS
# printers; useless if you're at home and print over USB or IPP directly.
# Comment out if you have a LAN printer:
sudo systemctl disable --now cups-browsed.service 2>/dev/null || true

# Bluetooth obex — OBEX file transfer over Bluetooth (sending a file from phone).
# Not used daily. Unmask if you need it.
sudo systemctl disable --now bluetooth-obex.service 2>/dev/null || true
```

### Things NOT to mask

These are commonly seen in "debloat" guides online but have real reasons to keep:

- `NetworkManager.service` — Wi-Fi. Don't touch.
- `ssh.service` — is not even installed by default; if you install `ssh` server later, you want it.
- `systemd-journald.service` — logging. Essential. We just cap its disk use in 3.3.3.
- `systemd-resolved.service` — DNS. Essential, especially for Podman's network namespace.
- `bluetooth.service` — Bluetooth itself. You want this for the headset.
- `packagekit.service` — Discover's backend. Disable only if you don't use Discover.
- `upower.service` — battery status readable by everything. Essential.
- `thermald.service` — is NOT installed on AMD by default; it's Intel-specific. No action needed.
- `fwupd.service` — firmware update daemon. Keep; your TUF A17 gets BIOS updates via LVFS.
- `smartmontools/smartd.service` — SMART disk health. Useful; keep.

## 3.3.3 Cap the journald disk footprint

systemd-journald's default behaviour: use up to 10 % of the filesystem size for logs. On a 512 GB SSD, that's 51 GB of logs. Way more than needed.

Edit `/etc/systemd/journald.conf`:

```bash
sudoedit /etc/systemd/journald.conf
```

Set / ensure these lines (uncomment existing ones, add missing):

```ini
[Journal]
Storage=persistent
Compress=yes
SystemMaxUse=400M
SystemMaxFileSize=50M
SystemKeepFree=1G
RuntimeMaxUse=100M
ForwardToSyslog=no
ForwardToWall=no
MaxLevelStore=info
MaxRetentionSec=1month
```

Explanation:

- **`SystemMaxUse=400M`** — journal uses at most 400 MB on disk. Sufficient for ~1 month of history on a normal-activity laptop.
- **`SystemMaxFileSize=50M`** — each individual journal file tops out at 50 MB. Makes `journalctl --vacuum-time=1week` faster.
- **`SystemKeepFree=1G`** — don't grow the journal when free space is under 1 GB.
- **`MaxLevelStore=info`** — drop debug-level messages from persistent storage (still visible in live `journalctl -f`). Saves a lot.
- **`MaxRetentionSec=1month`** — automatically drop anything older than 1 month.
- **`ForwardToSyslog=no`** — no syslog listener on this machine; don't bother.

Apply:

```bash
sudo systemctl restart systemd-journald
journalctl --disk-usage
# Expect: current journal size — should shrink to under 400 MB within a minute as it compacts.

# Vacuum now to reclaim space:
sudo journalctl --vacuum-size=400M
```

## 3.3.4 User-level services

Per-user systemd services also start on login. Check what your user has:

```bash
systemctl --user list-unit-files --state=enabled | head -30
```

Probably the Plasma / PipeWire / Flatpak / Podman stuff — leave those. Things you might want to kill:

```bash
# tracker-miner (file indexing) — if you use GNOME apps that rely on Tracker,
# keep. If you only use KDE apps, it's wasted cycles.
systemctl --user mask tracker-miner-fs-3.service 2>/dev/null || true
systemctl --user mask tracker-extract-3.service 2>/dev/null || true

# evolution-addressbook / evolution-calendar — background email/calendar sync
# agents. Disable if you don't use Evolution or KDE PIM. We purged KDE PIM
# in 2.1 but evolution-data-server may have been kept by GNOME Online Accounts.
systemctl --user mask evolution-addressbook-factory.service 2>/dev/null || true
systemctl --user mask evolution-calendar-factory.service 2>/dev/null || true
systemctl --user mask evolution-source-registry.service 2>/dev/null || true
```

## 3.3.5 `systemctl enable --user <name>` for things you DO want

Make sure these user services are enabled:

```bash
# Podman rootless:
systemctl --user enable --now podman.socket

# PipeWire chain (usually auto-enabled; verify):
systemctl --user enable --now pipewire.service pipewire-pulse.service wireplumber.service

# If any are "failed", check `systemctl --user status <name>` and fix before proceeding.
```

## 3.3.6 Linger for user services that must survive logout

By default, systemd user services stop when you log out. If you want rootless Podman containers, SSH-agent, local Ollama, or anything else to keep running when you log out:

```bash
sudo loginctl enable-linger $USER
```

Now your user services run from boot to shutdown, not just while you're logged in.

## 3.3.7 After — re-measure

```bash
systemd-analyze
# Compare to baseline-blame-<date>.txt
# Expect 1-3 s faster to graphical.target, depending on how many services you masked.

systemctl list-unit-files --state=enabled --type=service | wc -l
# Should be ~10-15 fewer than baseline.

journalctl --disk-usage
# Under 400 MB.

systemctl --failed
# Ideally 0 failed units. If something failed, fix before proceeding.
```

## 3.3.8 Reverting

Every `systemctl mask` is reversible with `systemctl unmask`. If something stops working ("my Bluetooth file transfer from my phone doesn't connect"), unmask the relevant service:

```bash
sudo systemctl unmask bluetooth-obex.service
sudo systemctl enable --now bluetooth-obex.service
```

No reboot needed.

## 3.3.9 Snapshot

```bash
sudo timeshift --create --comments "post-service-masking $(date -I)" --tags D
```

Proceed to [3.4 Filesystem, thermals, and power profiles](04-filesystem-thermals-power.md).

---

[Home](../README.md) · [↑ 03 Speed & Boot](README.md) · [← Previous: 3.2 zram & sysctl](02-zram-swap-sysctl.md) · **3.3 Service masking & journald** · [Next: 3.4 Filesystem & thermals →](04-filesystem-thermals-power.md)
