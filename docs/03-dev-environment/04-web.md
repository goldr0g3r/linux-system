[Home](../../README.md) · [↑ Dev Environment](README.md) · [← Previous: 3.3 AI/ML](03-ai-ml.md) · **3.4 Web** · [Next: 3.5 Roadmap →](05-learning-roadmap.md)

---

# 3.4 Next.js Blog Workflow

`fnm` (Fast Node Manager) for Node version switching, `pnpm` via corepack, Vercel CLI for deploys, optional containerized Node workspace for riskier experiments.

## Why `fnm` over `nvm`

- **`nvm`** is a ~1000-line bash script. Sourcing it adds ~150 ms to every shell startup. Every new Konsole tab pays that tax.
- **`fnm`** is a single static Rust binary. Shell overhead ~5 ms. 30x faster.
- Same interface: `fnm install`, `fnm use`, `fnm default`. Drop-in.

## Install `fnm`

```bash
curl -fsSL https://fnm.vercel.app/install | bash
# The installer appends to ~/.bashrc:
#   eval "$(fnm env --use-on-cd)"
source ~/.bashrc

fnm --version
```

`--use-on-cd` is the key feature: `cd` into a directory containing `.nvmrc` or `.node-version` and `fnm` auto-switches the active Node version.

## Install Node LTS + corepack + pnpm

```bash
fnm install --lts                 # latest Node LTS (20.x or 22.x)
fnm install 20                    # explicit major for project pinning
fnm default 22                    # default shell version
node --version
corepack enable                   # ships pnpm, yarn, bun shims managed via package.json
pnpm --version
```

`corepack` is a Node built-in that reads `packageManager: "pnpm@9.x"` from `package.json` and auto-invokes the right version. You never have to `npm install -g pnpm` again.

## Vercel CLI (deploy the blog from terminal)

```bash
pnpm add -g vercel
vercel --version
vercel login       # opens browser for OAuth
```

From your blog repo, one-time link + deploy:

```bash
cd ~/Blog
vercel link                    # link this directory to your Vercel project
vercel env pull .env.local     # pull production env vars for local dev
pnpm dev                       # localhost:3000
```

Deploy:

```bash
vercel           # preview URL
vercel --prod    # production
```

## Optional: containerized Node workspace

Useful when:

- Testing a different Node major (15/16/17) that clashes with your host's `fnm default`.
- A dependency pulls in a sketchy postinstall script you do not trust.
- You want fully reproducible CI-like builds.

```bash
distrobox create --name web-node \
  --image docker.io/library/ubuntu:24.04 \
  --home ~/Containers/web-node

distrobox enter web-node
# Inside the container:
#   - Install fnm + Node identically.
#   - `cd` into your host-bound repo path (host $HOME is visible).
#   - `pnpm dev` serves on localhost:3000, accessible from the host because
#     container sees host loopback by default via Distrobox networking.
```

If you need an isolated network (not sharing host's localhost), re-create with `--additional-flags "--publish 3000:3000"`.

### Practical stance

Keep `pnpm` on the host for the blog — it is lightweight, the Next.js dev server benefits from direct filesystem access, and you save the per-command Distrobox-enter overhead. Use the container only for riskier Node work.

## `.nvmrc` / `.node-version` in your blog repo

Pin the Node major you deployed to, so `fnm` auto-switches when you `cd` in:

```bash
cd ~/Blog
echo "22" > .node-version
git add .node-version && git commit -m "Pin Node 22"
```

Now when you open a new Konsole tab with the `blog` profile (from [2.7](../02-setup/07-ui-ux.md)) and `cd ~/Blog`, fnm silently activates Node 22.

## Useful global packages (via pnpm + corepack)

These are the few things genuinely useful as globals. Everything else goes in the blog's `devDependencies`.

```bash
pnpm add -g \
  vercel \
  @biomejs/biome \
  sharp \
  http-server
```

- **biome** — fast alternative to ESLint + Prettier, single Rust binary.
- **sharp** — used by Next.js image optimization locally.
- **http-server** — one-liner static server for quick sharing (`npx http-server`).

## Recommended Next.js setup for the blog

Assuming a headless-CMS blog (Contentful / Sanity / Strapi / Directus / Payload CMS):

```bash
# New project from scratch:
cd ~/
pnpm create next-app@latest blog --typescript --tailwind --app --src-dir --import-alias '@/*'
cd blog

# Key additions:
pnpm add \
  next-mdx-remote \
  @portabletext/react \
  contentful \
  rehype-slug rehype-autolink-headings remark-gfm \
  framer-motion \
  date-fns \
  zod

pnpm add -D \
  @types/node \
  prettier \
  @biomejs/biome \
  eslint-config-next

# Optional: analytics, CMS auth, etc.
pnpm add @vercel/analytics @vercel/speed-insights
```

## Blog → Markdown → Obsidian integration

Keep your draft markdown files in Obsidian (see [Part 4](../04-applications.md)) and publish with a sync script:

```bash
# ~/Blog/scripts/sync-from-obsidian.sh
#!/usr/bin/env bash
set -euo pipefail

VAULT="$HOME/Notes/Blog-Drafts"
POSTS="$HOME/Blog/content/posts"

rsync -av --delete \
  --include='*.md' --include='*/' --exclude='*' \
  "$VAULT/" "$POSTS/"

cd "$HOME/Blog"
pnpm dev
```

Then in Obsidian you write, save; the script rsyncs into the Next.js repo, Hot Module Reload picks up the new post, you preview at `localhost:3000/posts/<slug>`.

## Verification

```bash
which fnm node corepack pnpm vercel
fnm list
node --version
pnpm --version
vercel --version

# In a test Next.js dir:
mkdir -p /tmp/nextjs-test && cd /tmp/nextjs-test
pnpm create next-app@latest . --ts --tailwind --app
pnpm dev
# Visit http://localhost:3000 — default Next.js landing.
```

Proceed to [3.5 Learning roadmap](05-learning-roadmap.md).

---

[Home](../../README.md) · [↑ Dev Environment](README.md) · [← Previous: 3.3 AI/ML](03-ai-ml.md) · **3.4 Web** · [Next: 3.5 Roadmap →](05-learning-roadmap.md)
