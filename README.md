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

Services are toggled via `COMPOSE_PROFILES` in `.env`. Only `required` (Traefik + Authelia) runs by default — add others as you configure them.

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

## Prerequisites

Before starting, you need:

- A VPS — Oracle Cloud ARM64 free tier works well
- A domain name with [Cloudflare](https://cloudflare.com) as nameserver
- [Docker](https://docs.docker.com/engine/install/) installed on the VPS
- A [Real-Debrid](https://real-debrid.com) subscription (for the media stack)

---

## Setup

### Step 1 — Open firewall ports

**Oracle Cloud users:** ports 80 and 443 are blocked by default in the VCN Security List. Traefik needs both open or Let's Encrypt certificate issuance will silently fail.

In the Oracle Cloud console: **Networking → Virtual Cloud Networks → your VCN → Security Lists → Add Ingress Rules**

| Protocol | Port | Source |
|---|---|---|
| TCP | 80 | 0.0.0.0/0 |
| TCP | 443 | 0.0.0.0/0 |

Also ensure your VPS OS firewall allows them:
```bash
sudo firewall-cmd --permanent --add-service=http --add-service=https
sudo firewall-cmd --reload
```

### Step 2 — Clone the repo

```bash
sudo git clone https://github.com/YOUR_USERNAME/docker-stack-public.git /opt/docker/docker
cd /opt/docker/docker
```

> **Note:** All commands below must be run from `/opt/docker/docker` with `sudo` — the Docker socket is root-owned.

### Step 3 — Create required mount points

```bash
sudo mkdir -p /mnt/symlinks/{movies,movies4k,tv,anime-tv} /mnt/remote/realdebrid
```

### Step 4 — Fill in the root `.env`

```bash
sudo nano .env
```

Required values:

| Variable | Description |
|---|---|
| `DOMAIN` | Your domain (e.g. `example.com`) — all subdomains derive from this |
| `LETSENCRYPT_EMAIL` | Email for Let's Encrypt notifications |
| `PUID` / `PGID` | Your Linux user/group ID — run `id` to check (Oracle default: `1001`) |
| `TZ` | Your timezone — e.g. `Europe/London`, `America/New_York` |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token with **Edit zone DNS** permission |
| `AUTHELIA_SESSION_SECRET` | Random string: `openssl rand -hex 32` |
| `AUTHELIA_STORAGE_ENCRYPTION_KEY` | Random string: `openssl rand -hex 32` (different from above) |
| `AUTHELIA_JWT_SECRET` | Random string: `openssl rand -hex 32` (different from both above) |

Leave `COMPOSE_PROFILES="required"` for now — you'll add services after they're configured.

### Step 5 — Create your Authelia user

Generate a bcrypt hash of your password:
```bash
sudo docker run --rm authelia/authelia:latest authelia crypto hash generate bcrypt --password 'YourPassword'
```

Edit `apps/authelia/config/users.yml` and replace the placeholder values with your username, display name, email, and the hash output.

### Step 6 — Configure Decypharr

Edit `apps/decypharr/config.json`:
- Set `debrids[0].api_key` to your Real-Debrid API key (from [real-debrid.com/apitoken](https://real-debrid.com/apitoken))
- Set `rclone.uid` and `rclone.gid` to match your `PUID` / `PGID`

### Step 7 — Update the Honey dashboard domain

`apps/honey/config.json` has the domain hardcoded (it doesn't support env var substitution). Replace every occurrence of `your-domain.com` with your actual domain:
```bash
sudo sed -i 's/your-domain.com/example.com/g' apps/honey/config.json
```

### Step 8 — First boot (core only)

Start just the required core services to verify everything works:
```bash
sudo docker compose up -d
sudo docker compose ps
```

Wait a minute, then check that Traefik and Authelia are healthy. Visit `https://auth.your-domain.com` — it should load the Authelia login page with a valid TLS certificate.

**First login:** Authelia will ask for a verification code it "emailed" you. It's actually written to a local file:
```bash
cat apps/authelia/config/notification.txt
```

---

## Adding Services

Once core is working, enable services one category at a time.

### Media stack

Fill in API keys for each service:

**`apps/aiostreams/.env`** — set at minimum:
- `DEFAULT_REALDEBRID_API_KEY` — your Real-Debrid key
- `TMDB_API_KEY` and `TMDB_ACCESS_TOKEN` — from [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api)
- `SECRET_KEY` — `openssl rand -hex 32`
- `DEFAULT_PROXY_CREDENTIALS` — set to `:yourpassword` (colon prefix is required)
- `MEDIAFUSION_API_PASSWORD` — must match `API_PASSWORD` in `apps/mediafusion/.env`

**`apps/mediafusion/.env`** — set:
- `SECRET_KEY` — `openssl rand -hex 32`
- `API_PASSWORD` — must match `MEDIAFUSION_API_PASSWORD` in aiostreams
- `TMDB_API_KEY` — same key as above
- `PROWLARR_API_KEY` — available from Prowlarr UI after first boot (Settings → General)

**`apps/mediaflow-proxy/.env`** — set:
- `API_PASSWORD` — must match the password part of `DEFAULT_PROXY_CREDENTIALS` in aiostreams

**`apps/comet/.env`** — set `COMETNET_API_KEY` to any random string.

**`apps/stremthru/.env`** — set:
- `STREMTHRU_VAULT_SECRET` — `openssl rand -hex 32`
- `STREMTHRU_PROXY_AUTH` — `your-username:yourpassword`
- `STREMTHRU_STORE_AUTH` — replace `YOUR_REALDEBRID_API_KEY` with your key

**`apps/aiometadata/.env`** — set `TMDB_API`, `TVDB_API_KEY`, `FANART_API_KEY`.

Then enable the media profile group in `.env`:
```
COMPOSE_PROFILES="required,cloudflare-ddns,warp,honey,jellyfin,seerr,radarr,sonarr,prowlarr,bazarr,flaresolverr,decypharr,aiostreams,mediafusion,comet,stremthru,mediaflow-proxy,aiolists,aiometadata,tmdb-addon,tmdb-collections,zilean,recyclarr"
```

```bash
sudo docker compose up -d
```

**After arrs are running**, get their API keys from each UI (Settings → General → API Key) and fill in `apps/recyclarr/secrets.yml`, then restart recyclarr:
```bash
sudo docker compose up -d recyclarr
```

### Other tools

Use `grep -r "CHANGEME" apps/` to find any remaining unfilled placeholders before enabling a service.

Enable individual services by adding their profile name to `COMPOSE_PROFILES` and running `sudo docker compose up -d`.

---

## Common Commands

All commands run from `/opt/docker/docker` with `sudo`:

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

- `apps/*/compose.yaml` — service definitions
- `apps/*/.env` — per-service config
- `data/` — runtime data written by containers (do not edit directly)
- `.env` — domain, credentials, `COMPOSE_PROFILES`, shared hostname variables

---

## Recovery / Fresh Server

Once you've configured everything, push your working copy to a **private** GitHub repo so you can recover from any server loss with two commands.

**Set up SSH access to your private repo:**
```bash
ssh-keygen -t ed25519 -C "oracle-vps" -f ~/.ssh/github -N ""
cat ~/.ssh/github.pub
# Add to your private GitHub repo → Settings → Deploy keys → Add deploy key (allow write access)
ssh-keyscan github.com >> ~/.ssh/known_hosts
cat >> ~/.ssh/config << 'EOF'
Host github.com
  IdentityFile ~/.ssh/github
  User git
EOF
```

**Recovery (Docker already installed on new server):**
```bash
git clone git@github.com:your-username/your-private-stack.git /opt/docker/docker
sudo mkdir -p /mnt/symlinks/{movies,movies4k,tv,anime-tv} /mnt/remote/realdebrid
cd /opt/docker/docker
sudo docker compose up -d
```

Cloudflare DDNS updates DNS to the new server IP automatically. Traefik fetches new TLS certs automatically. CrowdSec enrollment is machine-specific — re-enrol manually after startup if needed.

---

## Key Config Files

| File | Purpose |
|---|---|
| `.env` | Domain, credentials, active service profiles |
| `apps/decypharr/config.json` | Real-Debrid API key, arr connections, rclone settings |
| `apps/authelia/config/configuration.yml` | SSO access rules |
| `apps/authelia/config/users.yml` | User accounts and password hashes |
| `apps/honey/config.json` | Dashboard links (hardcoded domain — update + `sudo docker compose restart honey` when adding/removing services) |
| `apps/recyclarr/recyclarr.yml` | TRaSH guide quality profile sync |
