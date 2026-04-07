# ARR Homelab Stack — Setup Guide

A Docker Compose media automation stack with ProtonVPN routing for all download traffic and Traefik as a reverse proxy with automatic Let's Encrypt SSL via Cloudflare DNS.

## Stack Overview

| Container    | Role                        | Local Fallback URL      | Traefik HTTPS URL                      |
|--------------|-----------------------------|-------------------------|----------------------------------------|
| traefik      | Reverse proxy / SSL         | —                       | https://traefik.home.yourdomain.com    |
| gluetun      | ProtonVPN WireGuard gateway | —                       | —                                      |
| qBittorrent  | Torrent client (via VPN)    | http://localhost:8090   | https://qbit.home.yourdomain.com       |
| SABnzbd      | Usenet client (via VPN)     | http://localhost:8080   | https://sabnzbd.home.yourdomain.com    |
| Prowlarr     | Indexer manager             | http://localhost:9696   | https://prowlarr.home.yourdomain.com   |
| Sonarr       | TV show automation          | http://localhost:8989   | https://sonarr.home.yourdomain.com     |
| Radarr       | Movie automation            | http://localhost:7878   | https://radarr.home.yourdomain.com     |
| Lidarr       | Music automation            | http://localhost:8686   | https://lidarr.home.yourdomain.com     |
| Jellyfin     | Media server                | http://localhost:8096   | https://jellyfin.home.yourdomain.com   |

> Replace `home.yourdomain.com` with your actual `BASE_DOMAIN` from `.env`.

### Network Architecture

- **qBittorrent** and **SABnzbd** share gluetun's network namespace — all their traffic exits through ProtonVPN.
- **Prowlarr, Sonarr, Radarr, Lidarr, Jellyfin** are on a separate `arrnet` bridge — they do NOT go through the VPN.
- ARR apps communicate with download clients via the `gluetun` hostname (e.g., `gluetun:8090` for qBittorrent).
- **Traefik** sits on `arrnet` and handles all inbound HTTPS traffic, issuing certs via Cloudflare DNS-01 — no internet port exposure required.

---

## After Cloning This Repo

Run these commands once after cloning before starting the stack:

```bash
# 1. Copy the environment template and fill in your values
cp .env.example .env

# 2. Create the Let's Encrypt cert storage file (must exist before Traefik starts)
#    It is gitignored because it will contain private certificate data at runtime
touch config/traefik/acme.json       # Linux / macOS / WSL
chmod 600 config/traefik/acme.json   # Linux / macOS / WSL only

# Windows PowerShell equivalent:
New-Item -ItemType File -Force config\traefik\acme.json

# 3. Create the media and download data directories
#    These are gitignored (large runtime data, not source)
$dirs = @(
  "data\torrents\incomplete","data\torrents\complete",
  "data\usenet\incomplete","data\usenet\complete",
  "data\media\tv","data\media\movies","data\media\music"
)
foreach ($d in $dirs) { New-Item -ItemType Directory -Force -Path $d }
```

Then continue with the setup steps below.

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running (WSL2 backend required on Windows)
- A ProtonVPN account (any paid plan)
- A Usenet provider subscription (for SABnzbd)
- A domain name with DNS managed by **Cloudflare** (free plan is fine)

---

## Config Files — What's Pre-Created vs Auto-Generated

Most services write their own config to the `config/` directory on first startup and need no manual files. Two services are exceptions and have pre-seeded config files included in this repo:

