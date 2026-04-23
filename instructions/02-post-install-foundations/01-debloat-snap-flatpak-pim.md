[Home](../README.md) · [↑ 02 Post-Install](README.md) · [← Previous: 02 Post-Install (section)](README.md) · **2.1 Debloat** · [Next: 2.2 NVIDIA & CUDA →](02-nvidia-driver-580-and-cuda.md)

---

# 2.1 Debloat: Snap Removal, Flatpak + Flathub, Firefox APT, KDE PIM Prune

The "make this system actually yours" step. Four sub-steps, each independent:

1. **Remove snap** entirely (daemon, mounts, `snap-store`).
2. **Install Flatpak + enable Flathub** as the universal GUI-app channel.
3. **Ensure Firefox comes from Mozilla's APT repo** (not snap, not Flatpak).
4. **Prune KDE PIM (Akonadi, KMail, Kontact)** if the Minimal install pulled any in.

## Snapshot first

```bash
sudo timeshift --create --comments "before-debloat $(date -I)" --tags D
```

If anything here goes sideways, `sudo timeshift --list` → pick this one → restore.

## 2.1.1 Remove snap

On Kubuntu 26.04 Minimal, snapd is still preinstalled as a dependency of a few Ubuntu meta-packages even though no snap apps are installed by default. It starts a daemon, mounts a `/snap` FUSE filesystem, and hooks `/etc/environment` to prepend `/snap/bin` to PATH. Removing it cleanly takes a few apt commands.

```bash
# Stop and disable the snap services first:
sudo systemctl disable --now snapd.service snapd.socket snapd.seeded.service
sudo systemctl mask snapd.service snapd.socket snapd.seeded.service 2>/dev/null || true

# Remove any installed snaps (should be none on Minimal, but defensive):
snap list 2>/dev/null | awk 'NR>1 {print $1}' | xargs -r sudo snap remove --purge

# Remove snap metapackages:
sudo apt purge -y snapd snap-confine

# Remove the snap data directory and mount leftovers:
sudo rm -rf /var/cache/snapd
sudo rm -rf /snap
rm -rf ~/snap

# Prevent snap from being pulled back in by a dependency upgrade:
sudo tee /etc/apt/preferences.d/nosnap.pref >/dev/null <<'EOF'
Package: snapd
Pin: release a=*
Pin-Priority: -10
EOF

sudo apt update
```

Verify:

```bash
snap list 2>&1 | head -1
# Expect: command not found: snap

systemctl status snapd 2>&1 | head -1
# Expect: Unit snapd.service could not be found (good — masked).

ls /snap 2>&1
# Expect: No such file or directory.
```

The `nosnap.pref` file prevents future `apt install some-package` from silently pulling in snapd as a recommended dependency. The pin priority of -10 marks snapd as "do not install under any circumstances." If a package genuinely requires snapd (rare; almost always means a Flatpak or apt alternative exists), apt will error out and you can decide then.

## 2.1.2 Install and enable Flatpak + Flathub

Flatpak is Kubuntu's officially-endorsed alternative to snap. It has better Qt integration (fewer visual quirks than snap Qt apps), and Flathub hosts the majority of third-party GUI apps you'll want.

```bash
sudo apt install -y flatpak plasma-discover-backend-flatpak kde-config-flatpak

# Add the Flathub remote (system-wide so all users get it):
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Also add the Flathub "beta" remote as a rarely-used fallback for preview builds:
# (uncomment if you want it)
# sudo flatpak remote-add --if-not-exists flathub-beta https://flathub.org/beta-repo/flathub-beta.flatpakrepo
```

