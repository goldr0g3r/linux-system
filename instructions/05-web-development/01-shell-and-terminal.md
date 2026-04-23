[Home](../README.md) · [↑ 05 Web Dev](README.md) · [← Previous: 05 Web Dev (section)](README.md) · **5.1 Shell & terminal** · [Next: 5.2 Git, SSH, GPG →](02-git-ssh-gpg.md)

---

# 5.1 Shell and Terminal Tooling

Zsh + Starship as the default shell, a small set of modern CLI replacements (`eza`, `bat`, `ripgrep`, `fd`, `fzf`, `zoxide`), and `zellij` (or `tmux`) for session multiplexing. One-time setup; daily dividends.

## 5.1.1 Install Zsh and make it default

```bash
sudo apt install -y zsh

# Set as default login shell:
chsh -s /bin/zsh

# Log out and back in (or `exec zsh` in a Konsole to test).
```

On first Zsh launch, you'll get a `zsh-newuser-install` wizard. Press `q` to skip — we're going to configure Zsh manually.

## 5.1.2 Install Starship as the prompt

```bash
curl -sS https://starship.rs/install.sh | sh
```

The installer drops `starship` in `/usr/local/bin/`. Activate at the bottom of `~/.zshrc`:

```bash
cat >> ~/.zshrc <<'EOF'

# ----- Starship prompt -----
eval "$(starship init zsh)"
EOF
```

Configure Starship via `~/.config/starship.toml`:

```bash
mkdir -p ~/.config
cat > ~/.config/starship.toml <<'EOF'
# Format
format = """
[╭─](bold green) $username$hostname$directory$git_branch$git_status$nodejs$python$rust$package$cmd_duration$jobs
[╰─](bold green)$character """

[username]
show_always = false
format = "[$user]($style)@"

[hostname]
ssh_only = true
format = "[$hostname]($style) "

[directory]
style = "bold cyan"
truncate_to_repo = true
truncation_length = 3

[git_branch]
symbol = " "
style = "bold purple"

[git_status]
ahead = " $count"
behind = " $count"
modified = " $count"
untracked = " $count"
staged = " $count"

[nodejs]
format = "[$symbol$version]($style) "
symbol = " "

[python]
format = "[$symbol${pyenv_prefix}(${version} )(\\($virtualenv\\) )]($style)"
symbol = " "

[cmd_duration]
min_time = 2000
format = " [$duration]($style)"
EOF
```

Open a new Konsole tab; you should see a clean, git-aware prompt.

## 5.1.3 Modern CLI replacements

| Traditional tool | Modern replacement | Install                          | Key advantages                                     |
| ---------------- | ------------------ | -------------------------------- | -------------------------------------------------- |
| `ls`             | `eza`              | `sudo apt install eza`           | Git status, icons, tree mode                       |
| `cat`            | `bat`              | `sudo apt install bat`           | Syntax highlighting, line numbers                  |
| `grep`           | `ripgrep` (`rg`)   | `sudo apt install ripgrep`       | Multi-GB faster, respects `.gitignore`             |
| `find`           | `fd-find` (`fd`)   | `sudo apt install fd-find`       | Intuitive syntax, colorised                        |
| `cd`             | `zoxide`           | `sudo apt install zoxide`        | Jumps to any dir by fuzzy match of history         |
| `du`             | `dust`             | `cargo install du-dust` or release | Interactive tree view of disk usage              |
| `df`             | `duf`              | `sudo apt install duf`           | Readable output                                    |
| `top`            | `btop`             | `sudo apt install btop`          | GPU + NIC + disk IO live graphs                    |
| `man`            | `tldr` (summary)   | `sudo apt install tealdeur`      | Community-curated cheat sheets                     |
| `kill`           | `killall` / `pkill`| (already installed)              | Target by name                                     |

Install the lot:

```bash
sudo apt install -y eza bat ripgrep fd-find duf btop tealdeur zoxide
```

On Ubuntu / Kubuntu, `fd-find` installs as `/usr/bin/fdfind` to avoid a naming clash. Alias:

```bash
echo 'alias fd=fdfind' >> ~/.zshrc
```

Also add the modern-tool aliases to `~/.zshrc`:

