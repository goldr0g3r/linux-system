[Home](../README.md) · [↑ 05 Web Dev](README.md) · [← Previous: 5.1 Shell & terminal](01-shell-and-terminal.md) · **5.2 Git, SSH, GPG** · [Next: 5.3 Runtimes →](03-runtimes-and-package-managers.md)

---

# 5.2 Git, SSH, GPG — Identity, Signing, Post-Quantum Keys

Setting up git identity, generating both Ed25519 (classical) and ML-KEM-hybrid (post-quantum) SSH keys, configuring commit signing, and integrating the GitHub CLI.

## Why two SSH keys

Ubuntu 26.04 enables post-quantum SSH key-exchange (ML-KEM-768) hybrid with X25519 **for the session key**, but the **authentication key** (the Ed25519 key in `~/.ssh/id_ed25519`) is still classical. That's fine for today — the post-quantum threat to authentication keys is "harvest now, decrypt later" for signed artefacts, not live sessions. But because OpenSSH 10 (shipping with 26.04) supports ML-KEM hybrid keys for authentication too, you can generate one and use it on servers that support it (all modern OpenSSH 10+ servers do, which means GitHub as of 2025, Codeberg, Forgejo, and most self-hosted setups).

The recommendation:

1. **Ed25519** — your primary daily key, works everywhere.
2. **ML-KEM-768 + Ed25519 hybrid** — your second key, used on servers that accept it, and as your committed/stored "long-lived" identity for things that might be harvested.

## 5.2.1 Git identity

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Better defaults:
git config --global init.defaultBranch main
git config --global pull.rebase true
git config --global push.autoSetupRemote true
git config --global rerere.enabled true          # remember conflict resolutions
git config --global fetch.prune true             # drop deleted remote branches on fetch
git config --global diff.colorMoved zebra        # visualise moved code separately
git config --global merge.conflictstyle zdiff3   # clearer 3-way merge markers
git config --global core.editor "code --wait"    # or "nvim" / "hx" / "nano"
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.st 'status -sb'
git config --global alias.lg "log --graph --abbrev-commit --decorate --date=short --format=format:'%C(bold blue)%h%C(reset) - %C(bold green)(%ad)%C(reset) %C(white)%s%C(reset) %C(dim white)- %an%C(reset)%C(bold yellow)%d%C(reset)'"
```

## 5.2.2 Install GitHub CLI

```bash
# Add GitHub CLI's APT repo:
(type -p wget >/dev/null || (sudo apt update && sudo apt install wget -y)) \
  && sudo mkdir -p -m 755 /etc/apt/keyrings \
  && wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg | \
      sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
  && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | \
      sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
  && sudo apt update \
  && sudo apt install -y gh

# Verify:
gh --version
# Expect: gh version 2.x
```

## 5.2.3 Generate the primary SSH key (Ed25519)

```bash
ssh-keygen -t ed25519 -a 100 -C "$(hostname)-$(date +%Y%m%d)" -f ~/.ssh/id_ed25519
```

- `-a 100` — 100 KDF rounds; slows brute-force on the key file if it gets stolen. 100 is plenty; higher is paranoid.
- `-C` — comment. I use `hostname-date` so I can identify the key across machines.

Set a passphrase — not empty. The passphrase unlocks the key from disk; the `ssh-agent` (below) caches the unlocked state in RAM.

Permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

## 5.2.4 Generate a post-quantum hybrid SSH key

OpenSSH 10 (on 26.04) supports `ssh-mlkem768x25519-sha256@openssh.com`. Generate a hybrid key:

```bash
ssh-keygen -t mlkem-x25519 -a 100 -C "pq-hybrid-$(hostname)-$(date +%Y%m%d)" -f ~/.ssh/id_mlkem
# Note: the key-type argument name may vary by OpenSSH version; if the above fails:
# ssh-keygen -t mlkem768 -a 100 -C "..." -f ~/.ssh/id_mlkem
# or check `ssh-keygen -Q key | grep -i mlkem` for the supported names.
```

Add both keys to `~/.ssh/config`:

```bash
cat >> ~/.ssh/config <<'EOF'

Host github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentityFile ~/.ssh/id_mlkem
  IdentitiesOnly yes
  AddKeysToAgent yes

Host *
  AddKeysToAgent yes
  IdentitiesOnly yes
EOF

chmod 600 ~/.ssh/config
```

Now SSH will try Ed25519 first, fall back to ML-KEM if the server prefers it.

## 5.2.5 SSH agent via GNOME Keyring

Kubuntu 26.04 launches `gnome-keyring` as a KDE-session-compatible agent; it's the path of least resistance for SSH key caching (no need to run a separate `ssh-agent` yourself).

Verify:

```bash
echo $SSH_AUTH_SOCK
# Expect: /run/user/<uid>/keyring/ssh  (gnome-keyring's socket)