Test it installs a simple package (we'll uninstall immediately):

```bash
flatpak install -y flathub org.gnome.dictionary
flatpak run org.gnome.dictionary &
# Verify a window appears. Close it.
flatpak uninstall -y org.gnome.dictionary
flatpak uninstall -y --unused
```

Verify Discover sees Flathub:

- Open Discover (Plasma's app store GUI).
- Go to Settings (top-right) → you should see Flatpak → Flathub listed as a source.
- You should **not** see any snap backend — `plasma-discover-backend-snap` was removed with snapd.

### Flatpak theming fix (Qt apps look ugly otherwise)

Flatpak apps live in sandboxes and don't automatically inherit your system's Qt theme. For Qt apps (most Flatpak apps you'll install here — Discord is Electron so skip this, but Telegram Desktop, Zen Browser, and KDE apps are Qt):

```bash
sudo flatpak override --filesystem=xdg-config/kdeglobals:ro
sudo flatpak override --filesystem=xdg-config/gtk-3.0:ro
sudo flatpak override --filesystem=xdg-config/gtk-4.0:ro

# Install the KDE Breeze theme as a Flatpak runtime extension so all Qt apps pick it up:
flatpak install -y flathub org.kde.KStyle.Breeze//6.6
flatpak install -y flathub org.kde.WaylandDecoration.QGnomePlatform//6.6 || true
```

After this, Qt-based Flatpak apps use Breeze widgets and the colour scheme matches Plasma.

## 2.1.3 Firefox via Mozilla's APT repository

Kubuntu 26.04 Minimal includes Firefox from the **Mozilla team APT repo** (not snap — Canonical's snap-only Firefox was an Ubuntu-GNOME thing; Kubuntu correctly routes around it). Confirm this is the case, and if for some reason you have the snap or the Flatpak version, remove those.

Verify:

```bash
apt policy firefox
# Expect first line: "firefox:" then lines showing 500 priority from Mozilla.
# The repo URL should contain "packages.mozilla.org/apt" or (on 26.04) the new "deb.mozilla.org" URL.
# If the top entry is from `Installed: 1:1snap1-0ubuntu...` that's the snap-wrapper. Remove it:
# sudo apt purge firefox  # only if this was a snap wrapper
```

If Firefox is missing entirely or from the wrong source, re-add Mozilla's repo:

```bash
# Key (download and trust Mozilla's APT repo key):
wget -q -O - https://packages.mozilla.org/apt/repo-signing-key.gpg | \
  sudo tee /etc/apt/keyrings/packages.mozilla.org.asc > /dev/null

# Verify the fingerprint — MUST match what Mozilla publishes:
gpg --show-keys /etc/apt/keyrings/packages.mozilla.org.asc
# Expected fingerprint: 35BAA0B33E9EB396F59CA838C0BA5CE6DC6315A3

# Add the repo:
echo "deb [signed-by=/etc/apt/keyrings/packages.mozilla.org.asc] https://packages.mozilla.org/apt mozilla main" | \
  sudo tee /etc/apt/sources.list.d/mozilla.list > /dev/null

# Pin Mozilla's Firefox higher than Ubuntu's (if Ubuntu's archive carries one):
sudo tee /etc/apt/preferences.d/mozilla >/dev/null <<'EOF'
Package: *
Pin: origin packages.mozilla.org
Pin-Priority: 1000
EOF

sudo apt update
sudo apt install -y firefox
```

Launch Firefox. Version should be 148+ (matching the 26.04 release notes). URL bar: `about:support`, scroll to "Application Basics" — "Update Channel" should say `release` and "Application Binary" should be `/usr/lib/firefox/firefox`. That confirms APT install, not snap.

## 2.1.4 Prune KDE PIM if present

The Minimal install should NOT include Kontact, KMail, Korganizer, Akregator, KAddressbook. If for some reason they are there (or you picked Normal and changed your mind):

```bash
sudo apt purge -y \
  kontact kmail korganizer kaddressbook akregator \
  kontactinterface5 akonadi-server akonadi-mime akonadi-contacts \
  akonadi-calendar akonadi-notes libkf5akonadi* \
  libkf5kmime* libkf5messagelib*

sudo apt autoremove -y --purge
```

The Akonadi server is the painful one to leave running — it indexes your home, runs several child processes, and is a leading source of memory leaks on the KDE bug tracker. Removing it is free performance.

## 2.1.5 Other minor debloat items

These are low-priority but take 30 seconds to do and add up.

```bash
# Popcon (Debian Popularity Contest — sends anonymised package usage to Debian servers):
sudo apt purge -y popularity-contest 2>/dev/null || true

# Ubuntu Advantage tools (the Pro-subscription client; useful but noisy on first boot):
sudo apt purge -y ubuntu-advantage-tools 2>/dev/null || true
# Note: if you plan to use Ubuntu Pro (free for 5 personal machines), skip this and instead
# silence the MOTD/login nags with:
# sudo pro config set apt_news=false
# sudo pro config set motd_esm=false  # only if esm is already disabled

# Whoopsie (Ubuntu crash reporter):
sudo systemctl disable --now whoopsie.service 2>/dev/null || true
sudo apt purge -y whoopsie 2>/dev/null || true

# Apport (crash-collection daemon):
sudo systemctl disable --now apport.service 2>/dev/null || true

# Remove "command-not-found" hook from bash (speeds up shell init by ~0.2s):
sudo apt purge -y command-not-found 2>/dev/null || true
```

The Whoopsie / Apport removals are also in [3.3](../03-speed-and-boot-optimizations/03-service-masking-and-journald.md) with more detail. Doing them here is fine; the two pages intersect deliberately (belt + suspenders).

## 2.1.6 Result

```bash
# Verify clean state:
dpkg -l | grep -E 'snap|apport|whoopsie|kmail|akonadi' || echo "clean"
# Expect: clean

# Verify Flatpak + Flathub:
flatpak remotes
# Expect: flathub  https://flathub.org/repo/

# Verify Firefox:
apt policy firefox | head -4
# Expect: Installed: 148.x.x from packages.mozilla.org

# Disk space recovered:
df -h /
# (informational — small at this point)
```

Take a Timeshift snapshot for the "post-debloat" state:

```bash
sudo timeshift --create --comments "post-debloat $(date -I)" --tags D
```

Proceed to [2.2 NVIDIA driver 580 and CUDA](02-nvidia-driver-580-and-cuda.md).

---

[Home](../README.md) · [↑ 02 Post-Install](README.md) · [← Previous: 02 Post-Install (section)](README.md) · **2.1 Debloat** · [Next: 2.2 NVIDIA & CUDA →](02-nvidia-driver-580-and-cuda.md)