| Service      | Config file                                           | Status            | Why                                                                 |
|--------------|-------------------------------------------------------|-------------------|---------------------------------------------------------------------|
| traefik      | `config/traefik/acme.json`                            | ✅ Pre-created     | Must exist (even if empty) before Traefik starts                    |
| qBittorrent  | `config/qbittorrent/qBittorrent/qBittorrent.conf`     | ✅ Pre-created     | v4.6+ rejects Traefik requests unless `HostHeaderValidation=false`  |
| SABnzbd      | `config/sabnzbd/sabnzbd.ini`                          | ✅ Pre-created     | Will 403 all Traefik requests unless `host_whitelist` is cleared    |
| Prowlarr     | `config/prowlarr/`                                    | 🔄 Auto-generated  | No reverse proxy issues; works out of the box                       |
| Sonarr       | `config/sonarr/`                                      | 🔄 Auto-generated  | No reverse proxy issues; works out of the box                       |
| Radarr       | `config/radarr/`                                      | 🔄 Auto-generated  | No reverse proxy issues; works out of the box                       |
| Lidarr       | `config/lidarr/`                                      | 🔄 Auto-generated  | No reverse proxy issues; works out of the box                       |
| Jellyfin     | `config/jellyfin/`                                    | 🔄 Auto-generated  | `JELLYFIN_PublishedServerUrl` env var handles reverse proxy URL     |
| gluetun      | `config/gluetun/`                                     | 🔄 Auto-generated  | All config via environment variables                                |

---

## Step 1 — Get Your ProtonVPN WireGuard Key

1. Log in at **account.proton.me**
2. Navigate to **VPN** in the left sidebar
3. Click **WireGuard configuration** under Downloads
4. Set **Platform** to **Router**
5. Choose your preferred server country (e.g., Netherlands, Switzerland)
6. Click **Create**, then **Download** the `.conf` file
7. Open the `.conf` file in a text editor — it looks like this:

   ```
   [Interface]
   PrivateKey = ABCdef1234yourPrivateKeyHere=
   Address = 10.2.0.2/32
   DNS = 10.2.0.1

   [Peer]
   PublicKey = ...
   AllowedIPs = 0.0.0.0/0
   Endpoint = x.x.x.x:51820
   ```

8. Copy the value after `PrivateKey = ` (the long base64 string)

> **Note:** gluetun fetches ProtonVPN server endpoints automatically. You only need the private key — not the full `.conf` file.

---

## Step 2 — Create a Cloudflare API Token

Traefik uses this token to create temporary DNS TXT records that prove you own the domain — no open ports to the internet needed.

1. Log in at **dash.cloudflare.com**
2. Go to **My Profile → API Tokens** (top-right avatar menu)
3. Click **Create Token**
4. Select the **"Edit zone DNS"** template
5. Under **Zone Resources**, set it to **"Specific zone"** → select your domain
6. Click **Continue to summary → Create Token**
7. Copy the token — you won't be able to see it again

---

## Step 3 — Set Up DNS Records in Cloudflare

Point each subdomain at your homelab server's LAN IP. Traefik handles routing from there.

1. In Cloudflare, go to your domain → **DNS → Records**
2. Add an **A record** for each service:

   | Name                    | Type | Content        | Proxy |
   |-------------------------|------|----------------|-------|
   | `traefik.home`          | A    | `192.168.1.x`  | DNS only (grey cloud) |
   | `qbit.home`             | A    | `192.168.1.x`  | DNS only (grey cloud) |
   | `sabnzbd.home`          | A    | `192.168.1.x`  | DNS only (grey cloud) |
   | `prowlarr.home`         | A    | `192.168.1.x`  | DNS only (grey cloud) |
   | `sonarr.home`           | A    | `192.168.1.x`  | DNS only (grey cloud) |
   | `radarr.home`           | A    | `192.168.1.x`  | DNS only (grey cloud) |
   | `lidarr.home`           | A    | `192.168.1.x`  | DNS only (grey cloud) |
   | `jellyfin.home`         | A    | `192.168.1.x`  | DNS only (grey cloud) |

   Replace `192.168.1.x` with your homelab server's actual LAN IP.
   Replace `home` with whatever prefix matches your `BASE_DOMAIN` (e.g., if `BASE_DOMAIN=home.example.com`, the zone is `example.com` and the name prefix is `*.home`).

   > **Proxy must be OFF (grey cloud).** Since these are LAN IPs, Cloudflare can't proxy them — it's only used for DNS-01 cert validation.

   **Tip:** You can use a single wildcard record instead of individual entries:

   | Name    | Type | Content       | Proxy |
   |---------|------|---------------|-------|
   | `*.home` | A   | `192.168.1.x` | DNS only |