```bash
cat >> ~/.zshrc <<'EOF'

# ----- Modern CLI -----
alias ls='eza --icons --group-directories-first'
alias ll='eza -lah --icons --group-directories-first --git'
alias tree='eza --tree --icons'
alias cat='bat --paging=never'
alias grep='rg'
alias du='dust'
alias df='duf'
alias top='btop'

# zoxide replaces cd; `z foo` jumps to most-recently-used dir matching foo
eval "$(zoxide init zsh)"

# fzf key bindings:
[ -f /usr/share/doc/fzf/examples/key-bindings.zsh ] && source /usr/share/doc/fzf/examples/key-bindings.zsh
[ -f /usr/share/doc/fzf/examples/completion.zsh ] && source /usr/share/doc/fzf/examples/completion.zsh
EOF
```

## 5.1.4 fzf — the productivity multiplier

```bash
sudo apt install -y fzf
```

Key bindings (after sourcing the key-bindings.zsh above):

- `Ctrl+R` — fuzzy-search command history (repeated commands dedup'd, scored).
- `Ctrl+T` — insert file path(s) from fuzzy-searched current directory.
- `Alt+C` — `cd` into fuzzy-searched subdirectory.

Once you use `Ctrl+R` for history for a week, you'll never want to work on a machine without it.

Further fzf recipes — use these in daily workflow:

```bash
# Fuzzy-pick a git branch to check out:
git checkout "$(git branch -a --format='%(refname:short)' | fzf)"

# Fuzzy-kill a process:
ps aux | fzf --bind 'enter:execute(echo {+})+abort' | awk '{print $2}' | xargs -r kill

# Fuzzy-jump to an SSH host from ~/.ssh/config:
ssh "$(rg -o '^Host \S+' ~/.ssh/config | sed 's/^Host //' | fzf)"
```

Wrap frequently-used recipes as Zsh functions in `~/.zshrc`.

## 5.1.5 Install oh-my-zsh or not?

I recommend **NOT** using oh-my-zsh. Reasons:

- It's slow — loads dozens of plugins at shell startup, adds 200-500 ms to every `zsh` launch.
- Starship already does the prompt; oh-my-zsh's prompt themes are obsolete by comparison.
- The plugins you actually want (autocompletion, syntax highlighting) can be installed directly from APT.

Install these two plugins directly:

```bash
sudo apt install -y zsh-autosuggestions zsh-syntax-highlighting
```

And add to `~/.zshrc` (before the Starship init):

```bash
cat >> ~/.zshrc <<'EOF'

# ----- Zsh plugins -----
source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh

# ----- History -----
HISTSIZE=50000
SAVEHIST=50000
HISTFILE=~/.zsh_history
setopt HIST_IGNORE_ALL_DUPS
setopt HIST_IGNORE_SPACE
setopt INC_APPEND_HISTORY
setopt SHARE_HISTORY

# ----- Completion -----
autoload -Uz compinit && compinit
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}'  # case-insensitive
zstyle ':completion:*' menu select
EOF
```

## 5.1.6 The `fish` alternative (if you prefer)

Fish has better out-of-box UX than Zsh for some users — it autosuggests from history without a plugin, has tab-completion for flags, and doesn't need a 200-line `.zshrc`. Trade-off: it's not POSIX-compatible, so you can't paste random Bash one-liners into a fish shell without translation.

```bash
sudo apt install -y fish
chsh -s /usr/bin/fish
```

Starship works identically. `eza`, `bat`, `rg` etc. work identically. Use fish if you like; this guide continues assuming Zsh.

## 5.1.7 zellij (modern tmux)

Zellij is a Rust-based terminal multiplexer. Compared to tmux:

- Better defaults (panes, tabs, status bar).
- Layouts are declarative.
- Keybindings are less obscure.

Install:

```bash
# Via APT (should be in 26.04's universe):
sudo apt install -y zellij || {
  # Fallback: install via cargo (after 5.1.8 below):
  cargo install --locked zellij
}
```

Launch: `zellij`. Leader key: `Ctrl+G` (remappable).

Basic keys:

- `Ctrl+P` + `n` — new pane horizontal.
- `Ctrl+P` + `d` — new pane vertical.
- `Ctrl+T` + `n` — new tab.
- `Alt+h/j/k/l` — navigate panes.
- `Alt+H/J/K/L` — resize panes.
- `Ctrl+S` — toggle search mode.

For SSH-heavy workflows or servers without zellij, keep `tmux` as a second option:

```bash
sudo apt install -y tmux
```

Drop a minimal `~/.tmux.conf`:

```bash
cat > ~/.tmux.conf <<'EOF'
# ----- tmux config -----
set -g prefix C-a
unbind C-b
bind C-a send-prefix

# 24-bit color
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",*256col*:Tc"

# Mouse
set -g mouse on

# Status bar
set -g status-style 'bg=colour234 fg=colour137'
set -g status-left ''
set -g status-right '#{?client_prefix,#[reverse]<Prefix>#[noreverse] ,}%Y-%m-%d %H:%M'
set -g status-right-length 50

# Pane splits
bind - split-window -v -c "#{pane_current_path}"
bind | split-window -h -c "#{pane_current_path}"

# Reload config
bind r source-file ~/.tmux.conf \; display "Reloaded"
EOF
```

## 5.1.8 Install Rust (for `cargo` tools)

Several modern CLI tools are easier to install with the latest features via `cargo`:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable
source ~/.cargo/env
```

Now you can:

```bash
cargo install --locked \
  bottom \
  dust \
  tokei \
  ripgrep-all \
  hyperfine
```

(Note: `ripgrep`, `fd`, `bat` above were from apt — you can upgrade to latest via cargo if the apt versions are older than you need.)

## 5.1.9 Global key remap — CapsLock → Ctrl/Esc (via KeyD)

CapsLock is useless; remap it to Esc when tapped, Ctrl when held. This is a system-wide remap, not a shell thing, so it works everywhere (including VS Code, browsers).

Install KeyD:

```bash
sudo apt install -y keyd 2>/dev/null || {
  # Not in Ubuntu repos yet? Build from source:
  git clone https://github.com/rvaiya/keyd /tmp/keyd
  cd /tmp/keyd
  make && sudo make install
}
```

Configure `/etc/keyd/default.conf`:

```bash
sudo tee /etc/keyd/default.conf >/dev/null <<'EOF'
[ids]
*

[main]
capslock = overload(control, esc)
EOF

sudo systemctl enable --now keyd
sudo keyd reload
```

Test: tap CapsLock → Esc. Hold CapsLock + another key → Ctrl+key. Works in every app.

Covered in more depth at [8.1](../08-productivity-security/01-keyboard-automation.md).

## 5.1.10 Final ~/.zshrc

Put it together. Here's a complete, commented `~/.zshrc` to start from:

```bash
cat > ~/.zshrc <<'EOF'
# ~/.zshrc — Kubuntu 26.04 baseline

# Path
typeset -U path
path=(~/bin ~/.local/bin ~/.cargo/bin $path)

# Plugins
source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh

# History
HISTSIZE=50000
SAVEHIST=50000
HISTFILE=~/.zsh_history
setopt HIST_IGNORE_ALL_DUPS HIST_IGNORE_SPACE INC_APPEND_HISTORY SHARE_HISTORY

# Completion
autoload -Uz compinit && compinit
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}'
zstyle ':completion:*' menu select

