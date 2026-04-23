[Home](../README.md) · [↑ 05 Web Dev](README.md) · [← Previous: 5.2 Git, SSH, GPG](02-git-ssh-gpg.md) · **5.3 Runtimes & package managers** · [Next: 5.4 Editors →](04-editors-and-ides.md)

---

# 5.3 Node.js Runtimes and Package Managers

`fnm` for runtime version management, Corepack for package-manager shims (pnpm/yarn/npm), Bun as a fast alternative, and Deno for TypeScript-first scripting.

## Why `fnm`, not `nvm` or system Node

- **System Node from APT** — 26.04's Ubuntu archive ships Node 22 LTS. Fine for one-off use; terrible for a dev machine where different projects pin different Node versions.
- **`nvm`** — the ubiquitous Bash script. Works, but adds 2-3 s to shell startup and is miserable on fast SSDs because it `stat`s dozens of files on every prompt.
- **`fnm`** — Rust-compiled binary, ~10x faster than `nvm`, reads `.nvmrc` / `.node-version` per project automatically, fully drop-in.

## 5.3.1 Install fnm

```bash
curl -fsSL https://fnm.vercel.app/install | bash -s -- --skip-shell

# Activate in ~/.zshrc:
cat >> ~/.zshrc <<'EOF'

# ----- fnm -----
export PATH="$HOME/.local/share/fnm:$PATH"
eval "$(fnm env --use-on-cd --shell zsh)"
EOF

source ~/.zshrc
```

Verify:

```bash
fnm --version
# Expect: fnm 1.x
```

## 5.3.2 Install Node versions

```bash
# Latest LTS (22.x at time of 26.04's release):
fnm install --lts

# Specific version:
fnm install 20
fnm install 18

# Use one globally:
fnm default 22
fnm use 22

# Verify:
node --version
npm --version
```

### Per-project version pinning

In any project directory:

```bash
echo '22' > .nvmrc
# or
echo '22.10.0' > .node-version
```

Now when you `cd` into that project, `fnm` auto-switches (because `--use-on-cd`). Exit the dir → reverts.

## 5.3.3 Enable Corepack (pnpm / yarn / npm shims)

Corepack ships with Node 16+. It lets you declare the package manager (pnpm, yarn, npm) in `package.json` and auto-uses the correct version without manual installation.

```bash
corepack enable

# Verify:
pnpm --version
yarn --version
```

In a project, pin the PM:

```json
// package.json
{
  "packageManager": "pnpm@9.12.0"
}
```

Now `pnpm install` in that directory uses exactly pnpm 9.12.0, regardless of what's installed globally.

## 5.3.4 Which package manager — recommendation

For new projects: **pnpm**. Reasons:

- Content-addressable store → one copy of each package on disk, even across projects. On a dev laptop with 10 Node projects, that's tens of GB saved vs npm.
- Deterministic: `pnpm-lock.yaml` resolves to the same tree on every machine.
- Fast: ~2–3x faster than npm on cold cache, ~5x faster on warm.
- Great monorepo support (`pnpm-workspace.yaml`).

```bash
corepack prepare pnpm@latest --activate
pnpm --version
# Expect: 9.x or 10.x (depending on 2026 cadence)
```

For working on legacy projects, `yarn` / `npm` are still one command away via corepack.

## 5.3.5 Bun — faster runtime + bundler + test runner + package manager

Bun is a Rust/Zig-native alternative to Node. It is a runtime, bundler, test runner, AND package manager in one binary. Not a drop-in Node replacement for every project (native addon compatibility lags Node), but outstanding for:

- CLI tools written in TypeScript — `bun run` starts 10x faster than `node` + `tsx`.
- Quick API prototypes — `bun` has a built-in HTTP server that's faster than Node's.
- Running ESM-first TypeScript without a transpiler.

Install:

```bash
curl -fsSL https://bun.sh/install | bash
source ~/.zshrc

bun --version
# Expect: 1.x or higher
```

When to use Bun in a project: mark it as the package manager in package.json (`"packageManager": "bun@1.2.x"`) and commit `bun.lockb`. For Next.js specifically (as of 2026), Bun's Next.js support is good but pnpm is still slightly more battle-tested.

## 5.3.6 Deno — TypeScript-first runtime (secondary)

Deno is the opposite philosophical take from Node: secure-by-default (no fs/network access without `--allow-*` flags), TypeScript built-in (no transpile step), ESM-only. Use for:

- Scripts that need filesystem access with explicit permissions.
- TypeScript scripts without npm.
- Rapid prototyping where `deno run https://example.com/script.ts` is the whole workflow.

Install:

```bash
curl -fsSL https://deno.land/install.sh | sh
# Add to PATH in ~/.zshrc:
echo 'export PATH="$HOME/.deno/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

deno --version
```

Low-priority install unless you have an existing Deno project.

## 5.3.7 `uv` — for Python (not Node, but bundled here for completeness)

Python's equivalent of "just install the fast Rust binary and forget about pyenv/pip/venv/poetry." Covered in depth in [7.2](../07-ai-ml-workstation/02-python-uv-pytorch-jax.md), but install here so web-dev Python scripts have it:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.zshrc
uv --version
```

## 5.3.8 Global tools: keep minimal, use `bun pm run` or `pnpm dlx`

Resist the temptation to `pnpm add -g` everything. Global installs pin versions out-of-band with your projects. Instead:

- Use `pnpm dlx <tool>` (like `npx`) for one-shot invocations.
- Use `bunx <tool>` for Bun-runnable tools.
- Pin frequently-used tools as project `devDependencies` in your `package.json`.

The few tools that really should be global (because they don't belong to a specific project):

```bash
# Vercel CLI (for deploying Next.js blogs):
pnpm add -g vercel

# Cloudflare Wrangler (for Pages / Workers):
pnpm add -g wrangler

# tsx (direct-execute TypeScript in Node):
# no — use pnpm dlx tsx instead of global install.

# TypeScript compiler:
# no — always pin per-project.
```

## 5.3.9 Installation verification

```bash
node --version
npm --version
pnpm --version
bun --version
deno --version 2>/dev/null || echo "(deno not installed)"
uv --version
```

## 5.3.10 Environment variables for Node projects

In `~/.zshrc`, add:

```bash
cat >> ~/.zshrc <<'EOF'

# ----- Node.js tuning -----
# Increase memory limit for large projects (e.g. Next.js build, TypeScript watch):
export NODE_OPTIONS="--max-old-space-size=8192"

# Disable telemetry:
export NEXT_TELEMETRY_DISABLED=1
export ASTRO_TELEMETRY_DISABLED=1
export GATSBY_TELEMETRY_DISABLED=1
EOF
```

## 5.3.11 Snapshot

```bash
sudo timeshift --create --comments "post-runtimes-and-pms $(date -I)" --tags D
```

Proceed to [5.4 Editors and IDEs](04-editors-and-ides.md).

---

[Home](../README.md) · [↑ 05 Web Dev](README.md) · [← Previous: 5.2 Git, SSH, GPG](02-git-ssh-gpg.md) · **5.3 Runtimes & package managers** · [Next: 5.4 Editors →](04-editors-and-ides.md)
