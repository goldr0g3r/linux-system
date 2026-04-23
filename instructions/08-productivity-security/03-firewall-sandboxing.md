[Home](../README.md) · [↑ 08 Productivity & Security](README.md) · [← Previous: 8.2 Backup](02-backup-strategy-3-2-1.md) · **8.3 Firewall & sandboxing** · [Next: 09 Apps catalog →](../09-apps-catalog.md)

---

# 8.3 Firewall, Sandboxing, YubiKey, ClamAV

The 80/20 security for a personal dev laptop: UFW firewall, Firejail for browser sandboxing, optional YubiKey for SSH + 2FA, and ClamAV for scan-only anti-malware.

## The threat model

You are:

- A dev on a laptop that travels to cafes, airports, hotels.
- Holding personal data (browser creds, SSH keys, Bitwarden master key) — interesting to credential thieves.
- **Not** a target for nation-state APTs; don't boil the ocean chasing that threat model.

So you prioritise:

1. Stop random LAN scans from finding listening ports (UFW).
2. Make browser-borne compromise limited in blast radius (Firejail, keep browser up to date).
3. Protect SSH / GitHub with hardware (YubiKey optional).
4. Periodic sanity scans for known malware signatures (ClamAV on-demand).

What you don't bother with:

- AppArmor policy tuning — Ubuntu's stock policy is fine.
- SELinux — Ubuntu isn't SELinux; don't port RHEL habits here.
- auditd — generates logs you won't read.
- Enterprise EDR — overkill.
- fail2ban — pointless on a desktop without exposed services.

## 8.3.1 UFW — the firewall

`ufw` is Ubuntu's simple firewall frontend. Enable default-deny-inbound:

```bash
sudo apt install -y ufw

# Default: deny all incoming, allow all outgoing. Standard.
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow responses to established outbound connections (implicit; listed for clarity):
# (UFW's stateful rules handle this.)

# Enable:
sudo ufw enable
sudo ufw status verbose
```

Expected output:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip
```

No allow rules by default. Nothing on your machine accepts incoming.

### Selective allows for dev workflows

If you run a local dev server and want to reach it from your phone on the same LAN:

```bash
# Allow port 3000 from LAN only (10.0.0.0/24 — adjust to your subnet):
sudo ufw allow from 192.168.1.0/24 to any port 3000 proto tcp
```

When the server is down, the rule is harmless. Reload:

```bash
sudo ufw reload
```

Never `sudo ufw allow 3000` without a source restriction unless you specifically need anyone-on-the-internet access (almost never for a laptop).

### Docker / Podman interaction

Rootless Podman does NOT add iptables rules that bypass UFW (unlike rootful Docker, which does). Your local Postgres on 5432 is reachable on localhost only — UFW blocks external access automatically.

## 8.3.2 Browser sandboxing with Firejail

Firejail is a SUID-root sandbox that restricts an app's access to the filesystem, network, kernel interfaces, and so on. The default Firejail profile for Firefox restricts Firefox to its own profile directory + `~/Downloads` + a few D-Bus interfaces it needs.

```bash
sudo apt install -y firejail firejail-profiles

# Test on Firefox:
firejail firefox
# Opens a Firefox in a sandboxed namespace. Confirm works, then exit.

