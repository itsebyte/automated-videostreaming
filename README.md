# Docker Compose Setup for Media Management Tools

This repository contains a `docker-compose.yaml` that sets up a complete, self-hosted media-automation stack: a VPN client (**Gluetun**), download clients (**qBittorrent** for torrents, **SABnzbd** for Usenet), the **Servarr** applications (**Sonarr**, **Radarr**, **Lidarr**, **Prowlarr**, **FlareSolverr**), and **Seerr** as a request front-end.

All download traffic is routed through the VPN, and everything is orchestrated with Docker Compose for easy deployment.

---

## Overview

The stack automatically searches, downloads, and organises media while keeping the download clients behind **AirVPN** (or any other Gluetun-supported provider). qBittorrent and SABnzbd share Gluetun's network namespace, so if the VPN drops, their traffic is cut off by Gluetun's kill-switch (`FIREWALL=on`).

---

## Services Overview

| Service | Default Port | Description |
|---------|--------------|-------------|
| [**Gluetun**](https://github.com/qdm12/gluetun) | – | VPN client. Routes qBittorrent and SABnzbd through your VPN provider. |
| [**qBittorrent**](https://github.com/qbittorrent/qBittorrent) | `8080` | Torrent download client. |
| [**SABnzbd**](https://github.com/sabnzbd/sabnzbd) | `8085` | Usenet (newsgroup) download client. |
| [**Sonarr**](https://github.com/Sonarr/Sonarr) | `8989` | Automated TV-series management. |
| [**Radarr**](https://github.com/Radarr/Radarr) | `7878` | Automated movie management. |
| [**Lidarr**](https://github.com/Lidarr/Lidarr) | `8686` | Automated music management. |
| [**Prowlarr**](https://github.com/Prowlarr/Prowlarr) | `9696` | Indexer manager that syncs indexers to Sonarr/Radarr/Lidarr. |
| [**FlareSolverr**](https://github.com/FlareSolverr/FlareSolverr) | `8191` | Proxy that solves Cloudflare / bot challenges for indexers. |
| [**Seerr**](https://github.com/seerr-team/seerr) | `5055` | Web UI for users to request movies and shows. |

> **Media server:** A media server such as **Jellyfin**, Plex or Emby is expected but **not** part of this stack — point it at the `./media` folders (`series`, `movies`, `music`).

> **Automatic container updates** are not included yet (see [Updating](#updating)). For now, updates are done manually with `docker compose pull && docker compose up -d`.

---

## Prerequisites

- **Docker** and the **Docker Compose** plugin installed
- An **AirVPN** account (or another [Gluetun-supported provider](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers))
- A media server (e.g. **Jellyfin**) to actually play the downloaded media
- A Usenet provider account if you want to use SABnzbd

---

## Installation

Clone the repository (for example into `/opt/`):

```bash
cd /opt
git clone https://github.com/itsebyte/automated-videostreaming
cd automated-videostreaming

# Config directory for Seerr (must be writable by UID/GID 1000)
mkdir -p seerr
chown -R 1000:1000 seerr/
```

---

## Configuration

### 1. Create your `.env` file

The repository ships an `env` template. Docker Compose only auto-loads a file named **`.env`**, so copy it first:

```bash
cp env .env
```

### 2. Fill in your VPN credentials

Edit `.env` and set at least the AirVPN / WireGuard values:

```bash
# --- AirVPN ---
WG_PRIVATE_KEY=<your-wireguard-private-key>
WG_PSK=<your-preshared-key>
WG_ADDRESS=10.5.79.2/32        # full address incl. subnet — not just /32
WG_SERVER_COUNTRY=Netherlands
```

You get these values from AirVPN's **Config Generator** (choose *WireGuard*).

> **Not using AirVPN?** Follow the [Gluetun VPN Provider Setup Guide](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers) and adjust `VPN_SERVICE_PROVIDER` in `docker-compose.yaml` accordingly.

### 3. Ports & timezone (optional)

All WebUI ports and the timezone live in `.env` and can be changed freely:

| Variable | Default | Service |
|----------|---------|---------|
| `QBIT_WEBUI_PORT` | `8080` | qBittorrent |
| `SABNZBD_PORT` | `8085` | SABnzbd |
| `VPN_TORRENT_PORT` | `34753` | Forwarded VPN port for torrents |
| `SONARR_WEBUI_PORT` | `8989` | Sonarr |
| `RADARR_WEBUI_PORT` | `7878` | Radarr |
| `LIDARR_WEBUI_PORT` | `8686` | Lidarr |
| `PROWLARR_WEBUI_PORT` | `9696` | Prowlarr |
| `FLARESOLVER_WEBUI_PORT` | `8191` | FlareSolverr |
| `SEERR_WEBUI_PORT` | `5055` | Seerr |
| `TZ` | `Europe/Amsterdam` | All services |

---

## Start the Stack

```bash
cd /opt/automated-videostreaming
docker compose pull
docker compose up -d
```

Check that everything is healthy (especially that Gluetun connected):

```bash
docker compose ps
docker logs gluetun
```

---

## Post-Setup Configuration

### qBittorrent

- Open qBittorrent at `http://<server-ip>:8080`.
- Retrieve the temporary admin password from the logs:

  ```bash
  docker logs qbittorrent
  ```

- Change it under `WebUI → Authentication → Username/Password`.
- Go to `Settings → Advanced → Network Interface` and set it to `tun0` (the VPN interface), then **Save**. This ensures qBittorrent only uses the VPN tunnel.

---

### SABnzbd

> **Important — port:** SABnzbd defaults to port **8080** internally, which collides with qBittorrent because both share Gluetun's network. On first launch, open `Config → General → Web Server` and set the port to **8085** (matching `SABNZBD_PORT`). After that it is reachable at `http://<server-ip>:8085`.

#### Initial setup

On first launch SABnzbd runs a wizard: pick language/theme and set a WebUI username and password. You can change these later under `Config → General → Web Server`.

#### Retrieve the API key

`Config → General → API Key`. Note the difference:

- **API Key** – used by Sonarr / Radarr / Lidarr
- **NZB Key** – used by some indexers

#### Add your Usenet provider

`Config → Servers → Add Server`:

| Field | Example |
|-------|---------|
| Host | `news.newshosting.com` |
| Port | `563` |
| Username | `yourusername` |
| Password | `yourpassword` |
| Connections | `10` |
| SSL | ✔️ |

Click **Test Server** → *Connection Successful* → **Save Changes**.

---

### Servarr Apps (Sonarr, Radarr, Lidarr, Prowlarr)

- Complete each initial setup wizard.
- Copy each app's API key from `Settings → General → Security → API Key`.
- In **Prowlarr**, add FlareSolverr as a proxy for Cloudflare-protected indexers:
  `Settings → Indexers → + Add Indexer Proxy → FlareSolverr`
  - **Tag:** `proxy`
  - **Host:** `http://flaresolverr:8191/`

More detail: [Servarr Wiki](https://wiki.servarr.com/).

---

## Connecting Download Clients

Because all containers share the Compose network, use the **service name** as the hostname (e.g. `qbittorrent`, `sabnzbd`) — not `localhost` or an IP.

### SABnzbd in Sonarr / Radarr / Lidarr

`Settings → Download Clients → + Add → SABnzbd`

| Field | Value |
|-------|-------|
| Host | `sabnzbd` |
| Port | `8085` |
| API Key | *(from SABnzbd → Config → General)* |
| Category | `tv` (Sonarr) / `movies` (Radarr) / `music` (Lidarr) |

### qBittorrent in Prowlarr / Sonarr / Radarr / Lidarr

`Settings → Download Clients → + Add → qBittorrent`

| Field | Value |
|-------|-------|
| Host | `qbittorrent` |
| Port | `8080` |
| Username / Password | *(your qBittorrent login)* |
| Category | `tv` / `movies` / `music` |

Click **Test** → *Connection Successful* → **Save**.

> Once Prowlarr is linked to Sonarr/Radarr/Lidarr (`Settings → Apps`), it automatically pushes indexers to them, and the download clients above are used for the actual downloads. Make sure all apps share the same `/downloads` volume path.

---

## Updating

Automatic updates are planned but not yet part of this stack. Until then, update manually:

```bash
cd /opt/automated-videostreaming
docker compose pull
docker compose up -d
docker image prune -f   # optional: clean up old images
```

If you want automation, a container such as [Watchtower](https://github.com/containrrr/watchtower) can be added to pull and restart updated images on a schedule.

---

## Troubleshooting

- **Compose won't start / YAML error:** ensure the file begins with a top-level `services:` key.
- **All variables empty / `WARN[…] variable is not set`:** you forgot `cp env .env`.
- **No internet in containers:** check `docker logs gluetun` — a wrong `WG_ADDRESS` (must include the IP, e.g. `10.5.79.2/32`) or key is the usual cause.
- **Can't reach SABnzbd on 8085:** its internal WebUI port is still 8080 — change it in `Config → General → Web Server`.

---

## Contact / Issues

Found a problem or want to contribute? Please open an **Issue** or **Pull Request** on [GitHub](https://github.com/itsebyte/automated-videostreaming).
