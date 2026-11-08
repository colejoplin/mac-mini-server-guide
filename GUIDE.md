# Mac Mini Server Guide

> **Last verified:** November 2026, macOS 27 on Apple silicon.
> Replace `myproject` and `example.com` with your own names.

1. [Make the Mac behave like a server](#1-make-the-mac-behave-like-a-server)
2. [Docker Desktop](#2-docker-desktop)
3. [Your first stack](#3-your-first-stack)
4. [Putting it online](#4-putting-it-online)
5. [Backups](#5-backups)
6. [Local models](#6-local-models)
7. [Your own Swift tools](#7-your-own-swift-tools)
8. [Updating](#8-updating)
9. [Gotchas](#9-gotchas)

---

## 1. Make the Mac behave like a server

Skip this if you're learning on a laptop.

```bash
sudo pmset -a sleep 0          # never sleep
sudo pmset -a autorestart 1    # power back on after an outage
pmset -g                       # check both
```

### Decide on FileVault

Docker Desktop, Ollama and any LaunchAgent run **inside your login session**. After a restart,
nothing starts until someone logs in, and FileVault blocks automatic login.

| Choice | After a restart |
|---|---|
| **FileVault on + SSH unlock** (recommended) | Waits at the login screen until you unlock it remotely. Data stays encrypted. |
| **FileVault off + automatic login** | Logs in and starts everything on its own. Anyone holding the Mac can read the data. |

**SSH unlock (macOS 26+, Apple silicon):** turn on **Remote Login** (System Settings → General →
Sharing), use wired Ethernet, and after a restart run `ssh you@your-mac.local` from another machine
on the same network. Enter your password; the session drops while macOS finishes booting.

### UPS

If you use one, connect its **USB data** cable and check that macOS sees it:

```bash
pmset -g ups      # empty output means macOS can't see it
```

---

## 2. Docker Desktop

1. Install Docker Desktop for Apple silicon.
2. Settings → General → **Start Docker Desktop when you sign in**.
3. Check: `docker compose version`

> **Why not Apple's native containers?** They're promising, but they don't yet restart a crashed
> service, and Compose support comes only from community add-ons. Docker also matches what other
> servers run, so your stacks stay portable.

---

## 3. Your first stack

```bash
mkdir -p ~/projects
cp -R stack-template ~/projects/myproject
cd ~/projects/myproject
cp .env.example .env
open -e .env
```

In `.env`, change every `CHANGE_ME` (use `openssl rand -base64 32`), set
`COMPOSE_PROJECT_NAME`, and pick local ports no other stack uses. **Copy `N8N_ENCRYPTION_KEY` to
your password manager now.**

```bash
docker compose up -d
docker compose ps        # postgres "healthy", everything else "running"
```

| Service | Local address |
|---|---|
| n8n | `http://127.0.0.1:<N8N_LOCAL_PORT>` (create the owner account) |
| pgAdmin | `http://127.0.0.1:<PGADMIN_LOCAL_PORT>` |
| Website | `http://127.0.0.1:<WEB_LOCAL_PORT>` |

Inside pgAdmin, connect to host `postgres`, port `5432`, with your Postgres user.

**Another project?** Copy the template again with a new name and new ports. One stack per project.

---

## 4. Putting it online

Use a **Cloudflare Tunnel**: cloudflared connects outbound, so nothing opens on your router. Follow
Cloudflare's own tunnel docs for setup, run cloudflared however you like, and follow these rules:

1. **Join cloudflared to each stack's network** (`myproject-network`) so it can reach containers by
   name.
2. **Route to `http://<container>:<port>`**, e.g. `http://myproject-n8n:5678`. Use `http://`, not
   `https://`, because Cloudflare handles HTTPS at its edge.
3. **Put a login in front of every admin page.** In Cloudflare Access, add a one-time-PIN policy for
   your n8n and pgAdmin hostnames.
4. **Let webhooks through.** Add a second Access app for `n8n.example.com/webhook` with a
   **Bypass** policy, *before* gating n8n, or your webhooks stop working.
5. **Set `N8N_SECURE_COOKIE=true`** once n8n is served over HTTPS, then recreate it.
6. **Test from your phone on cellular.** Admin pages should ask for a PIN; websites and webhooks
   should not.

---

## 5. Backups

| What | Where it lives | Back it up with |
|---|---|---|
| Code, website | Your repo | git |
| **n8n workflows and credentials, app data** | **Postgres** | **`pg_dump`** |
| **n8n encryption key** | `.env` | Your password manager |

```bash
# Back up (from the stack folder)
mkdir -p backups
docker exec myproject-postgres pg_dump -U myproject myproject > backups/$(date +%F-%H%M).sql

# Restore into a fresh, empty stack
docker exec -i myproject-postgres psql -U myproject -d myproject < backups/<file>.sql
```

Take one before every upgrade. Practice a restore once. Time Machine is a good safety net for the
machine, but a copy of a **running** database's files isn't a reliable database backup. Keep at
least one copy off the Mac.

---

## 6. Local models

Local models run on the Mac's **GPU**. Docker on a Mac **can't reach the GPU**, so run Ollama
natively and call it from containers:

```
http://host.docker.internal:11434
```

Unified memory is shared by macOS, Docker and your model. Check what the stack already uses before
choosing one:

```bash
docker stats --no-stream     # memory per container
ollama ps                    # memory per loaded model
```

Prefer small models that do one job well.

---

## 7. Your own Swift tools

Docker containers are Linux and can't run Mac apps. But your Swift tools can run **beside**
Docker, and containers call them the same way they call Ollama:

```
n8n (Docker) → http://host.docker.internal:<port> → your Swift tool on 127.0.0.1:<port>
```

The pattern:

- A command-line tool that listens on **`127.0.0.1` only** (the Network framework is enough)
- Requires a token header on every request
- Kept running by a **LaunchAgent** with `KeepAlive`
- Takes JSON in, writes files or returns JSON

That makes every Apple framework you already use (AVFoundation, Vision, Core Image, Core ML, Core
Graphics) a backend for a website or an automation.

---

## 8. Updating

1. Back up.
2. Change **one** pinned tag in `docker-compose.yml`. Read the release notes for major versions.
3. `docker compose pull <service> && docker compose up -d <service>`
4. Check `docker compose logs --tail 50 <service>` and the live site.

---

## 9. Gotchas

Every one of these seemed obvious afterward.

| Symptom | Fix |
|---|---|
| A config change "didn't work" | `docker compose up -d --force-recreate <service>`. Editing the file doesn't change a running container. |
| A service jumped a major version | Pin every image tag; never use `:latest`. |
| Bad gateway through the tunnel | Route to `http://<container>:<port>`, not `https://`. |
| Tunnel can't reach a new stack | Add that stack's network to cloudflared and recreate it. |
| `localhost:5678` hit the wrong n8n | Several stacks on one Mac: use container names and unique local ports. |
| Webhooks broke after adding Access | Add the `/webhook` Bypass app. |
| pgAdmin had no login | Keep server mode on (the template does). |
| Backups are huge | n8n execution history. The template prunes after 7 days. |
| Imported workflow runs old logic | Imported isn't published. Publish it, then test live. |
| Nothing came back after a power cut | FileVault blocked login. See section 1. |