# Make it the default for Firefox (desktop file):
firecfg
# This modifies ~/.local/share/applications/*.desktop for supported apps.
```

Apps firejail auto-wraps when present and profile exists:

- Firefox, Brave, Chromium
- Thunderbird
- Zoom, Signal-Desktop
- Discord, Telegram
- VLC, mpv
- LibreOffice
- GIMP, Inkscape

Notes:

- Some apps (e.g. Discord Flatpak) don't coexist with Firejail because they're already in a Flatpak sandbox. Don't double-sandbox.
- If an app misbehaves under Firejail (e.g. Firefox can't open a downloaded PDF), check `~/.config/firejail/firefox.local` — you may need to relax a filesystem restriction.

### Firejail vs Flatpak sandboxes

Flatpak's bubblewrap sandbox is generally stronger and more actively maintained than Firejail. For apps you install via Flatpak (Discord, Signal, Zen Browser), rely on Flatpak's sandbox — don't add Firejail.

For apt-installed apps (Firefox from Mozilla repo, Brave from Brave repo, Thunderbird), Firejail adds value.

## 8.3.3 YubiKey for SSH + 2FA (optional hardware)

If you own a YubiKey 5 series (or similar hardware token), it's a one-time ~30-minute setup that replaces your SSH key's disk file with hardware and shifts 2FA from TOTP-app to tap-to-confirm.

```bash
sudo apt install -y yubikey-manager ykman yubioath-desktop

# Check YubiKey is detected:
ykman info
# Expect: device name, serial, capabilities.
```

### SSH key on YubiKey (FIDO2 / Ed25519-SK)

OpenSSH 8.2+ supports hardware-backed Ed25519 keys via FIDO2. Generate:

```bash
ssh-keygen -t ed25519-sk -O resident -O application=ssh:github -C "yubikey-$(date +%Y%m%d)" -f ~/.ssh/id_ed25519_sk
```

The `-O resident` option stores the key on the YubiKey itself (can be re-exported to another machine by tapping). `-O application=ssh:github` segregates from other services.

Upload `~/.ssh/id_ed25519_sk.pub` to GitHub as you did the Ed25519 key in [5.2.6](../05-web-development/02-git-ssh-gpg.md#526-upload-the-ssh-public-key-to-github).

Now on every `git push`, YubiKey blinks; you tap to approve. Without the physical key, your SSH identity can't be used.

### Bitwarden with YubiKey 2FA

In Bitwarden web vault → Two-step login → FIDO2 WebAuthn → Register your YubiKey. On next login, USB tap is required in addition to password. Phishing-resistant.

### GPG on YubiKey (smart-card mode)

If you GPG-sign commits instead of SSH-signing (see [5.2.7](../05-web-development/02-git-ssh-gpg.md#527-commit-signing--ssh-signing-modern-simple)), you can put the GPG private key on the YubiKey. Requires more setup; covered in [Yubico's guide](https://developers.yubico.com/PGP/Importing_keys.html). Do this after you're comfortable with the SSH-on-YubiKey flow.

## 8.3.4 ClamAV — on-demand scanner (not realtime)

ClamAV is primarily useful for scanning incoming files (email attachments, USB sticks) for known Windows malware before forwarding them. Not useful as a real-time on-access scanner on Linux.

```bash
sudo apt install -y clamav clamav-freshclam

# Update signature database:
sudo systemctl stop clamav-freshclam
sudo freshclam
sudo systemctl start clamav-freshclam
sudo systemctl enable --now clamav-freshclam
```

On-demand scan:

```bash
# Scan a specific directory:
clamscan -r ~/Downloads
# Reports any infected files.

