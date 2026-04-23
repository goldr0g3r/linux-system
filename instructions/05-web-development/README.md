[Home](../README.md) · [← Previous: 4.6 Plasma UX](../04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md) · **05 — Web Development** · [Next: 5.1 Shell →](01-shell-and-terminal.md)

---

# 05 — Web Development

A reproducible, modern 2026 web-dev toolchain on Kubuntu 26.04: shell, Git identity, Node runtime manager, editors, frameworks, and local databases, all rootless where possible.

## Steps

| #   | Step                                                                           | Time    | Core?                    |
| ---:| ------------------------------------------------------------------------------ | ------- | ------------------------ |
| 5.1 | [Shell and terminal (Zsh + Starship, tmux/zellij, fzf/rg/bat/eza)](01-shell-and-terminal.md) | 15 min  | Yes                      |
| 5.2 | [Git, SSH, GPG (Ed25519 + ML-KEM hybrid, commit signing)](02-git-ssh-gpg.md)   | 20 min  | Yes                      |
| 5.3 | [Node.js runtimes and package managers (`fnm`, Corepack, Bun, Deno)](03-runtimes-and-package-managers.md) | 15 min | Yes                      |
| 5.4 | [Editors and IDEs (VS Code, Cursor, Zed, JetBrains, Helix)](04-editors-and-ides.md) | 20 min  | Yes                      |
| 5.5 | [Frameworks and local databases (Next.js, Astro, Postgres/Redis via Podman)](05-frameworks-and-databases.md) | 20 min | Yes                      |

Total: ~90 minutes for the complete stack.

## Toolchain summary

| Layer                 | Tool                                     | Why                                                                       |
| --------------------- | ---------------------------------------- | ------------------------------------------------------------------------- |
| Shell                 | **Zsh + Starship**                       | Modern, fast, cross-OS; Starship is one binary, no 17-plugin bashrc farm   |
| Terminal              | **Konsole** (fallback Yakuake drop-down) | KDE-native, best Wayland integration, tabs + split-panes mature            |
| Multiplexer           | **zellij** (fallback `tmux`)             | Modern UX, sensible defaults; `tmux` when you need scripting universality  |
| Version control       | **git** + **gh** (GitHub CLI)            | Standard; gh CLI handles PRs/issues from terminal                          |
| SSH keys              | **Ed25519 + ML-KEM hybrid** (post-quantum) | 26.04 defaults to post-quantum; use both now to future-proof              |
| Commit signing        | **GPG** (ed25519) or SSH-signed commits  | Signing is cheap; not signing is 2026's "no HTTPS"                         |
| Node runtime manager  | **fnm** (fast-node-manager)              | Rust binary, ~100x faster than `nvm`, auto-switches via `.nvmrc`           |
| JS package manager    | **pnpm** (via Corepack)                  | Content-addressable store, deterministic, massively faster on monorepos    |
| JS alternative runtime| **Bun**                                  | Rust-native npm alt + bundler + test runner, useful for some projects     |
| JS third runtime      | **Deno**                                 | TypeScript-first, secure-by-default; when you need it you'll know         |
| IDE primary           | **VS Code** (APT via `packages.microsoft.com`) | Ubiquitous, extension ecosystem, not snap                          |
| IDE AI-first          | **Cursor** (.deb)                        | VS Code fork with first-class AI integration                              |
| Editor fast/terminal  | **Helix**                                | Modal, batteries-included, no config needed to be productive               |
| DB Postgres           | **Podman Quadlet** `postgres:17-alpine`  | Rootless, per-project, survives reboot via `loginctl enable-linger`        |
| DB Redis              | **Podman Quadlet** `redis:7-alpine`      | Same pattern                                                              |
| Deploy                | **Vercel CLI** + **Cloudflare Wrangler** + **gh** | Three tools cover Vercel, Cloudflare Pages/Workers, and GitHub Actions triggers |

---

[Home](../README.md) · [← Previous: 4.6 Plasma UX](../04-daily-driver-stack/06-plasma-66-ux-tiling-activities.md) · **05 — Web Development** · [Next: 5.1 Shell →](01-shell-and-terminal.md)
