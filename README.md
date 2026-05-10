# Self-Hosted VPS Stack

A modular Docker Compose stack for VPS deployment. Tested on Oracle Cloud ARM64 (free tier).

**What you get:** a private media server (Jellyfin) that streams any content on demand via Real-Debrid, Stremio streaming addons, monitoring tools, and self-hosted utilities — all behind a reverse proxy with automatic TLS and single sign-on.

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

**Decypharr** acts as a fake qBittorrent that Sonarr/Radarr think they're talking to. When they "send a download", Decypharr adds it to Real-Debrid and mounts the content via rclone FUSE at `/mnt/remote/realdebrid/`. Sonarr/Radarr then import it into `/mnt/symlinks/` where Jellyfin picks it up.

---

## Prerequisites

- A VPS — Oracle Cloud ARM64 free tier works well
- A domain name with [Cloudflare](https://cloudflare.com) as your nameserver
- [Docker](https://docs.docker.com/engine/install/) installed on the VPS
- A [Real-Debrid](https://real-debrid.com) subscription (required for the media stack)
- API keys from [TMDB](https://www.themoviedb.org/settings/api) (free)

---

## Setup

### Step 1 — Open firewall ports

**Oracle Cloud users:** ports 80 and 443 are blocked by default at the network level. Traefik needs both open or Let's Encrypt certificate issuance will fail silently and all your services will be unreachable.

In the Oracle Cloud console: **Networking → Virtual Cloud Networks → your VCN → Security Lists → Add Ingress Rules**

| Protocol | Port | Source |
|---|---|---|
| TCP | 80 | 0.0.0.0/0 |
| TCP | 443 | 0.0.0.0/0 |

Also open ports in the OS firewall:
```bash
sudo firewall-cmd --permanent --add-service=http --add-service=https
sudo firewall-cmd --reload
```

### Step 2 — Clone the repo

```bash
sudo git clone https://github.com/YOUR_USERNAME/SelfHosted-Debrid-Media-Stack-Oracle-ARM.git /opt/docker/docker
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

> **Note:** The arr API keys (`YOUR_SONARR_API_KEY`, etc.) can only be filled in after Sonarr/Radarr have started for the first time. Leave them as placeholders for now.

### Step 7 — Update the Honey dashboard domain

`apps/honey/config.json` has the domain hardcoded. Replace every occurrence of `your-domain.com` with your actual domain:
```bash
sudo sed -i 's/your-domain.com/example.com/g' apps/honey/config.json
```

### Step 8 — First boot (core only)

Start just the required core services to verify everything works before enabling the full stack:
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

### Media stack

#### Configure per-service credentials

Some services share credentials that **must match**:

| This value... | ...must equal this value |
|---|---|
| `DEFAULT_PROXY_CREDENTIALS` in `apps/aiostreams/.env` (the password after `:`) | `API_PASSWORD` in `apps/mediaflow-proxy/.env` |
| `MEDIAFUSION_API_PASSWORD` in `apps/aiostreams/.env` | `API_PASSWORD` in `apps/mediafusion/.env` |

Fill in the following:

**`apps/aiostreams/.env`** — set at minimum:
- `SECRET_KEY` — `openssl rand -hex 32`
- `DEFAULT_REALDEBRID_API_KEY` — your Real-Debrid key
- `TMDB_API_KEY` — from [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api)
- `DEFAULT_PROXY_CREDENTIALS` — set to `:yourpassword` (the colon prefix is required; the password must match `API_PASSWORD` in mediaflow-proxy)
- `MEDIAFUSION_API_PASSWORD` — any password (must match `API_PASSWORD` in mediafusion)

**`apps/mediafusion/.env`** — set:
- `SECRET_KEY` — `openssl rand -hex 32`
- `API_PASSWORD` — must match `MEDIAFUSION_API_PASSWORD` in aiostreams
- `TMDB_API_KEY` — same key as above
- `PROWLARR_API_KEY` — get this from Prowlarr UI after first boot (Settings → General)

**`apps/mediaflow-proxy/.env`** — set:
- `API_PASSWORD` — must match the password part of `DEFAULT_PROXY_CREDENTIALS` in aiostreams

**`apps/comet/.env`** — set `COMETNET_API_KEY` to any random string.

**`apps/stremthru/.env`** — set:
- `STREMTHRU_VAULT_SECRET` — `openssl rand -hex 32`
- `STREMTHRU_PROXY_AUTH` — `your-username:yourpassword`
- `STREMTHRU_STORE_AUTH` — replace `YOUR_REALDEBRID_API_KEY` with your key

**`apps/aiometadata/.env`** — set `TMDB_API`, `TVDB_API_KEY`, `FANART_API_KEY`.

#### Enable the media stack

Add the following profiles to `COMPOSE_PROFILES` in `.env`:
```
COMPOSE_PROFILES="required,cloudflare-ddns,warp,honey,jellyfin,seerr,radarr,sonarr4k,prowlarr,bazarr,flaresolverr,decypharr,aiostreams,mediafusion,comet,stremthru,mediaflow-proxy,aiolists,aiometadata,tmdb-addon,tmdb-collections,zilean,recyclarr"
```

```bash
sudo docker compose up -d
sudo docker compose ps  # wait for everything to show healthy
```

#### Post-boot UI configuration

Once containers are healthy, do these one-time steps in each service's UI:

**Prowlarr** (`prowlarr.your-domain.com`):
- Settings → General → copy the API key → paste into `apps/mediafusion/.env` as `PROWLARR_API_KEY`, then `sudo docker compose up -d mediafusion`
- Indexers → Add Indexer → add the indexers you want (Knaben, Zilean torznab, etc.)
- Indexers → Add App → add Sonarr4K and Radarr with their API keys and URLs

**Sonarr4K** (`4k.sonarr.your-domain.com`) and **Radarr** (`radarr.your-domain.com`):
- Settings → General → copy each API key → paste into `apps/decypharr/config.json` (`arrs[].token`), then `sudo docker compose up -d decypharr`
- Settings → Download Clients → Add → qBittorrent → Host: `decypharr`, Port: `8282`
- Settings → Media Management → Root Folders → add `/mnt/symlinks/tv` (Sonarr) or `/mnt/symlinks/movies` (Radarr)

**Seerr** (`seerr.your-domain.com`):
- Connect Radarr and Sonarr4K using their hostnames and API keys
- Set Radarr as the default for movies, Sonarr4K for TV shows

**Recyclarr** (runs automatically on startup/schedule):
- Fill `apps/recyclarr/secrets.yml` with the arr API keys, then `sudo docker compose up -d recyclarr`
- Recyclarr syncs quality profiles and custom formats from TRaSH guides automatically

#### Setting up Stremio streaming

AIOStreams is the single install point for all Stremio addons:

1. Visit `https://aiostreams.your-domain.com` and configure your addons (Real-Debrid key, enable MediaFusion, Comet, StremThru)
2. Install the generated manifest URL into Stremio (desktop or web app)
3. Search for any movie or TV show in Stremio — streams from your Real-Debrid library will appear

### Other tools

Check for any remaining unfilled placeholders before enabling a service:
```bash
grep -r "CHANGEME" apps/
```

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

## Troubleshooting

**TLS certificate fails / services unreachable**
- Oracle Cloud users: check VCN Security List has TCP 80 and 443 open (Step 1). This is the most common issue.
- Check Traefik logs: `sudo docker compose logs -f traefik`

**Container keeps restarting**
- Check its logs: `sudo docker compose logs --tail=50 <service>`
- Check health: `sudo docker compose ps`

**Authelia "email" verification code not received**
- The code is written to a local file, not emailed: `cat apps/authelia/config/notification.txt`

**rclone FUSE mount dies ("Transport endpoint is not connected")**
- This breaks all symlink-based playback. Fix: `sudo docker compose up -d decypharr` — it remounts on startup.

**No search results in Sonarr/Radarr**
- Check Prowlarr has indexers configured and they're synced to the arr instances
- Test indexers directly in Prowlarr (Indexers → Test All)

**Decypharr shows 0 results**
- Real-Debrid must have the torrent cached. Uncached torrents are skipped (`download_uncached: false`)
- Verify via Real-Debrid's cache check: [real-debrid.com/torrents](https://real-debrid.com/torrents)

**AIOStreams streams don't work / credential error**
- Verify `DEFAULT_PROXY_CREDENTIALS` password in `apps/aiostreams/.env` matches `API_PASSWORD` in `apps/mediaflow-proxy/.env`
- Verify `MEDIAFUSION_API_PASSWORD` in aiostreams matches `API_PASSWORD` in mediafusion

---

## Directory Layout

- `apps/*/compose.yaml` — service definitions
- `apps/*/.env` — per-service config
- `data/` — runtime data written by containers (do not edit directly)
- `.env` — domain, credentials, `COMPOSE_PROFILES`, shared hostname variables

---

## Recovery / Fresh Server

Once you've configured everything, push your working copy to a **private** GitHub repo. Recovery from a new server is then two commands.

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