# Scan a USB stick:
clamscan -r /media/$USER/USBNAME
```

Don't run `clamav-daemon`. On-access scanning on a laptop costs battery and finds nothing that apt-package vetting and Firejail/Flatpak sandboxing don't already cover.

## 8.3.5 Web security practices

These aren't installs; they're habits:

1. **Keep browsers auto-updating.** You set Firefox to update via apt (Mozilla APT) in [2.1.3](../02-post-install-foundations/01-debloat-snap-flatpak-pim.md#213-firefox-via-mozillas-apt-repository); `sudo apt update && sudo apt upgrade` weekly keeps it current.
2. **Password manager, unique passwords per site.** Bitwarden from [4.5](../04-daily-driver-stack/05-comms-and-productivity.md).
3. **2FA everywhere it's offered** (GitHub, GitLab, Google, Vercel, Cloudflare). TOTP in KeePassXC + YubiKey where supported.
4. **Browser profile hygiene:** compartmentalise (multi-account containers in Firefox, or the multi-profile Firefox approach in [4.1.1](../04-daily-driver-stack/01-browsers.md#multi-profile-workflow)).
5. **Don't install random Chromium extensions.** Every extension has access to every page; vetted authors only.

## 8.3.6 Password hygiene summary

- **Bitwarden** (cloud-synced, master password) — daily credentials, recovery codes, shared with devices.
- **KeePassXC** (local file, separate master password) — anything you want absolutely local: TOTP secrets, offline codes, machine-local root passwords.
- **Keys on YubiKey** — SSH, primary 2FA WebAuthn. Own at least 2 YubiKeys for redundancy.

## 8.3.7 Disk encryption — revisited

We deliberately skipped LUKS in [1.4](../01-install/04-installation-and-partitioning.md). If the threat model changes (carrying proprietary research, regulated data), come back and enable LUKS via a reinstall or the `btrfs-convert + luksFormat-in-place` (risky) procedure. For a personal laptop today, the cost/benefit was against it.

## 8.3.8 Secure boot — revisited

Also deliberately off in [1.2.6](../01-install/02-windows-prep-and-bios.md#126-secure-boot--turn-off-for-install). If you want SB on:

1. Install `mokutil`.
2. Generate a MOK (Machine Owner Key) and enroll it.
3. Sign the NVIDIA DKMS module with it on every rebuild.
4. Handle kernel upgrades (DKMS re-sign step).

Doable, adds ~5 min to each NVIDIA DKMS rebuild. Benefit: blocks boot-time malware injection on the off-chance someone physically modifies your laptop. For most threat models this isn't worth it.

## 8.3.9 SSH hardening on the laptop (if you're an SSH client, not server)

We don't install `sshd` (laptops should not be servers). But when you SSH to things, use a strong client config.

`~/.ssh/config`:

```text
Host *
  Protocol 2
  HashKnownHosts yes
  HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256,ssh-mlkem768x25519-sha256@openssh.com
  KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,mlkem768x25519-sha256@openssh.com
  MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
  Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
  IdentitiesOnly yes
  AddKeysToAgent yes
  ControlMaster auto
  ControlPath ~/.ssh/socket/%r@%h:%p
  ControlPersist 4h
```

Create the socket dir: `mkdir -p ~/.ssh/socket`.

`ControlMaster` reuses TCP connections to the same host for 4 hours — makes multi-hop `ssh` and `rsync` faster.

## 8.3.10 What this system is NOT protected against

Be clear-eyed:

- **Physical theft** — laptop is stolen, someone with physical access can remove the SSD and read it (no LUKS). Mitigation: don't lose the laptop. Carry it like you'd carry $2000 cash.
- **Sophisticated malware via a 0-day in Firefox** — Firejail helps; doesn't eliminate. Mitigation: don't run Firefox as root; don't click random downloaded .deb files.
- **Social engineering** — phishing, SIM swap, YubiKey-removed coercion. Mitigation: 2FA on YubiKey + password manager + situational awareness.
- **Supply-chain attack via a trojaned npm package** — you install, it runs in your user context. Mitigation: `pnpm audit`; `sudo loginctl enable-linger` means rogue npm scripts could keep running post-logout — watch `systemctl --user list-units` periodically.

## 8.3.11 Snapshot

```bash
sudo timeshift --create --comments "post-security-hardening $(date -I)" --tags D
```

Section 08 complete. Proceed to [09 Apps catalog](../09-apps-catalog.md) or [10 Troubleshooting](../10-troubleshooting.md).

---

[Home](../README.md) · [↑ 08 Productivity & Security](README.md) · [← Previous: 8.2 Backup](02-backup-strategy-3-2-1.md) · **8.3 Firewall & sandboxing** · [Next: 09 Apps catalog →](../09-apps-catalog.md)
