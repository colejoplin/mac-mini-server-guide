# Mac Mini Server — Installation Guide

Step-by-step setup for a Mac that hosts websites, n8n automation, Postgres and local AI.

> **Last verified:** November 2026, macOS 27 on Apple silicon. Commands assume an admin account.
> Replace every `myproject`, `example.com` and placeholder value with your own.

**Contents**
1. [macOS server settings](#1-macos-server-settings)
2. [Docker Desktop](#2-docker-desktop)
3. [Your first project stack](#3-your-first-project-stack)
4. [Cloudflare Tunnel](#4-cloudflare-tunnel)
5. [Locking down admin pages](#5-locking-down-admin-pages)
6. [Backups](#6-backups)
7. [Local models with Ollama](#7-local-models-with-ollama)
8. [The native bridge](#8-the-native-bridge)
9. [Updating safely](#9-updating-safely)
10. [Gotchas](#10-gotchas)

---

## 1. macOS server settings

By default a Mac behaves like a desktop. These settings make it behave like a server.
Skip this section if you're learning on a laptop.

### Never sleep, and restart after a power failure

```bash
sudo pmset -a sleep 0          # never sleep the system
sudo pmset -a autorestart 1    # power back on after a power failure
pmset -g                       # verify: sleep 0, autorestart 1
```

The display can still sleep; that doesn't affect services.

### Decide on FileVault deliberately

Docker Desktop, Ollama and any LaunchAgent you install run **inside your login session**. After a
restart, nothing starts until someone logs in.

FileVault blocks automatic login. That leaves two choices:

| Choice | Result after a restart |
|---|---|
| **FileVault on + remote unlock** (recommended) | The Mac waits at the login screen until you unlock it over SSH. Data stays encrypted. |
| **FileVault off + automatic login** | The Mac logs in and every service starts on its own. Anyone with the machine can read the data. |

**Remote unlock (macOS 26 and later, Apple silicon):**

1. Turn on **Remote Login** in System Settings → General → Sharing.
2. Use a **wired Ethernet** connection for the most reliable pre-boot network.
3. After a restart, from another machine on the same network:
   ```bash
   ssh youruser@your-mac.local
   ```
   Enter your password at the prompt. The SSH session drops while macOS finishes booting. Then
   your services start normally.

Check your current state:

```bash
fdesetup status
sysadminctl -autologin status
```

### If you use a UPS

Connect its **USB data cable** (not just power). Then confirm macOS sees it:

```bash
pmset -g ups
```

If this prints nothing, macOS can't see the UPS and can't shut down cleanly on low battery.
Plug your router into the UPS too: the tunnel and remote unlock both need the network.

---

## 2. Docker Desktop

1. Install **Docker Desktop for Mac (Apple silicon)** from docker.com.
2. Settings → General → turn on **Start Docker Desktop when you sign in**.
3. Settings → Resources: the default memory allocation is fine to start. Raise it only if
   `docker stats` shows containers near the limit.
4. Verify:
   ```bash
   docker version
   docker compose version
   ```

> **Why Docker instead of Apple's container tooling?** Apple's native containers (macOS 26+) are
> promising, but they don't yet support restart policies, and Compose support comes only from
> community add-ons. For a server, a service that doesn't come back after a crash isn't ready.
> Docker also matches what most other servers run, so your stacks stay portable.

---

## 3. Your first project stack

Each project gets one identical, isolated stack in its own folder:

```
~/projects/myproject/
├── docker-compose.yml
├── .env                 # your secrets (never commit this)
├── postgres/            # database files
├── n8n/                 # n8n settings
├── pgadmin/             # pgAdmin settings
├── nginx/default.conf   # website config
├── website/             # your static site
└── backups/             # pg_dump output
```

### Create it

```bash
mkdir -p ~/projects
cp -R stack-template ~/projects/myproject
cd ~/projects/myproject
cp .env.example .env
```

### Fill in `.env`

Every value marked `CHANGE_ME` must change. In particular:

- **`PROJECT`**: a short name like `myproject`. It names the containers and the network.
- **`POSTGRES_PASSWORD`**, **`PGADMIN_DEFAULT_PASSWORD`**: long random strings.
- **`N8N_ENCRYPTION_KEY`**: a long random string. n8n uses it to encrypt saved credentials.
  **Store it somewhere safe.** If you lose it, a restored database's credentials can't be decrypted.
- **`N8N_HOST`**: the public hostname you'll use, e.g. `n8n.example.com`.

Generate random values with:

```bash
openssl rand -base64 32
```

### Start it

```bash
docker compose up -d
docker compose ps
```

Postgres should show `healthy`. n8n waits for it before starting.

### What the template does for you

- Pinned image versions
- `restart: unless-stopped` on every service
- A Postgres healthcheck; n8n starts only once the database is healthy
- One bridge network per project, named `<PROJECT>-network`
- **No public ports.** For local access, n8n and the website are published on `127.0.0.1` only.

### Open it locally

- n8n: `http://127.0.0.1:<N8N_LOCAL_PORT>`; create the owner account on first visit
- Website: `http://127.0.0.1:<WEB_LOCAL_PORT>`

Give each project different local ports. Two stacks can't share one.

### Adding another project

Copy the template again into a new folder, set a new `PROJECT` name and new local ports, and
`docker compose up -d`. That's it: one stack per project.

---

## 4. Cloudflare Tunnel

The tunnel makes your sites reachable on the internet **without opening any port on your router**.
cloudflared connects outbound to Cloudflare, and Cloudflare sends traffic back through that
connection.

> **Learning or in-house only?** Skip this section. Everything still works on your local network.

### Prerequisites

- A domain whose DNS is managed by Cloudflare (the free plan is enough).

### Create the tunnel

1. In the Cloudflare **Zero Trust** dashboard, go to Networks → Tunnels and create a
   **cloudflared** tunnel.
2. Copy the **tunnel token** it shows you.
3. Set up the tunnel stack:
   ```bash
   mkdir -p ~/infrastructure
   cp -R tunnel ~/infrastructure/tunnel
   cd ~/infrastructure/tunnel
   cp .env.example .env
   open -e .env        # paste the token into TUNNEL_TOKEN
   ```
4. Edit `docker-compose.yml` so cloudflared joins **each project's network** (the template shows
   how). The project stacks must already be running so their networks exist.
5. Start it:
   ```bash
   docker compose up -d
   docker compose logs -f     # look for "Registered tunnel connection"
   ```

### Route a hostname to a container

In the tunnel's settings, add a public hostname route for each service:

| Public hostname | Service |
|---|---|
| `example.com` | `http://myproject-website:80` |
| `n8n.example.com` | `http://myproject-n8n:5678` |
| `pgadmin.example.com` | `http://myproject-pgadmin:80` |

Two rules matter:

- Use **`http://`**, not `https://`. Cloudflare handles HTTPS at its edge; the hop inside your Mac
  is plain HTTP. `https://` here gives a bad gateway.
- Use the **container name**, not `localhost` or a host port. cloudflared reaches containers
  directly over the shared Docker network.

**Before you save the n8n and pgAdmin routes, do section 5.**

### When you add a new project later

Add the new project's network to the tunnel's `docker-compose.yml`, then recreate cloudflared:

```bash
cd ~/infrastructure/tunnel
docker compose up -d --force-recreate
```

All your sites blip for a few seconds while it reconnects.

---

## 5. Locking down admin pages

n8n's editor and pgAdmin are admin tools. Never put them on the internet without a login in front.

### Cloudflare Access (one-time PIN)

In Zero Trust → Access, create an **application** for each admin hostname
(`n8n.example.com`, `pgadmin.example.com`) with a policy that allows only your email address,
using a **one-time PIN**.

Visitors now have to enter a code sent to your email before Cloudflare lets them through.

### Let webhooks through

n8n webhooks are called by apps, payment providers and other services that can't log in.
Create a **second** Access application scoped to the path `n8n.example.com/webhook` with a
**Bypass** policy. Create it **before** you turn on Access for the main n8n hostname, or your
webhooks stop working.

### pgAdmin in server mode

The template runs pgAdmin in **server mode**, which requires its own login. Keep it that way. In
desktop mode, pgAdmin has no login at all.

### Verify, don't assume

After any change, test from a phone on cellular data (off your network):

- `https://n8n.example.com` → should ask for a one-time PIN
- `https://pgadmin.example.com` → should ask for a one-time PIN
- `https://n8n.example.com/webhook/...` → should reach n8n

---

## 6. Backups

There are three kinds of data, and each needs its own backup:

| What | Where it lives | Back it up with |
|---|---|---|
| Code and website files | Your git repo | git |
| **n8n workflows, credentials, and app data** | **Postgres** | **`pg_dump`** |
| n8n encryption key | `.env` / `n8n/` folder | Copy it to your password manager |

**Your automations are not in your repo.** n8n stores workflows and credentials in Postgres.

### Take a backup

```bash
cd ~/projects/myproject
../../path/to/scripts/pg-backup.sh      # or copy the script into the project
```

The script runs `pg_dump` inside the Postgres container and writes a timestamped file to
`backups/`.

Take one **before every upgrade** and **before editing workflows in bulk**.

### Restore

The script's header comments show the restore command. Practice a restore once, into a throwaway
stack, before you need it.

### Time Machine

Time Machine is a good whole-machine safety net. But copying a **running** database's files is not
a reliable database backup. Rely on `pg_dump` for the database.

### Off-site

Keep at least one copy somewhere other than the Mac and the drive sitting next to it.

### Keep the database small

n8n saves every workflow execution. Set execution pruning (the template sets 7 days) or the
execution history will dwarf everything else in your backups.

---

## 7. Local models with Ollama

### Why Ollama runs outside Docker

On a Mac, local models run on the **GPU**, through Metal and Apple's MLX framework. Docker
containers on a Mac **can't reach the GPU**. So Ollama runs natively, and containers call it over
the network.

(Ollama doesn't use the Neural Engine. Only Apple's own frameworks reach that.)

### Install

1. Install **Ollama** for macOS from ollama.com.
2. Pull a small model:
   ```bash
   ollama pull <model-name>
   ollama list
   ```
3. Test it:
   ```bash
   ollama run <model-name> "Say hello in five words."
   ```

### Call it from n8n

Inside a container, the host Mac is reachable as `host.docker.internal`. Point n8n's Ollama
credential (or an HTTP Request node) at:

```
http://host.docker.internal:11434
```

Ollama stays bound to `127.0.0.1`, so it's never exposed on your network.

### Budget your memory

Unified memory is shared by macOS, Docker and your models. Before choosing a model, check what the
stack already uses:

```bash
docker stats --no-stream      # memory per container
ollama ps                     # memory per loaded model
memory_pressure               # overall
```

Every container you add takes memory away from your model. Prefer **small models that do one
specific job well**. If a model takes most of your memory just to load, it isn't sustainable on
that machine.

---

## 8. The native bridge

Docker containers are Linux, even on a Mac, so they can't run Mac apps. But your Swift tools can
run **beside** Docker as small HTTP services, and containers call them just like they call Ollama.

```
n8n (Docker) ──► http://host.docker.internal:8400 ──► your Swift tool (LaunchAgent, 127.0.0.1:8400)
```

This means every Apple framework you already know — AVFoundation, Vision, Core Image, Core ML,
Core Graphics, and more — can be the backend of a website or an automation.

### The example in `native-bridge/`

A minimal Swift command-line tool that:

- listens on `127.0.0.1` with the **Network** framework
- answers `GET /health` with JSON
- requires a token header on every request

### Build and run it

```bash
cd native-bridge/HelloBridge
swift build -c release
.build/release/hellobridge --port 8400
```

Test it from a container:

```bash
docker run --rm curlimages/curl -s \
  -H "X-Bridge-Token: <your-token>" \
  http://host.docker.internal:8400/health
```

### Keep it running with a LaunchAgent

1. Edit the paths in `native-bridge/com.example.hellobridge.plist`.
2. Install it:
   ```bash
   cp native-bridge/com.example.hellobridge.plist ~/Library/LaunchAgents/
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.hellobridge.plist
   launchctl print gui/$(id -u)/com.example.hellobridge | head
   ```
3. After you rebuild, restart it:
   ```bash
   launchctl kickstart -k gui/$(id -u)/com.example.hellobridge
   ```

`KeepAlive` restarts it if it crashes. Because it's a LaunchAgent, it runs in your login session,
so section 1's FileVault choice applies to it too.

### Replace "hello" with your own tool

Keep the shape: listen on loopback, require a token, take JSON in, write files or return JSON.
Everything else is the Swift you already write every day.

---

## 9. Updating safely

1. Back up first (section 6).
2. Change **one** pinned tag in `docker-compose.yml`. Read the release notes for major versions.
3. Pull and recreate:
   ```bash
   docker compose pull <service>
   docker compose up -d --force-recreate <service>
   ```
4. Check the logs and the live site:
   ```bash
   docker compose logs --tail 50 <service>
   ```
5. Keep the old image until you're sure. Clean up unused images later:
   ```bash
   docker image prune
   ```

---

## 10. Gotchas

Every one of these seemed obvious afterward.

| Symptom | Cause | Fix |
|---|---|---|
| A config fix "didn't work" | Editing `docker-compose.yml` doesn't change a running container | `docker compose up -d --force-recreate <service>`, then check the running settings |
| A service quietly jumped a major version | It was on `:latest` | Pin every tag |
| Bad gateway through the tunnel | Route uses `https://` | Use `http://<container>:<port>` |
| Tunnel can't reach a new stack | cloudflared isn't on that stack's network | Add the network, recreate cloudflared |
| `localhost:5678` hit the wrong n8n | Several stacks on one Mac | Address containers by name, and give each stack its own local ports |
| Webhooks stopped after adding Access | Access gates every path | Add a `/webhook` Bypass application first |
| pgAdmin had no login | Desktop mode | Server mode, behind Access |
| Database backups are huge | n8n execution history | Prune executions |
| `docker compose down` fails with "active endpoints" | The tunnel is attached to the network | Recreate instead of tearing down |
| Imported an n8n workflow but the live endpoint runs old logic | Imported isn't the same as published | Publish/activate the new version, then test live |
| Nothing came back after a power cut | FileVault blocked auto-login | See section 1 |
| `NWListener` fails with "invalid argument" | Both a required local endpoint and a port were set | Set one, not both |
