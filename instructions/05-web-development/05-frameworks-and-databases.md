[Home](../README.md) · [↑ 05 Web Dev](README.md) · [← Previous: 5.4 Editors](04-editors-and-ides.md) · **5.5 Frameworks & databases** · [Next: 06 ROS 2 →](../06-ros2-robotics/README.md)

---

# 5.5 Frameworks and Local Databases — Next.js, Astro, Vite, Postgres/Redis via Podman

The last web-dev page: bootstrapping a Next.js blog / Astro site / Vite app, and running Postgres + Redis + MinIO locally via rootless Podman so they survive reboot without a system daemon.

## 5.5.1 Prerequisite: rootless Podman

If you haven't already enabled rootless Podman (the [docs/03-dev-environment/01-containers.md](../../docs/03-dev-environment/01-containers.md) pattern), do it now:

```bash
sudo apt install -y podman distrobox uidmap slirp4netns fuse-overlayfs

# Enable rootless socket:
systemctl --user enable --now podman.socket

# Verify rootless:
podman info | grep -E "rootless|graphDriver"
# Expect: rootless: true, Driver: overlay (or fuse-overlayfs)

# Container config (avoids journald spam, enables systemd cgroup integration):
mkdir -p ~/.config/containers
cat > ~/.config/containers/containers.conf <<'EOF'
[engine]
events_logger = "file"
cgroup_manager = "systemd"

[containers]
label = false
pids_limit = 0
EOF
```

Enable linger so containers survive logout:

```bash
sudo loginctl enable-linger $USER
```

## 5.5.2 Podman Quadlet — the modern way to run long-lived containers

Quadlet is Podman's native systemd-unit format. You write a simple `.container` file, systemd handles startup/restart/journal integration.

Directory: `~/.config/containers/systemd/`.

## 5.5.3 Postgres 17 via Quadlet

```bash
mkdir -p ~/.config/containers/systemd ~/Containers/postgres/data

cat > ~/.config/containers/systemd/postgres.container <<'EOF'
[Unit]
Description=Postgres 17 (local dev)
After=network.target

[Container]
Image=docker.io/postgres:17-alpine
ContainerName=postgres
PublishPort=5432:5432
Environment=POSTGRES_PASSWORD=dev
Environment=POSTGRES_USER=dev
Environment=POSTGRES_DB=dev
Environment=POSTGRES_INITDB_ARGS=--data-checksums
Volume=%h/Containers/postgres/data:/var/lib/postgresql/data:Z
HealthCmd=pg_isready -U dev
HealthInterval=10s
HealthRetries=3

[Service]
Restart=on-failure
TimeoutStartSec=30

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user start postgres.service

# Verify:
systemctl --user status postgres.service
psql -h localhost -U dev -d dev -c '\l'
# Password prompt: 'dev'. Expect list of databases including 'dev'.
```

Wire into VS Code with `cweijan.vscode-postgresql-client2` extension, or use `psql`, `pgcli`, or `DBeaver` (Flatpak: `io.dbeaver.DBeaverCommunity`).

### Schema migrations

Pick one tool, use consistently:

- **Drizzle** — TypeScript-first, generates SQL from TS schema; pair with Drizzle Kit for migrations. Best for Next.js + TypeScript.
- **Prisma** — classic; schema file separate from TS. Generates a client.
- **Kysely** + **Kysely-codegen** — more manual, more control; good for complex queries.

Your `src/db/schema.ts` with Drizzle:

```typescript
// @ts-nocheck
import { pgTable, serial, text, timestamp } from 'drizzle-orm/pg-core';

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  slug: text('slug').notNull().unique(),
  title: text('title').notNull(),
  content: text('content').notNull(),
  createdAt: timestamp('created_at').notNull().defaultNow(),
});
```

```bash
pnpm add drizzle-orm postgres
pnpm add -D drizzle-kit
pnpm drizzle-kit generate
pnpm drizzle-kit push
```

## 5.5.4 Redis 7 via Quadlet

```bash
mkdir -p ~/Containers/redis

cat > ~/.config/containers/systemd/redis.container <<'EOF'
[Unit]
Description=Redis 7 (local dev)
After=network.target

[Container]
Image=docker.io/redis:7-alpine
ContainerName=redis
PublishPort=6379:6379
Volume=%h/Containers/redis:/data:Z
Exec=redis-server --appendonly yes

[Service]
Restart=on-failure

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user start redis.service

# Test:
redis-cli ping
# Expect: PONG
```

## 5.5.5 MinIO (S3-compatible object storage) via Quadlet

For Next.js image uploads, blog media, etc. without paying for S3:

```bash
mkdir -p ~/Containers/minio

cat > ~/.config/containers/systemd/minio.container <<'EOF'
[Unit]
Description=MinIO S3-compatible storage
After=network.target

[Container]
Image=quay.io/minio/minio:latest
ContainerName=minio
PublishPort=9000:9000
PublishPort=9001:9001
Environment=MINIO_ROOT_USER=dev
Environment=MINIO_ROOT_PASSWORD=devpassword
Volume=%h/Containers/minio:/data:Z
Exec=server /data --console-address ":9001"

[Service]
Restart=on-failure

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user start minio.service
```

Console: http://localhost:9001 (user `dev`, pw `devpassword`). API: http://localhost:9000.

