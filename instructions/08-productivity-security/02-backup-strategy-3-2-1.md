[Home](../README.md) · [↑ 08 Productivity & Security](README.md) · [← Previous: 8.1 Keyboard](01-keyboard-automation.md) · **8.2 Backup (3-2-1)** · [Next: 8.3 Firewall & sandboxing →](03-firewall-sandboxing.md)

---

# 8.2 Backup Strategy — Restic + rclone + 3-2-1

**Timeshift is not a backup. Timeshift is a rollback tool.** Real backups live on a different physical device. This page sets up Restic for encrypted, incremental, deduplicated backups plus a 3-2-1 schedule.

## The 3-2-1 rule

At least **3 copies** of your data, on **2 different storage media**, with **1 off-site**. Applied to your laptop:

- **Copy 1**: live data on the laptop's BTRFS.
- **Copy 2**: Restic repo on an external USB SSD, rotated weekly.
- **Copy 3**: Restic repo on cloud storage (Backblaze B2, Cloudflare R2, Wasabi, or rsync.net), hourly/daily.

Both Restic repos are encrypted with the same password — loss of the external SSD doesn't leak data.

## What to back up

| Location                 | Back up?   | Why                                            |
| ------------------------ | ---------- | ---------------------------------------------- |
| `~/` (your home)         | **Yes**    | Your actual work, dotfiles, notes, photos     |
| `/etc/`                  | Yes        | System config (cap on growth; small)           |
| `~/Containers/`          | Selectively| Podman volumes: yes. Container images: no.    |
| `~/.local/share/Trash/`  | No         | Exclude                                        |
| `~/.cache/`              | No         | Regenerable                                    |
| `~/node_modules/` anywhere | No      | pnpm store re-fetches                           |
| `~/.venv/`, `.venv/`     | No         | uv sync regenerates                           |
| `~/work/<project>/.venv/`| No         | uv sync regenerates                            |
| `/opt/ros/*`             | No         | apt can reinstall                              |
| `/var/log/*`             | No         | Pre-rotated, not critical                      |
| `/boot`                  | No         | apt can reinstall                              |
| VM images (if any)       | Maybe      | Large; decide per-VM                           |

Rule: back up things you **created** (notes, code, photos, configs), not things that can be **re-downloaded** (caches, venvs, container images, installed packages).

## 8.2.1 Install Restic

```bash
sudo apt install -y restic

restic version
# Expect: 0.17.x or later.
```

## 8.2.2 Set up the external-SSD repository

Plug in an external USB SSD. Format it once (use exFAT or ext4):

```bash
lsblk -f
# Identify the external SSD — say /dev/sdb1 (it's USB, not NVMe).

# For ext4 (recommended for Restic — Linux-native, handles hard links):
sudo mkfs.ext4 -L restic-backup /dev/sdb1

# Create a persistent mount point:
sudo mkdir -p /mnt/backup
echo 'LABEL=restic-backup  /mnt/backup  ext4  noatime,nofail,x-systemd.automount,x-systemd.device-timeout=5  0 2' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount /mnt/backup
```

Initialize the Restic repository:

```bash
# Generate a strong password (keep somewhere safe — password manager, printed paper in a safe):
openssl rand -base64 32 > ~/.restic-password.txt
chmod 600 ~/.restic-password.txt

# Create repo:
RESTIC_PASSWORD_FILE=~/.restic-password.txt restic init --repo /mnt/backup/restic-repo
# Expect: "created restic repository ..."
```

**Record the password somewhere you can read it if your laptop is destroyed.** Without it, the backup is useless.

## 8.2.3 First backup

```bash
export RESTIC_PASSWORD_FILE=~/.restic-password.txt

restic -r /mnt/backup/restic-repo backup \
  ~/ \
  --tag initial \
  --exclude-caches \
  --exclude-file=<(cat <<'EOF'
~/.cache/**
~/.local/share/Trash/**
~/Containers/**
~/.venv/**
~/snap/**
~/node_modules/**
~/**/node_modules/**
~/**/.venv/**
~/**/.next/**
~/**/target/**
~/**/dist/**
~/**/build/**
~/**/.git/objects/pack/*.pack
EOF
)
```

