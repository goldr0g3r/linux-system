[Home](../README.md) · [↑ 05 Web Dev](README.md) · [← Previous: 5.3 Runtimes](03-runtimes-and-package-managers.md) · **5.4 Editors & IDEs** · [Next: 5.5 Frameworks →](05-frameworks-and-databases.md)

---

# 5.4 Editors and IDEs

VS Code as primary, Cursor as AI-first secondary, Zed as a lightweight third option, JetBrains Toolbox for people who want WebStorm / PyCharm / RustRover, and Helix for terminal purists.

## 5.4.1 VS Code (primary)

**Do not install the snap.** Install from Microsoft's own APT repo — you get auto-updates via `apt update` and no snap confinement issues.

```bash
# Key:
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg
rm packages.microsoft.gpg

# Repo:
echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" | \
  sudo tee /etc/apt/sources.list.d/vscode.list

# Install:
sudo apt update
sudo apt install -y code

# Verify:
code --version
# Expect: 1.96.x or later
```

### Wayland support

VS Code runs under XWayland by default. For native Wayland (better HiDPI, less lag, proper scaling):

Edit `~/.config/code-flags.conf`:

```text
--enable-features=UseOzonePlatform,WaylandWindowDecorations
--ozone-platform-hint=auto
```

Or launch with flags. Wayland-native VS Code is good in 2026; no longer considered unstable.

### Core extensions

Install via command line:

```bash
code --install-extension ms-python.python
code --install-extension ms-vscode.cpptools-extension-pack
code --install-extension ms-vscode-remote.remote-containers
code --install-extension eamodio.gitlens
code --install-extension esbenp.prettier-vscode
code --install-extension dbaeumer.vscode-eslint
code --install-extension ms-vscode.vscode-typescript-next
code --install-extension bradlc.vscode-tailwindcss
code --install-extension yzhang.markdown-all-in-one
code --install-extension tamasfe.even-better-toml
code --install-extension redhat.vscode-yaml
code --install-extension christian-kohler.path-intellisense
code --install-extension usernamehw.errorlens
code --install-extension rust-lang.rust-analyzer

# AI assistants:
code --install-extension continue.continue             # Ollama integration, covered in 7.3
code --install-extension github.copilot                # paid, optional
code --install-extension github.copilot-chat           # paired with Copilot
```

### settings.json template

`~/.config/Code/User/settings.json`:

```json
{
  "editor.fontFamily": "'JetBrainsMono Nerd Font', 'Fira Code', monospace",
  "editor.fontLigatures": true,
  "editor.fontSize": 13,
  "editor.lineHeight": 1.6,
  "editor.rulers": [80, 120],
  "editor.renderWhitespace": "boundary",
  "editor.bracketPairColorization.enabled": true,
  "editor.inlineSuggest.enabled": true,
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit",
    "source.organizeImports": "explicit"
  },
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.autoSave": "onFocusChange",
  "files.exclude": {
    "**/node_modules": true,
    "**/.next": true,
    "**/dist": true,
    "**/.venv": true
  },
  "search.exclude": {
    "**/node_modules": true,
    "**/.next": true,
    "**/pnpm-lock.yaml": true,
    "**/bun.lockb": true
  },
  "terminal.integrated.fontFamily": "'JetBrainsMono Nerd Font'",
  "terminal.integrated.defaultProfile.linux": "zsh",
  "workbench.colorTheme": "Default Dark Modern",
  "workbench.iconTheme": "material-icon-theme",
  "window.titleBarStyle": "native",
  "window.menuBarVisibility": "toggle",
  "[javascript]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[typescript]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[typescriptreact]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "[python]": { "editor.defaultFormatter": "charliermarsh.ruff" },
  "[json]": { "editor.defaultFormatter": "esbenp.prettier-vscode" },
  "typescript.preferences.importModuleSpecifier": "non-relative",
  "explorer.confirmDelete": true,
  "git.autofetch": true,
  "git.confirmSync": false
}
```

### Profiles for context switching

VS Code Profiles (File → Profiles → Create Profile) let you have different extension sets and settings per context:

- **Default** — web-dev extensions, Prettier, ESLint.
- **Python** — python extension, Ruff, pylance.
- **ROS 2** — C++ extensions, CMake tools.

Switch profiles via **File → Profiles → Switch Profile**.

## 5.4.2 Cursor (AI-first)

Cursor is a VS Code fork with native AI integration. If you pay for Cursor's subscription or just want the UX, install the `.deb`:

```bash
# Download from https://www.cursor.com (Linux .deb)
cd ~/Downloads
wget https://downloader.cursor.sh/linux/appImage/x64 -O cursor.AppImage
# Or .deb directly if they publish one:
# wget https://downloader.cursor.sh/linux/deb/x64 -O cursor.deb
# sudo apt install ./cursor.deb

# If AppImage:
chmod +x cursor.AppImage
mkdir -p ~/.local/bin ~/.local/share/applications
mv cursor.AppImage ~/.local/bin/cursor

# Create a .desktop:
tee ~/.local/share/applications/cursor.desktop <<EOF
[Desktop Entry]
Name=Cursor
Exec=$HOME/.local/bin/cursor %U
Icon=cursor
Terminal=false
Type=Application
Categories=Development;IDE;
EOF
```