---

## Step 4 — Generate Traefik Dashboard Credentials

The Traefik dashboard is protected by HTTP basic auth. Generate a bcrypt password hash:

**On Linux / macOS / WSL:**
```bash
htpasswd -nb admin yourpassword
```

**On Windows (PowerShell, no extra tools needed):**
```powershell
docker run --rm httpd:alpine htpasswd -nb admin yourpassword
```

The output looks like:
```
admin:$2y$05$abc123xyz...longstring...
```

**Important:** When pasting into `.env`, escape every `$` as `$$`:
```
TRAEFIK_DASHBOARD_AUTH=admin:$$2y$$05$$abc123xyz...
```

---

## Step 5 — Configure `.env`

Open `.env` in the project root and fill in all required values:

```dotenv
# ProtonVPN
WIREGUARD_PRIVATE_KEY=   # from Step 1
VPN_SERVER_COUNTRIES=    # e.g. Netherlands

# Traefik / SSL
BASE_DOMAIN=             # e.g. home.yourdomain.com
ACME_EMAIL=              # your email for Let's Encrypt notices
CF_DNS_API_TOKEN=        # from Step 2
TRAEFIK_DASHBOARD_AUTH=  # from Step 4 (with $$ escaping)

# General
TZ=                      # e.g. America/New_York
```

Full timezone list: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones

---

## Step 6 — Set Permissions on acme.json (Linux hosts only)

Let's Encrypt requires the cert storage file to be readable only by its owner:

```bash
chmod 600 config/traefik/acme.json
```

> On **Windows with Docker Desktop**, this is handled automatically and you can skip this step.

---

## Step 7 — Validate the VPN

Start gluetun on its own first to confirm ProtonVPN connects before bringing up the full stack:

```bash
cd C:\Projects\arrlab
docker compose up gluetun
```

Watch the logs for a line like `Connected to ProtonVPN` or `TUN device tun0 is up`. Once you see the healthcheck pass, press `Ctrl+C`.

If gluetun fails to connect, double-check your `WIREGUARD_PRIVATE_KEY` in `.env`.

---

## Step 8 — Start the Full Stack

```bash
docker compose up -d
```

Check that all containers are running:

```bash
docker compose ps
```

All services should show **Up** or **Up (healthy)**.

**Watch Traefik obtain certificates** (first boot only — takes ~30 seconds):
```bash
docker logs -f traefik
```

Look for lines like `Successfully obtained certificate` or `Certificates obtained`. Once done, all HTTPS URLs will work.

---

## Step 9 — Configure qBittorrent

1. Open **http://localhost:8090** (or `https://qbit.home.yourdomain.com` once certs are ready)
2. The first-run temporary password is printed in the container logs:
   ```bash
   docker logs qbittorrent
   ```
3. Log in, then go to **Tools → Options → Downloads**:
   - **Default Save Path:** `/data/torrents/complete`
   - **Keep incomplete torrents in:** `/data/torrents/incomplete`
   - Enable **Use category-specific save paths**
4. Go to **Tools → Options → Web UI** and set a permanent username and password

---

## Step 10 — Configure SABnzbd

1. Open **http://localhost:8080** (or `https://sabnzbd.home.yourdomain.com`)
2. Complete the setup wizard — add your Usenet provider (server, port, credentials, SSL)
3. Go to **Config → Folders**:
   - **Temporary Download Folder:** `/data/usenet/incomplete`
   - **Completed Download Folder:** `/data/usenet/complete`
4. Note your **API Key** from **Config → General** — you'll need it for Sonarr/Radarr/Lidarr

---

## Step 11 — Configure Prowlarr

1. Open **http://localhost:9696** (or `https://prowlarr.home.yourdomain.com`)
2. Go to **Indexers → Add Indexer** and add your torrent and/or Usenet indexers
3. Go to **Settings → Apps** and connect each ARR app:

   | App      | URL                      |
   |----------|--------------------------|
   | Sonarr   | http://sonarr:8989       |
   | Radarr   | http://radarr:7878       |
   | Lidarr   | http://lidarr:8686       |

   Each app requires its API key, found under **Settings → General** in the respective app.