First backup is full: will take 30 minutes to several hours depending on `~/` size. Subsequent backups are differential/deduplicating.

## 8.2.4 Verify and list snapshots

```bash
restic -r /mnt/backup/restic-repo snapshots
# Expect one row: ID, Time, Host, Tags, Paths.

# Check repo health:
restic -r /mnt/backup/restic-repo check
# Run with --read-data monthly for deep integrity.
```

## 8.2.5 Schedule daily backups via systemd timer

Create `/etc/systemd/system/restic-local.service`:

```bash
sudo tee /etc/systemd/system/restic-local.service >/dev/null <<'EOF'
[Unit]
Description=Restic daily backup to external SSD
After=network.target

[Service]
Type=oneshot
User=YOURUSERNAME
Environment="RESTIC_PASSWORD_FILE=/home/YOURUSERNAME/.restic-password.txt"
ExecStart=/usr/bin/restic -r /mnt/backup/restic-repo backup \
    /home/YOURUSERNAME \
    --tag daily \
    --exclude-caches \
    --exclude="/home/YOURUSERNAME/.cache" \
    --exclude="/home/YOURUSERNAME/.local/share/Trash" \
    --exclude="/home/YOURUSERNAME/Containers" \
    --exclude="/home/YOURUSERNAME/.venv" \
    --exclude="/home/YOURUSERNAME/snap" \
    --exclude-file=/etc/restic/exclude-patterns.txt

ExecStartPost=/usr/bin/restic -r /mnt/backup/restic-repo forget \
    --keep-daily 7 --keep-weekly 5 --keep-monthly 12 --prune

Nice=19
IOSchedulingClass=idle
EOF

sudo mkdir -p /etc/restic
sudo tee /etc/restic/exclude-patterns.txt >/dev/null <<'EOF'
**/node_modules
**/.venv
**/venv
**/__pycache__
**/.next
**/target/debug
**/target/release
**/dist
**/build
**/.git/objects/pack/*.pack
EOF
```

Replace `YOURUSERNAME` with your actual username.

Then the timer:

```bash
sudo tee /etc/systemd/system/restic-local.timer >/dev/null <<'EOF'
[Unit]
Description=Run Restic local backup daily

[Timer]
OnCalendar=daily
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now restic-local.timer
systemctl list-timers restic-local.timer
```

Test the service manually:

```bash
sudo systemctl start restic-local.service
journalctl -u restic-local.service -f
# Watch output; takes minutes.
```

## 8.2.6 Off-site backup via rclone to Backblaze B2

Backblaze B2 is $6/TB/month — cheapest credible cloud storage with decent latency for 2026. Cloudflare R2, Wasabi, Hetzner Storage Box are alternatives.

```bash
sudo apt install -y rclone

# Configure (interactive):
rclone config
```

