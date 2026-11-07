# Mac Mini Server

A practical guide to running websites, automation and local AI on a Mac — from the Cocoaheads talk
**"Creating an AI-enabled Mac Mini Server."**

This repo is a software installation guide. It covers macOS settings, Docker, n8n, Postgres,
Cloudflare Tunnel, local models with Ollama, and how to connect your own native Mac tools to the
rest of the stack.

> **Last verified:** November 2026, on macOS 27 / Apple silicon. Image versions are pinned in the
> templates. Check for newer versions before you start.

---

## The idea in one picture

```
Internet
   │
   ▼
Cloudflare (SSL, Access login)            ← nothing open on your router
   │  outbound-only tunnel
   ▼
┌──────────────────────── Your Mac ────────────────────────┐
│  Docker Desktop                                          │
│   ├─ tunnel stack:  cloudflared                          │
│   └─ one stack per project:                              │
│        nginx website · n8n · pgAdmin · Postgres          │
│                    │                                     │
│                    │  http://host.docker.internal:<port> │
│                    ▼                                     │
│  Native macOS (outside Docker)                           │
│   ├─ Ollama (local models, runs on the GPU)              │
│   └─ Your own Swift tools as LaunchAgent HTTP services   │
└──────────────────────────────────────────────────────────┘
```

- **Docker is the portable layer.** A project's stack moves to any Linux, Windows or Mac server
  as-is.
- **Native macOS is the capability layer.** Local models and Apple frameworks run outside Docker,
  and containers call them like any other API.

---

## Three ways to use this

1. **Learning.** Start on the Mac you already have. Docker Desktop and Ollama run fine on a laptop.
   Follow the guide up through local models and skip the tunnel.
2. **Hosting client work.** A dedicated, always-on Mac running one stack per client, exposed through
   Cloudflare Tunnel.
3. **In-house and private.** The same setup with the tunnel removed. Everything stays on the
   office network.

---

## What's in the repo

| Path | What it is |
|---|---|
| [`GUIDE.md`](GUIDE.md) | The full step-by-step guide |
| `stack-template/` | One project stack: Postgres, n8n, pgAdmin and an nginx website |
| `tunnel/` | The Cloudflare Tunnel stack (cloudflared as a container) |
| `scripts/pg-backup.sh` | `pg_dump` backup for a stack's database |
| `native-bridge/` | A minimal Swift HTTP service and its LaunchAgent |

---

## Quick start

```bash
# 1. Copy the template for a new project
cp -R stack-template ~/projects/myproject
cd ~/projects/myproject

# 2. Fill in your settings
cp .env.example .env
open -e .env            # set project name, passwords, n8n encryption key, hostname

# 3. Start it
docker compose up -d
docker compose ps       # Postgres should show "healthy"
```

Then open n8n at `http://127.0.0.1:<N8N_LOCAL_PORT>` and create your owner account.

Read [`GUIDE.md`](GUIDE.md) before you expose anything to the internet.

---

## Ground rules this setup follows

- **Pin every image tag.** `:latest` will eventually jump a major version on you.
- **`restart: unless-stopped`** on every service.
- **No host ports by default.** The tunnel reaches containers by name. Any port you do publish
  binds to `127.0.0.1` only.
- **Every admin page goes behind a login** (Cloudflare Access) before it goes on the internet.
- **Recreate containers after config changes.** Editing a compose file does not change a running
  container.
- **Back up the database, not just the repo.** n8n stores workflows and credentials in Postgres.

---

## License

MIT. Use it, change it, share it.
