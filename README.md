# Self-Hosted VPS Stack

A modular Docker Compose stack for VPS deployment. Tested on Oracle Cloud ARM64 (free tier). Includes a media stack (Jellyfin + Real-Debrid), Stremio addons, monitoring, and self-hosted tools — all behind Traefik reverse proxy with Authelia SSO.

---

## What's Included

| Category | Services |
|---|---|
| **Core** | Traefik, Authelia, CrowdSec, Cloudflare DDNS, Warp, Watchtower |
| **Media** | Jellyfin, Seerr, Sonarr4K, Radarr, Radarr4K, Prowlarr, Bazarr, Bazarr4K, Decypharr, FlareSolverr, Recyclarr |
| **Stremio** | AIOStreams, MediaFusion, Comet, StremThru, MediaFlow Proxy, AIOLists, AIOMetadata, TMDB Addon, TMDB Collections, Zilean |
| **Monitoring** | Beszel, Dozzle, Uptime Kuma, Portainer, Streamystats |
| **Tools** | Honey (dashboard), Vaultwarden, Yamtrack, SearXNG, AdGuard |

Services are toggled via `COMPOSE_PROFILES` in `.env`. Only `required` (Traefik + Authelia) is active by default.

---

## Architecture

```
Internet → Traefik (reverse proxy, auto TLS via Let's Encrypt)
         → Authelia (SSO forward-auth)
         → Services (all on internal Docker network)
```

All services use `expose:` — reachable only through Traefik. Subdomains are auto-generated from your domain (e.g. `jellyfin.your-domain.com`).

---

## Media Stack

Real-Debrid via Decypharr — no local storage, all content is symlinks over rclone FUSE.

```
Seerr → Radarr/Sonarr4K → Prowlarr → Decypharr → Real-Debrid → rclone FUSE
      → /mnt/symlinks/{movies,movies4k,tv,anime-tv}/ → Jellyfin
```

---

## Setup

### Prerequisites

- A VPS (Oracle Cloud ARM64 free tier works well)
- A domain name with Cloudflare as nameserver
- A [Real-Debrid](https://real-debrid.com) subscription (for the media stack)
- [Docker](https://docs.docker.com/engine/install/) installed on the VPS

### Required mounts

```bash
mkdir -p /mnt/symlinks/{movies,movies4k,tv,anime-tv} /mnt/remote/realdebrid
```

### Step 1 — Fill in the root `.env`

All required variables are marked. The critical ones:

| Variable | Description |
|---|---|
| `DOMAIN` | Your domain (e.g. `example.com`) — all subdomains derive from this |
| `LETSENCRYPT_EMAIL` | Email for Let's Encrypt certificate notifications |
| `PUID` / `PGID` | Linux user/group ID — run `id` to find yours |
| `TZ` | Your timezone (e.g. `Europe/London`) |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token for auto-DDNS |
| `AUTHELIA_SESSION_SECRET` | Random 64-char string — `openssl rand -hex 32` |
| `AUTHELIA_STORAGE_ENCRYPTION_KEY` | Random 64-char string — different from above |
| `AUTHELIA_JWT_SECRET` | Random 64-char string — different from both above |

### Step 2 — Configure required service files

| File | What to set |
|---|---|
| `apps/decypharr/config.json` | Your Real-Debrid API key |
| `apps/authelia/config/users.yml` | Your username and bcrypt password hash |
| `apps/recyclarr/secrets.yml` | Radarr / Sonarr4K API keys |

Generate a bcrypt password hash for Authelia:
```bash
docker run --rm authelia/authelia:latest authelia crypto hash generate bcrypt --password 'YourPassword'
```

### Step 3 — Fill in per-service `.env` files

Search for `CHANGEME` across all env files to find everything that needs a value:
```bash
grep -r "CHANGEME" apps/
```

Key services that need API keys:
- `apps/aiostreams/.env` — Real-Debrid key, TMDB key, service passwords
- `apps/mediafusion/.env` — API password, Prowlarr key
- `apps/mediaflow-proxy/.env` — API password (must match `DEFAULT_PROXY_CREDENTIALS` in aiostreams)
- `apps/comet/.env` — Real-Debrid key
- `apps/stremthru/.env` — vault secret, Real-Debrid key
- `apps/vaultwarden/.env` — admin token

### Step 4 — Update the Honey dashboard

Edit `apps/honey/config.json` and replace `your-domain.com` with your actual domain. This file has no env var substitution — it must be updated manually.

### Step 5 — Enable services

Add service names to `COMPOSE_PROFILES` in `.env`:
```
COMPOSE_PROFILES=required,jellyfin,seerr,radarr,sonarr4k,prowlarr,...
```

### Step 6 — Start

```bash
sudo docker compose up -d
```

Traefik will fetch TLS certificates automatically. First-time Authelia login sends a verification code to `apps/authelia/config/notification.txt`.

---

## Common Commands

```bash
sudo docker compose up -d                              # start/recreate all enabled services
sudo docker compose up -d <service>                    # recreate a single service
sudo docker compose restart <service>                  # restart (does NOT pick up config changes)
sudo docker compose pull && sudo docker compose up -d  # update all images
sudo docker compose logs -f <service>                  # live logs
sudo docker compose ps                                 # check health
```

---

## Directory Layout

- `apps/*/compose.yaml` — service definitions (edit these for structural changes)
- `apps/*/.env` — per-service config
- `data/` — runtime data written by containers (do not edit directly)
- `.env` — domain, credentials, `COMPOSE_PROFILES`, shared hostname variables

---

## Recovery / Fresh Server

If starting fresh on a new server (Docker already installed):

**1. Add your repo to GitHub and set up SSH access**
```bash
ssh-keygen -t ed25519 -C "oracle-vps" -f ~/.ssh/github -N ""
cat ~/.ssh/github.pub
# Add to GitHub repo → Settings → Deploy keys → Add deploy key
ssh-keyscan github.com >> ~/.ssh/known_hosts
cat >> ~/.ssh/config << 'EOF'
Host github.com
  IdentityFile ~/.ssh/github
  User git
EOF
```

**2. Clone and start**
```bash
git clone git@github.com:your-username/your-private-stack.git /opt/docker/docker
mkdir -p /mnt/symlinks/{movies,movies4k,tv,anime-tv} /mnt/remote/realdebrid
cd /opt/docker/docker
sudo docker compose up -d
```

Cloudflare DDNS updates DNS automatically. Traefik fetches new TLS certs automatically. CrowdSec enrollment is machine-specific and must be redone manually.

---

## Key Config Files

| File | Purpose |
|---|---|
| `apps/decypharr/config.json` | RD API key, arr connections, rclone settings |
| `apps/authelia/config/configuration.yml` | SSO access rules |
| `apps/authelia/config/users.yml` | User accounts and password hashes |
| `apps/honey/config.json` | Dashboard links (hardcoded domain — update + restart honey when adding/removing apps) |
| `apps/recyclarr/recyclarr.yml` | TRaSH guide quality profile sync |