Cursor imports your VS Code extensions and settings on first run, so migrating from VS Code is smooth.

### When to use Cursor vs VS Code + Continue.dev

- **Cursor**: first-class AI completions, chat, diff-apply (the "agentic" flow that VS Code doesn't have as cleanly). Subscription required for the best models.
- **VS Code + Continue.dev + Ollama**: private, local, free. Tab-autocomplete is slower (~500 ms vs Cursor's 100 ms) but offline. Covered in [7.3](../07-ai-ml-workstation/03-local-llms-ollama-openwebui.md).

Pick one as your primary. I use Cursor for greenfield + Claude flows, VS Code for long-living refactors.

## 5.4.3 Zed (lightweight)

Zed is a Rust-native editor from the ex-Atom team. Collaborative editing is a superpower. Install via APT:

```bash
# Zed's official APT repo:
curl -f https://zed.dev/install.sh | sh

# Or if you prefer Flatpak:
# flatpak install flathub dev.zed.Zed

zed --version
```

Zed is extremely fast and has excellent Vim mode built-in. Use for:

- Fast one-off edits.
- Live-collaboration with a team on the same file (like Google Docs for code).
- Projects where VS Code feels heavy.

Not a full VS Code replacement — extensions ecosystem is smaller, debugger support is behind VS Code.

## 5.4.4 JetBrains Toolbox (for WebStorm / PyCharm / RustRover)

Some workflows live in JetBrains' IDEs — huge Java/Kotlin projects, Python scientific work (PyCharm's debugger is still the best in the industry), Rust development with RustRover. Install the Toolbox:

```bash
# Download Toolbox from https://www.jetbrains.com/toolbox-app
cd ~/Downloads
wget https://download.jetbrains.com/toolbox/jetbrains-toolbox-<latest>.tar.gz
tar -xzf jetbrains-toolbox-*.tar.gz
./jetbrains-toolbox-*/jetbrains-toolbox  # first run; installs to ~/.local/share/JetBrains
```

Toolbox then manages installations of WebStorm, IntelliJ, PyCharm, RustRover, GoLand, etc. All are subscription-based (JetBrains Pro or individual products); students free via GitHub Education.

## 5.4.5 Helix (modal terminal editor)

Helix is a Rust-native modal editor — think Vim with modern defaults, tree-sitter-based highlighting out of the box, and LSP built-in.

```bash
sudo apt install -y helix
# Or via repo if Ubuntu's package is too old:
# sudo add-apt-repository ppa:maveonair/helix-editor
# sudo apt install helix

hx --version
# Expect: 25.x or later (depends on 26.04 archive freshness)
```

Config `~/.config/helix/config.toml`:

```toml
theme = "catppuccin_mocha"

[editor]
line-number = "relative"
mouse = true
bufferline = "multiple"
color-modes = true
cursor-shape = { insert = "bar", normal = "block", select = "underline" }

[editor.lsp]
display-inlay-hints = true

[editor.indent-guides]
render = true
character = "▎"
```

Install language servers:

```bash
# TypeScript:
pnpm add -g typescript-language-server typescript

# Rust:
rustup component add rust-analyzer

# Python:
uv tool install pyright
```

Helix auto-detects these on PATH.

Use Helix for: quick edits in terminal, SSH to remote, anything where a GUI IDE is overkill.

## 5.4.6 Which do I use day-to-day?

Personal recommendation:

- **VS Code** (open most of the day) — daily driver for web projects.
- **Cursor** (open alongside VS Code) — for AI-heavy tasks and new feature prototyping.
- **Helix** (in Konsole, always a keystroke away) — quick edits, SSH, one-shot scripts.
- **JetBrains (PyCharm)** (monthly) — only when debugging gnarly Python scientific code.
- **Zed** (occasionally) — when a teammate wants to pair-program on a file.

You don't need all five. Start with VS Code + Helix. Add Cursor if you like the AI workflow.

## 5.4.7 Editor config standards

Drop `.editorconfig` in every repo for consistent indentation/line-endings across editors:

```ini
# .editorconfig
root = true

[*]
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 2
charset = utf-8
trim_trailing_whitespace = true

[*.py]
indent_size = 4

[*.md]
trim_trailing_whitespace = false
```

All of the above editors honour `.editorconfig` natively.

## 5.4.8 Snapshot

```bash
sudo timeshift --create --comments "post-editors $(date -I)" --tags D
```

Proceed to [5.5 Frameworks and databases](05-frameworks-and-databases.md).

---

[Home](../README.md) · [↑ 05 Web Dev](README.md) · [← Previous: 5.3 Runtimes](03-runtimes-and-package-managers.md) · **5.4 Editors & IDEs** · [Next: 5.5 Frameworks →](05-frameworks-and-databases.md)