Answer: `n` (new remote), name `b2`, storage `b2`, paste your Application Key ID + Key from [secrets.backblaze.com](https://secrets.backblaze.com/). Confirm.

Test:

```bash
rclone lsd b2:
# Expect: list of your B2 buckets (initially empty or one you pre-created).

# Create a bucket (or via B2 web UI):
rclone mkdir b2:my-laptop-backup-2026
```

### Restic to B2 via rclone remote

```bash
# Init a second Restic repo, backed by B2 via rclone:
export RESTIC_PASSWORD_FILE=~/.restic-password.txt
restic -r "rclone:b2:my-laptop-backup-2026/restic" init

# First upload:
restic -r "rclone:b2:my-laptop-backup-2026/restic" backup ~/ \
  --tag offsite-initial \
  --exclude-caches \
  --exclude-file=/etc/restic/exclude-patterns.txt
```

First upload takes hours (your home is 10-50 GB; B2 upload is limited by your ISP upload speed). Subsequent uploads are tiny deltas (MB range).

### Scheduled offsite backup

```bash
sudo tee /etc/systemd/system/restic-offsite.service >/dev/null <<'EOF'
[Unit]
Description=Restic daily backup to Backblaze B2
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=YOURUSERNAME
Environment="RESTIC_PASSWORD_FILE=/home/YOURUSERNAME/.restic-password.txt"
ExecStart=/usr/bin/restic -r "rclone:b2:my-laptop-backup-2026/restic" backup \
    /home/YOURUSERNAME \
    --tag daily-offsite \
    --exclude-caches \
    --exclude-file=/etc/restic/exclude-patterns.txt

ExecStartPost=/usr/bin/restic -r "rclone:b2:my-laptop-backup-2026/restic" forget \
    --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune

Nice=19
IOSchedulingClass=idle
EOF

sudo tee /etc/systemd/system/restic-offsite.timer >/dev/null <<'EOF'
[Unit]
Description=Daily offsite Restic backup

[Timer]
OnCalendar=daily
RandomizedDelaySec=2h
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now restic-offsite.timer
```

Run manually first time:

```bash
sudo systemctl start restic-offsite.service
```

## 8.2.7 Restore walkthrough

When something goes wrong:

```bash
# List available snapshots:
restic -r /mnt/backup/restic-repo snapshots

# Restore the most recent snapshot into /tmp/restore:
restic -r /mnt/backup/restic-repo restore latest --target /tmp/restore

# Restore only a specific path:
restic -r /mnt/backup/restic-repo restore latest --target /tmp/restore --include /home/YOURUSERNAME/work

# Mount a snapshot for browsing:
mkdir /tmp/mount
restic -r /mnt/backup/restic-repo mount /tmp/mount &
cd /tmp/mount/snapshots/latest/
# ... browse files ...
fusermount -u /tmp/mount  # unmount
```

## 8.2.8 Test the restore every 6 months

A backup that's never been restored is a theoretical backup. Schedule a restore drill every 6 months:

```bash
# Restore a small file you chose at random, verify contents match:
restic -r /mnt/backup/restic-repo restore latest --target /tmp/restore-test --include /home/YOURUSERNAME/work/notes/scratch.md
diff ~/work/notes/scratch.md /tmp/restore-test/home/YOURUSERNAME/work/notes/scratch.md
# Expect: no output (files identical).
```

If the diff is non-empty, something is wrong — investigate before you need the backup for real.

## 8.2.9 Alternatives

- **Kopia** — similar to Restic, maintained, has a GUI. More beginner-friendly.
- **Borg Backup** — older, solid, scripting-oriented.
- **Duplicity** — GPG-encrypted, tar-based. Older model; prefer Restic.
- **rsnapshot** — rsync-based snapshots. Fast to set up, no dedup.
- **Syncthing** — file synchronisation, not a backup (no versioning). Useful alongside Restic, not instead of.

Pick Restic unless you have a specific reason for another. It's fast, encrypted, deduplicated, mountable, battle-tested.

## 8.2.10 rsync.net as an alternative off-site

rsync.net is a sysadmin-friendly SSH-only cloud storage. Slightly more expensive than B2 ($18/TB/month) but has a native "Restic, Borg, Kopia, rsnapshot-as-a-Service" pricing tier for $7/TB/month.

Preference: B2 if cost-sensitive, rsync.net if you want simpler SSH-only semantics.

## 8.2.11 Password rotation

Never rotate your Restic password unless you have reason to believe it's leaked. Rotating means re-encrypting the whole repo (expensive) and losing access to prior snapshots until decrypted. Store the password securely once and leave it.

## 8.2.12 Snapshot

```bash
sudo timeshift --create --comments "post-backup-strategy $(date -I)" --tags D
```

Proceed to [8.3 Firewall and sandboxing](03-firewall-sandboxing.md).

---

[Home](../README.md) · [↑ 08 Productivity & Security](README.md) · [← Previous: 8.1 Keyboard](01-keyboard-automation.md) · **8.2 Backup (3-2-1)** · [Next: 8.3 Firewall & sandboxing →](03-firewall-sandboxing.md)