# Modern CLI
alias ls='eza --icons --group-directories-first'
alias ll='eza -lah --icons --group-directories-first --git'
alias tree='eza --tree --icons'
alias cat='bat --paging=never'
alias top='btop'
alias fd='fdfind'

# zoxide
eval "$(zoxide init zsh)"

# fzf
[ -f /usr/share/doc/fzf/examples/key-bindings.zsh ] && source /usr/share/doc/fzf/examples/key-bindings.zsh
[ -f /usr/share/doc/fzf/examples/completion.zsh ] && source /usr/share/doc/fzf/examples/completion.zsh

# Distrobox quick-enter
alias dbe='distrobox enter'
alias ros2j='distrobox enter ros2-jazzy'

# Apt-with-snapshot
alias apts='apt-with-snapshot'

# Starship (last)
eval "$(starship init zsh)"
EOF
```

## 5.1.11 Snapshot

```bash
sudo timeshift --create --comments "post-shell-setup $(date -I)" --tags D
```

Proceed to [5.2 Git, SSH, GPG](02-git-ssh-gpg.md).

---

[Home](../README.md) · [↑ 05 Web Dev](README.md) · [← Previous: 05 Web Dev (section)](README.md) · **5.1 Shell & terminal** · [Next: 5.2 Git, SSH, GPG →](02-git-ssh-gpg.md)