## 5.5.6 Next.js — scaffold a blog

```bash
cd ~/work/web
pnpm create next-app@latest my-blog
# Choose: TypeScript Yes, ESLint Yes, Tailwind Yes, `src/` Yes, App Router Yes, customize import alias No.

cd my-blog
pnpm add drizzle-orm postgres @tanstack/react-query
pnpm add -D drizzle-kit @types/pg
```

### Dev server

```bash
pnpm dev
```

Opens at http://localhost:3000. Hot-reload works out-of-box; Wayland + Firefox renders crisply.

### Deploy to Vercel

```bash
pnpm add -g vercel
vercel login
vercel
# Answers: bind to new / existing project, pick scope, deploy.
```

`vercel --prod` deploys production. First-class DX — this is why Vercel owns Next.js hosting.

### Deploy to Cloudflare Pages

Next.js 15 supports Cloudflare's edge runtime via `@cloudflare/next-on-pages`:

```bash
pnpm add -D @cloudflare/next-on-pages wrangler
pnpx next-on-pages
wrangler pages deploy .vercel/output/static
```

## 5.5.7 Astro — for content-first sites

```bash
cd ~/work/web
pnpm create astro@latest my-astro-site
# Answers: empty / blog starter, strict TypeScript, install deps yes, init git yes.

cd my-astro-site
pnpm dev
# Opens at http://localhost:4321.
```

Astro is excellent for blog / marketing sites because it generates static HTML by default (fastest Core Web Vitals), with opt-in interactive islands via React/Vue/Svelte/Solid/Lit.

## 5.5.8 Vite — for single-page apps

Raw Vite (without Next.js or Astro framing) for React / Vue / Svelte SPAs:

```bash
cd ~/work/web
pnpm create vite@latest my-spa
# Choose: React + TypeScript + SWC (SWC is the Rust-based replacement for Babel, faster).

cd my-spa
pnpm install
pnpm dev
```

## 5.5.9 Environment variables management

Never commit `.env`. Use:

```bash
# .env.local (gitignored)
DATABASE_URL=postgres://dev:dev@localhost:5432/dev
REDIS_URL=redis://localhost:6379
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=dev
S3_SECRET_KEY=devpassword

# .env.example (committed, with placeholders)
DATABASE_URL=
REDIS_URL=
S3_ENDPOINT=
S3_ACCESS_KEY=
S3_SECRET_KEY=
```

For a production deploy, inject env vars via Vercel / Cloudflare / systemd Environment directive — never in a committed file.

## 5.5.10 Local HTTPS for OAuth / cookies development

Some features (PWA install, strict cookies, Stripe webhooks) demand HTTPS even on localhost. Use `mkcert`:

```bash
sudo apt install -y mkcert libnss3-tools
mkcert -install
mkcert localhost 127.0.0.1 ::1
# Produces localhost+2.pem and localhost+2-key.pem.
```

For Next.js custom HTTPS dev (wrapper script):

```bash
# create https-dev.mjs
cat > https-dev.mjs <<'EOF'
import { createServer } from 'https';
import { parse } from 'url';
import next from 'next';
import fs from 'fs';

const app = next({ dev: true });
const handle = app.getRequestHandler();

const options = {
  key: fs.readFileSync('./localhost+2-key.pem'),
  cert: fs.readFileSync('./localhost+2.pem'),
};

await app.prepare();

createServer(options, (req, res) => {
  handle(req, res, parse(req.url, true));
}).listen(3000);
EOF

node https-dev.mjs
# Opens at https://localhost:3000 with a browser-trusted cert.
```

## 5.5.11 Container lifecycle management

```bash
# Start/stop:
systemctl --user start|stop|restart postgres.service

# View logs:
journalctl --user -u postgres.service -f

# Upgrade image version:
sudoedit ~/.config/containers/systemd/postgres.container
# Change "Image=docker.io/postgres:17-alpine" to "postgres:18-alpine" etc.
systemctl --user daemon-reload
systemctl --user restart postgres.service
# Old container is removed, new image pulled, data volume reused.
```

Backups of Postgres data: pg_dump snapshots. The container is recreateable; the data volume (`~/Containers/postgres/data/`) is what matters — it's included in your Restic backup from [8.2](../08-productivity-security/02-backup-strategy-3-2-1.md).

## 5.5.12 Docker Compose compatibility

If a project ships a `docker-compose.yml`:

```bash
# podman-compose is a Python wrapper that reads docker-compose.yml and drives podman:
sudo apt install -y podman-compose

# Usage identical:
podman-compose up -d
podman-compose ps
podman-compose down
```

For new projects, prefer Quadlet (systemd-integrated); for consuming open-source projects with docker-compose.yml, `podman-compose` works fine.

## 5.5.13 Snapshot

```bash
sudo timeshift --create --comments "post-frameworks-and-dbs $(date -I)" --tags D
```

Section 05 complete. Proceed to [06 ROS 2 Robotics](../06-ros2-robotics/README.md).

---

[Home](../README.md) · [↑ 05 Web Dev](README.md) · [← Previous: 5.4 Editors](04-editors-and-ides.md) · **5.5 Frameworks & databases** · [Next: 06 ROS 2 →](../06-ros2-robotics/README.md)