# Add key to agent (enter passphrase once per session):
ssh-add ~/.ssh/id_ed25519
ssh-add ~/.ssh/id_mlkem

# Verify:
ssh-add -l
# Both keys listed.
```

If `SSH_AUTH_SOCK` is empty (agent not running), enable:

```bash
systemctl --user enable --now ssh-agent.service
```

For a longer-running session (lingering after logout), combine with `loginctl enable-linger` from [3.3.6](../03-speed-and-boot-optimizations/03-service-masking-and-journald.md#336-linger-for-user-services-that-must-survive-logout).

## 5.2.6 Upload the SSH public key to GitHub

```bash
# Authenticate gh first:
gh auth login
# Pick SSH as auth method; gh uploads your ~/.ssh/id_ed25519.pub for you.

# Or manually:
cat ~/.ssh/id_ed25519.pub
# Copy the output, paste at https://github.com/settings/keys → "New SSH key".

# Same for the ML-KEM public key (GitHub as of 2025 accepts hybrid keys):
cat ~/.ssh/id_mlkem.pub
# Paste as a separate SSH key on GitHub.
```

Test:

```bash
ssh -T git@github.com
# Expect: "Hi <your-username>! You've successfully authenticated..."
```

If SSH tries the wrong key or times out:

```bash
ssh -vT git@github.com 2>&1 | head -30
# Debug: shows which keys are tried, which is accepted.
```

## 5.2.7 Commit signing — SSH signing (modern, simple)

Git 2.34+ can sign commits with an SSH key directly (no GPG needed). This is simpler than GPG and equally secure. Enable:

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

Create an allowed-signers file so `git log --show-signature` can verify:

```bash
mkdir -p ~/.ssh
tee ~/.ssh/allowed_signers <<EOF
you@example.com namespaces="git" $(cat ~/.ssh/id_ed25519.pub)
EOF

git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

Test:

```bash
cd /tmp
mkdir test-signing && cd test-signing
git init
git commit --allow-empty -m "test signed commit"
git log --show-signature
# Expect: "Good 'git' signature ..."
```

Upload the SSH public key on GitHub as a **signing** key (not auth key) under Settings → SSH and GPG keys → "New signing key". GitHub will show a green "Verified" badge on your commits.

### If you prefer GPG signing (classical)

```bash
sudo apt install -y gnupg pinentry-curses pinentry-gtk2

# Generate key:
gpg --full-generate-key
# Choose: ECC (9), Curve 25519 (1), 0 (no expiry or 2 years), name + email.

# Get the key ID:
gpg --list-secret-keys --keyid-format=long
# Copy the key ID from "sec   ed25519/<KEY_ID>"

# Configure git:
git config --global gpg.format openpgp
git config --global user.signingkey <KEY_ID>
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# Export public key:
gpg --armor --export <KEY_ID>
# Paste on GitHub → Settings → SSH and GPG keys → "New GPG key".
```

GPG is more intricate (subkeys, pinentry, agents) but battle-tested. SSH signing is newer and simpler; pick whichever fits your mental model.

## 5.2.8 Create working directories

```bash
mkdir -p ~/work/{web,robotics,ai,notes,scratch}

# Clone your main repos (examples):
cd ~/work/web
gh repo clone yourusername/your-blog
```

## 5.2.9 `.gitignore` globals

Things you never want to commit, ever, from any repo:

```bash
cat > ~/.gitignore_global <<'EOF'
# OS
.DS_Store
Thumbs.db

# Editors
.vscode/
.idea/
*.swp
*.swo

# Dependencies
node_modules/
__pycache__/
*.pyc
.venv/
venv/

# Builds
dist/
build/
target/
*.log

# Env secrets
.env
.env.local
.env.*.local
EOF

git config --global core.excludesfile ~/.gitignore_global
```

## 5.2.10 Git LFS (if you work with large binary assets)

```bash
sudo apt install -y git-lfs
git lfs install
```

Only relevant if you commit images, model weights (AI/ML), PDF assets, etc. Regular web-dev repos don't need it.

## 5.2.11 Snapshot

```bash
sudo timeshift --create --comments "post-git-ssh-gpg $(date -I)" --tags D
```

Proceed to [5.3 Node runtimes and package managers](03-runtimes-and-package-managers.md).

---

[Home](../README.md) · [↑ 05 Web Dev](README.md) · [← Previous: 5.1 Shell & terminal](01-shell-and-terminal.md) · **5.2 Git, SSH, GPG** · [Next: 5.3 Runtimes →](03-runtimes-and-package-managers.md)
