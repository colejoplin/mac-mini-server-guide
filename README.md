# Mac Mini Server Guide

Run websites, n8n automation and local AI on a Mac. Companion to the Cocoaheads talk
**"Creating an AI-enabled Mac Mini Server."**

This is a software setup guide: macOS settings, Docker, one project stack, putting it online
safely, local models, and connecting your own Swift tools to the rest of the stack.

> **Last verified:** November 2026, macOS 27 on Apple silicon. Image versions are pinned in the
> template. Check for newer versions before you start.

---

## The idea in one picture

```
Internet
   │
   ▼
Cloudflare (HTTPS, Access login)          ← nothing open on your router
   │  outbound-only tunnel
   ▼
┌──────────────────────── Your Mac ────────────────────────┐
│  Docker  (the portable layer)                            │
│    one stack per project:                                │
│    nginx website · n8n · pgAdmin · Postgres              │
│                    │                                     │
│                    │  http://host.docker.internal:<port> │
│                    ▼                                     │
│  Native macOS  (the capability layer)                    │
│    Ollama · your own Swift tools                         │
└──────────────────────────────────────────────────────────┘
```

- **Docker is portable.** A project's stack moves to any Linux, Windows or Mac server as-is.
- **Native macOS is your edge.** Local models and Apple frameworks run outside Docker, and
  containers call them like any other API.

---

## Three ways to use this

1. **Learning.** Start on the Mac you already have. Skip the "putting it online" section.
2. **Hosting client work.** An always-on Mac running one stack per client, behind a tunnel.
3. **In-house and private.** The same setup with no tunnel. Everything stays on the office network.

---

## What's here

| Path | What it is |
|---|---|
| [`GUIDE.md`](GUIDE.md) | The setup guide |
| [`stack-template/`](stack-template) | One project stack: Postgres, n8n, pgAdmin and an nginx website |

## Quick start

```bash
cp -R stack-template ~/projects/myproject
cd ~/projects/myproject
cp .env.example .env
open -e .env              # change every CHANGE_ME
docker compose up -d
docker compose ps         # postgres should show "healthy"
```

Then open n8n at `http://127.0.0.1:5680` (or your `N8N_LOCAL_PORT`).

Read [`GUIDE.md`](GUIDE.md) before you put anything on the internet.

---

## The rules that matter

- **Pin every image tag.** `:latest` will eventually jump a major version on you.
- **Nothing listens on your network.** Published ports bind to `127.0.0.1`; the tunnel reaches
  containers by name.
- **Every admin page gets a login in front** before it goes on the internet.
- **Recreate containers after config changes.** Editing a compose file doesn't change a running
  container.
- **Back up the database and the n8n encryption key.** Your automations live in Postgres, not
  your repo.

---

## License

MIT