Prowlarr will automatically sync indexers to all connected apps.

---

## Step 12 — Add Download Clients to ARR Apps

Repeat this in **Sonarr**, **Radarr**, and **Lidarr**.

Go to **Settings → Download Clients → Add (+)**:

### qBittorrent

| Field    | Value                     |
|----------|---------------------------|
| Type     | qBittorrent               |
| Host     | `gluetun`                 |
| Port     | `8090`                    |
| Username | *(your qBit user)*        |
| Password | *(your qBit pass)*        |
| Category | `tv` / `movies` / `music` |

### SABnzbd

| Field   | Value            |
|---------|------------------|
| Type    | SABnzbd          |
| Host    | `gluetun`        |
| Port    | `8080`           |
| API Key | *(from SABnzbd)* |

> **Why `gluetun` as the hostname?** qBittorrent and SABnzbd run inside gluetun's network namespace. From the ARR apps' perspective, the download clients are reachable at the gluetun container's DNS name on those ports.

---

## Step 13 — Configure Root Folders in ARR Apps

In each app, go to **Settings → Media Management → Root Folders** and add:

| App     | Root Folder          |
|---------|----------------------|
| Sonarr  | `/data/media/tv`     |
| Radarr  | `/data/media/movies` |
| Lidarr  | `/data/media/music`  |

> **Hardlinks:** Because all containers mount the same `/data` tree, Sonarr/Radarr/Lidarr can hardlink completed downloads into the media library instead of copying them — zero extra disk space used.

---

## Step 14 — Configure Jellyfin

1. Open **http://localhost:8096** (or `https://jellyfin.home.yourdomain.com`)
2. Complete the setup wizard — create your admin account
3. Add your media libraries:

   | Library Type | Folder               |
   |--------------|----------------------|
   | Movies       | `/data/media/movies` |
   | TV Shows     | `/data/media/tv`     |
   | Music        | `/data/media/music`  |

4. Let Jellyfin scan and match your media metadata

---

## Step 15 — Verify VPN Isolation

Confirm that download client traffic goes through the VPN but ARR apps use your home IP:

```bash
# Should show a ProtonVPN IP address
docker exec -it qbittorrent curl -s https://ipinfo.io/json

# Should show your home/ISP IP address
docker exec -it sonarr curl -s https://ipinfo.io/json
```

---

## Common Commands

```bash
# Start the stack
docker compose up -d

# Stop the stack
docker compose down

# View logs for a specific container
docker logs -f traefik
docker logs -f gluetun
docker logs -f qbittorrent

# Restart a single container
docker compose restart sonarr

# Pull latest images and recreate containers
docker compose pull
docker compose up -d
```

---

## Port Reference

| Service     | Host Port | Purpose                                  |
|-------------|-----------|------------------------------------------|
| Traefik     | 80        | HTTP → redirects to HTTPS                |
| Traefik     | 443       | HTTPS — all proxied services             |
| SABnzbd     | 8080      | Direct access fallback (bypass Traefik)  |
| qBittorrent | 8090      | Direct access fallback (bypass Traefik)  |
| qBit P2P    | 6881      | Torrent traffic (TCP + UDP, always open) |
| Prowlarr    | 9696      | Direct access fallback                   |
| Sonarr      | 8989      | Direct access fallback                   |
| Radarr      | 7878      | Direct access fallback                   |
| Lidarr      | 8686      | Direct access fallback                   |
| Jellyfin    | 8096      | Direct access fallback                   |

---

## Directory Structure

