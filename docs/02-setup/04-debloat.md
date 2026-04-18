[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.3 Partitioning](03-install-partitioning.md) · **2.4 Debloat** · [Next: 2.5 Kernel/Boot →](05-kernel-boot.md)

---

# 2.4 First-Boot Debloat

After first boot, the default Kubuntu install carries a handful of packages and services you do not want: snapd, KDE PIM (Kontact + Akonadi + MariaDB), some Ubuntu telemetry. This page removes them, adds Flatpak + Flathub, and leaves you with a lean base for the rest of the setup.

## One block, top to bottom

```bash
# System is up to date.
sudo apt update && sudo apt full-upgrade -y
sudo apt autoremove --purge -y

# Nuke snap entirely (Kubuntu 24.04 ships snapd even though it uses nothing by default).
sudo systemctl disable --now snapd.service snapd.socket snapd.seeded.service 2>/dev/null || true
sudo apt purge -y snapd
sudo apt-mark hold snapd
sudo rm -rf /var/cache/snapd ~/snap /snap
# Prevent accidental reinstall via dependency:
sudo tee /etc/apt/preferences.d/no-snap.pref >/dev/null <<'EOF'
Package: snapd
Pin: release a=*
Pin-Priority: -10
EOF

# Flatpak + Flathub + Discover backend.
sudo apt install -y flatpak plasma-discover-backend-flatpak
sudo flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo

# Prune KDE PIM / Akonadi / games (you do not use Kontact; Akonadi runs a MariaDB just to cache email metadata).
sudo apt purge -y \
  akonadi-backend-mysql akonadi-server akregator kontact kmail2 korganizer kaddressbook \
  kmines kpatience ksudoku kmahjongg \
  kruler kcharselect \
  elisa

# Ubuntu telemetry leftovers that may exist.
sudo apt purge -y apport apport-gtk apport-symptoms whoopsie popularity-contest ubuntu-report 2>/dev/null || true

sudo apt autoremove --purge -y
```

## What each block does and why

### Upgrade and autoremove (lines 1–3)

Catches up on security updates published between when the ISO was built and now. `autoremove --purge` cleans leftover kernel headers / metapackages from the live installer.

### Snap removal (lines 5–13)

- `systemctl disable --now snapd*` stops the daemon and its socket activation so the purge is clean.
- `apt purge snapd` removes the package.
- `apt-mark hold snapd` prevents accidental reinstall via unattended upgrades.
- `rm -rf /var/cache/snapd ~/snap /snap` removes residual caches and the mountpoint stub.
- The `apt preferences.d/no-snap.pref` pin sets snapd priority to negative — even if a dependency tries to pull it back in, APT refuses.

**Why remove snap at all?** On Kubuntu the answer is "because you never use it and it wastes ~50 MB of resident memory for nothing". On Ubuntu the answer is much longer — see [Part 1 Telemetry section](../01-verdict.md#telemetry-snap-and-package-manager-bloat).

### Flatpak enable (lines 15–17)

Flatpak is how you install GUI apps that want sandboxing (e.g. Discord, Spotify, Signal, Obsidian, Zotero) without Snap's startup-latency problems. Flathub is the main repo. `plasma-discover-backend-flatpak` makes the KDE software centre show Flatpak results alongside APT.

### KDE PIM pruning (lines 19–24)

Kubuntu ships Kontact (KDE's Outlook equivalent) with Akonadi (a personal information database that runs its own MariaDB instance). You use Thunderbird, Mailspring, or the Gmail web app. Kontact is ~800 MB of package + a running MariaDB + an indexer daemon. Purge it.

The game packages (`kmines`, `kpatience`, `ksudoku`, `kmahjongg`) are the KDE equivalents of Windows' built-in games. Remove them.

`elisa` is KDE's music player — keep if you want, remove if you will use Spotify / Tidal / MPD.

### Ubuntu telemetry (lines 26–27)

`apport` and `whoopsie` are the crash reporter and its popup daemon. `ubuntu-report` fires one-time telemetry on first login. `popularity-contest` sends package-installation statistics (opt-in by default on Kubuntu, but the package is still installed). None of these do anything useful for you; remove them.

### Final autoremove (line 29)

Cleans up any orphan dependencies the purges created.

## Verification

```bash
# Confirm snap is gone:
command -v snap || echo "snap removed"
systemctl status snapd 2>&1 | head -n3
ls /snap 2>&1

# Confirm Flatpak + Flathub:
flatpak remotes
# Expect: flathub   system

# Confirm no Akonadi daemon is running:
pgrep -af akonadi || echo "no Akonadi"
pgrep -af mariadb || echo "no mariadb"
```

## What you did not remove (on purpose)

- `kwallet*` — KDE's password/credential store. Kept because many Qt apps use it transparently.
- `kdeconnect` — phone/laptop integration (SMS, clipboard, file transfer). Kept because it is genuinely useful; you can disable it in System Settings later.
- `baloo-kde` — file indexer. You can disable it via `balooctl disable` if it is chewing CPU, but do not purge it because some KDE apps soft-depend on it.
- `plasma-vault` — per-folder encryption. No cost to leave installed; potentially useful later.

Proceed to [2.5 Kernel & boot optimization](05-kernel-boot.md).

---

[Home](../../README.md) · [↑ Setup Guide](README.md) · [← Previous: 2.3 Partitioning](03-install-partitioning.md) · **2.4 Debloat** · [Next: 2.5 Kernel/Boot →](05-kernel-boot.md)
