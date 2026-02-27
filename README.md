# Docker Compose Setup for Media Management Tools

This repository contains a `docker-compose.yaml` configuration that sets up a complete media management stack including a VPN client (**Gluetun**), torrent client (**qBittorrent**), and several Servarr applications (**Sonarr**, **Radarr**, **Prowlarr**, **FlareSolverr**), along with **Seerr**, **SABnzbd**, and **Soon** for automatic updates.

---

## Overview

The stack securely manages and downloads media content while maintaining privacy through **AirVPN** (or any other supported VPN provider).  
All services are orchestrated using **Docker Compose** for easy deployment and management.

---

## Services Overview

| Service | Description |
|----------|--------------|
| [**Gluetun**](https://github.com/qdm12/gluetun) | VPN client routing traffic through your chosen VPN provider. |
| [**qBittorrent**](https://github.com/qbittorrent/qBittorrent) | Open-source torrent client for downloading media. |
| [**SABnzbd**](https://github.com/sabnzbd/sabnzbd) | Usenet downloader for binary newsgroups. |
| [**Sonarr**](https://github.com/Sonarr/Sonarr) | TV series management tool that automates episode downloads. |
| [**Radarr**](https://github.com/Radarr/Radarr) | Movie management tool for automatic film downloads. |
| [**FlareSolverr**](https://github.com/FlareSolverr/FlareSolverr) | Service that bypasses bot and Cloudflare protection. |
| [**Prowlarr**](https://github.com/Prowlarr/Prowlarr) | Indexer manager for Sonarr and Radarr. |
| [**Jellyseerr**](https://github.com/seerr-team/seerr) | Web interface for managing requests to Sonarr/Radarr. |
| [**Soon**]() | Automatically updates Docker containers. |

---

## Prerequisites

Make sure you have:

- A running **Jellyfin** instance  
- Installed **Docker** and **Docker Compose**  
- Access to an **AirVPN** (or other) account  

Clone the repository (for example, into `/opt/`):

```bash
cd /opt
git clone https://github.com/itsebyte/automated-videostreaming
cd automated-videostreaming
docker compose pull
```

---

## Configuration

Edit the `.env` file to include your VPN credentials:

```bash
# --- AirVPN ---
WG_PRIVATE_KEY=
WG_PSK=
WG_ADDRESS=10.5.79.2/32
WG_SERVER_COUNTRY=Netherlands
```

**Note:**  
If you’re not using AirVPN, follow the [Gluetun VPN Provider Setup Guide](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers) to configure your preferred provider.

---

## Start the Stack

```bash
cd /opt/automated-videostreaming
docker compose up -d
```

---

## Post-Setup Configuration

### qBittorrent

- Open qBittorrent in your browser (use the port from your Compose file).
- Retrieve the default login credentials:

  ```bash
  docker logs qbittorrent
  ```

- Change them under:  
  `WebUI → Authentication → Username/Password`

- Go to `Settings → Advanced → Network Interface` and set it to `tun0` (the VPN interface).  
- Save all changes.

---

### Servarr Apps (Sonarr, Radarr, Prowlarr, FlareSolverr)

- Complete the initial setup wizards.  
- In Sonarr and Radarr, copy their API keys from  
  `Settings → General → Security → API Key`  
- In Prowlarr, go to  
  `Settings → Indexers → + Add Indexer → FlareSolverr`  

  Example configuration:

  - **Tag (optional):** proxy  
  - **Host:** `http://flaresolverr:8191/`  

Now you can add indexers protected by Cloudflare.  
[Servarr Wiki](https://wiki.servarr.com/)

---

### SABnzbd

- Navigate to SABnzbd in your browser using its configured port (default: `http://<server-ip>:8080`).

#### Initial Setup

On first launch, SABnzbd starts a setup wizard.  
Choose your language and theme.  
When prompted, create a username and password to secure the WebUI.

Example:

```
Username: admin
Password: yourstrongpassword
```

You can later change this under:  
`Config → General → Web Server → Enable access password`

After completing the wizard, you can log in to the web interface via `http://<server-ip>:8080` (or your mapped Docker port).

---

#### Retrieve the API Key

Go to `Config → General` and scroll down to the **API Key** section.

**Note:**

- **API Key** – used for apps like Sonarr/Radarr  
- **NZB Key** – used for indexers  

Usually, you’ll want the main API Key.

---

#### Add Your Usenet Provider

Go to `Config → Servers → Add Server` and enter your provider’s info:

| Field | Example |
|--------|----------|
| **Server name** | Newshosting |
| **Host** | `news.newshosting.com` |
| **Port** | `563` |
| **Username** | `yourusername` |
| **Password** | `yourpassword` |
| **Connections** | `10` |
| **Enable SSL** | ✔️ |

Click **Test Server** → should display “Connection Successful”.  
Then click **Save Changes**.

---

#### Integration with Sonarr / Radarr

In **Sonarr** or **Radarr**, connect SABnzbd as a download client:

- Navigate to: `Settings → Download Clients → + Add → SABnzbd`
- Enter the following fields:

| Field | Value |
|--------|--------|
| **Host** | `http://sabnzbd:8080` |
| **API Key** | *(from SABnzbd → Config → General)* |
| **Category** | `tv` for Sonarr, `movies` for Radarr |

Click **Test** to confirm connectivity, then **Save**.

---

## Integration: qBittorrent with Prowlarr, Sonarr and Radarr

### Step 1: Connect qBittorrent to Prowlarr

1. Open **Prowlarr** in your browser (`http://<server-ip>:9696`).  
2. Go to `Settings → Download Clients → + Add → qBittorrent`
3. Fill in the following fields:

| Field | Example |
|------|-----------|
| **Name** | qBittorrent |
| **Host** | `http://qbittorrent:8080` |
| **Username** | *(your qBittorrent username)* |
| **Password** | *(your qBittorrent password)* |
| **Category** | `tv`, `movies`, `music` |
| **Add Paused** | false |
| **Use SSL** | false |

4. Click **Test** → should display “Connection Successful”.  
5. Click **Save**.

**Tip:** Use `qbittorrent` as hostname when running in Docker, since all containers share the same network.

---

### Step 2: Connect qBittorrent to Sonarr

1. Open **Sonarr** (`http://<server-ip>:8989`).  
2. Go to `Settings → Download Clients → + Add → qBittorrent`
3. Enter:

| Field | Value |
|------|------|
| **Host** | `http://qbittorrent:8080` |
| **Username** | *(your qBittorrent username)* |
| **Password** | *(your qBittorrent password)* |
| **Category** | `tv` |
| **Completed Download Handling** | Enabled |
| **Remove Completed** | Optional |

4. Click **Test** → “Connection Successful”.  
5. Click **Save**.

Ensure that **Sonarr** and **qBittorrent** use the same shared volume path (e.g., `/downloads`).

---

### Step 3: Connect qBittorrent to Radarr

1. Open **Radarr** (`http://<server-ip>:7878`).  
2. Go to `Settings → Download Clients → + Add → qBittorrent`
3. Enter:

| Field | Value |
|------|------|
| **Host** | `http://qbittorrent:8080` |
| **Username** | *(your qBittorrent username)* |
| **Password** | *(your qBittorrent password)* |
| **Category** | `movies` |
| **Completed Download Handling** | Enabled |

4. Click **Test**, then **Save**.

**Note:** Once **Prowlarr** is connected to Sonarr and Radarr, it will automatically sync the indexers, and qBittorrent will serve as the unified download client.

---

## Updating

Soon

---

### Contact / Issues

If you encounter problems or want to contribute improvements, please open an **Issue** or **Pull Request** on [GitHub](https://github.com/itsebyte/automated-videostreaming).

---