```
C:\Projects\arrlab\
├── docker-compose.yml
├── .env                       ← fill this in before starting
├── SETUP.md                   ← this file
├── config\
│   ├── traefik\
│   │   └── acme.json                    ← pre-created (empty); Traefik populates with certs
│   ├── qbittorrent\
│   │   └── qBittorrent\
│   │       └── qBittorrent.conf         ← pre-created; disables host header validation
│   ├── sabnzbd\
│   │   └── sabnzbd.ini                  ← pre-created; clears host_whitelist for Traefik
│   ├── gluetun\                         ← auto-generated on first run
│   ├── prowlarr\                        ← auto-generated on first run
│   ├── sonarr\                          ← auto-generated on first run
│   ├── radarr\                          ← auto-generated on first run
│   ├── lidarr\                          ← auto-generated on first run
│   └── jellyfin\                        ← auto-generated on first run
└── data\
    ├── torrents\
    │   ├── incomplete\        ← active torrent data
    │   └── complete\          ← finished torrents (hardlink source)
    ├── usenet\
    │   ├── incomplete\        ← active NZB downloads
    │   └── complete\          ← finished NZB downloads (hardlink source)
    └── media\                 ← final media library (Jellyfin reads here)
        ├── tv\
        ├── movies\
        └── music\
```

---

## Troubleshooting

**gluetun won't connect**
- Verify `WIREGUARD_PRIVATE_KEY` in `.env` is correct (no extra spaces or line breaks)
- Make sure Docker Desktop is using the WSL2 backend (required for `/dev/net/tun` on Windows)
- Try a different `VPN_SERVER_COUNTRIES` value

**qBittorrent or SABnzbd web UI not reachable**
- These containers won't start until gluetun passes its healthcheck (`depends_on: service_healthy`)
- Check: `docker compose ps` — gluetun should show `(healthy)`
- Check logs: `docker logs gluetun`

**Traefik isn't issuing certificates**
- Check Traefik logs: `docker logs traefik`
- Verify `CF_DNS_API_TOKEN` has `Zone:DNS:Edit` permission for your domain
- Verify DNS A records in Cloudflare are set correctly and **proxy is OFF** (grey cloud)
- Verify `BASE_DOMAIN` and `ACME_EMAIL` are set in `.env`
- On Linux: verify `config/traefik/acme.json` has permissions `600`

**Browser shows certificate warning**
- Traefik may still be obtaining the cert — wait ~60 seconds after first boot
- Check `docker logs traefik` for any ACME errors
- Confirm your DNS A records resolve to the right IP: `nslookup sonarr.home.yourdomain.com`

**qBittorrent returns 403 "Host header invalid" via Traefik**
- This is a qBittorrent 4.6+ security feature that rejects requests with unexpected Host headers
- The pre-seeded `config/qbittorrent/qBittorrent/qBittorrent.conf` disables this with `HostHeaderValidation=false`
- If you're still seeing the error, verify the file exists and was not overwritten: check that `WebUI\HostHeaderValidation=false` is present
- As a fallback, go to **Tools → Options → Web UI** in qBittorrent and disable "Enable Host header validation"

**SABnzbd returns 403 or redirects to an error page via Traefik**
- SABnzbd's host whitelist is blocking the domain name in the Host header
- The pre-seeded `config/sabnzbd/sabnzbd.ini` clears the whitelist with `host_whitelist =` (empty = allow all)
- If you're still seeing the error, open SABnzbd via direct access at `http://localhost:8080`, go to **Config → General**, and clear the "Host whitelist" field

**ARR apps can't reach download clients**
- Use `gluetun` (not `localhost` or `127.0.0.1`) as the host in download client settings
- Port: qBit = `8090`, SABnzbd = `8080`
- Test: `docker exec -it sonarr curl -s http://gluetun:8090`

**Traefik dashboard shows "404 page not found"**
- Ensure the router rule matches `traefik.home.yourdomain.com` exactly
- Check that `TRAEFIK_DASHBOARD_AUTH` in `.env` has `$$` instead of `$` for the hash

**Jellyfin LAN discovery not working**
- DLNA (port 1900/udp) requires host networking on some systems
- If needed, add `network_mode: host` to the jellyfin service (remove `networks` and `ports` blocks)

**Files are being copied instead of hardlinked**
- Hardlinks only work within the same filesystem/volume
- All ARR apps and download clients must mount the same `data/` volume at the same path (`/data`)
- Verify each container's volume mount in `docker-compose.yml`
