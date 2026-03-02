# Plex

Mac mini home server running a VPN-isolated media automation stack using Docker (Colima).

This document covers:

- **First-time setup on a new system**
- **Daily operation & recovery**
- **Architecture & reference**

Once setup is complete, **all normal usage happens via Overseerr on a phone**.

---

## Quick Start (First-Time Setup)

### Requirements

- macOS (Apple Silicon recommended)
- External drive for media (APFS or HFS+ recommended)
- NordVPN subscription (service credentials)
- Homebrew installed

---

### Install Dependencies

```bash
brew install colima docker docker-compose
```

Start Docker runtime:

```bash
colima start
```

Verify Docker:

```bash
docker info
```

---

### Media Folder Setup

Create media folders on the external drive:

```
/Volumes/storage/Media
├── Movies
├── TV Shows
└── Downloads
    ├── incomplete
    ├── movies
    └── tv
```

Enable ownership on the drive (recommended):

```bash
sudo diskutil enableOwnership /Volumes/storage
```

---

### Docker Stack Setup

Place the provided `docker-compose.yml` in:

```
~/docker/servarr
```

Create the folder if needed:

```bash
mkdir -p ~/docker/servarr
cd ~/docker/servarr
```

Create `.env` file with NordVPN **service credentials**:

```bash
NORD_USER=your_nordvpn_service_username
NORD_PASS=your_nordvpn_service_password
```

---

### First Start

```bash
colima start
cd ~/docker/servarr
docker-compose up -d
```

Verify containers:

```bash
docker-compose ps
```

---

## Initial Service Configuration (One-Time)

Open each service in a browser:

| Service     | URL                   |
| ----------- | --------------------- |
| Overseerr   | http://<host-ip>:5055 |
| Radarr      | http://<host-ip>:7878 |
| Sonarr      | http://<host-ip>:8989 |
| Prowlarr    | http://<host-ip>:9696 |
| qBittorrent | http://<host-ip>:8080 |

---

### qBittorrent Setup

- Set WebUI password
- Enable incomplete downloads:
  - `/downloads/incomplete`
- Create categories:
  - `movies` → `/downloads/movies`
  - `tv` → `/downloads/tv`

---

### Prowlarr (Indexers)

Add indexers

Ensure all indexers test **green**.

Prowlarr automatically syncs indexers to Radarr and Sonarr.

---

### Radarr Setup (Movies)

- Root folder: `/movies`
- Add qBittorrent download client:
  - Host: `gluetun`
  - Port: `8080`
  - Category: `movies`
- Enable upgrades in quality profile

---

### Sonarr Setup (TV)

- Root folder: `/tv`
- Add qBittorrent download client:
  - Host: `gluetun`
  - Port: `8080`
  - Category: `tv`
- Enable upgrades in quality profile

---

### Plex Integration

In **Radarr** and **Sonarr**:

- Settings → Connect → Plex Media Server
- Host: Plex server IP
- Port: `32400`
- Token: Plex API token
- Trigger: **On Import**

This ensures Plex updates automatically when downloads complete.

---

### Overseerr Setup (User Interface)

- Sign in using Plex account
- Connect Radarr and Sonarr
- Enable auto-approval for requests
- Users authenticate via Plex login

Phone usage:

- Open Overseerr in mobile browser
- Add to home screen for app-like UX

---

### Test

Request:

- One movie
- One TV episode

Confirm:

- Download starts
- File moves to Movies / TV Shows
- Plex updates automatically

---

## Architecture Overview

- **Plex** runs natively on macOS (no VPN)
- **Docker stack (Colima)** handles automation and downloads
- **Torrent traffic is forced through VPN (NordVPN)**
- **Requests are done via phone (Overseerr)**

Flow:

```
Phone → Overseerr → Radarr / Sonarr → qBittorrent (VPN) → Media folders → Plex
```

---

## Media Folders

```
/Volumes/storage/Media
├── Movies
├── TV Shows
└── Downloads
    ├── incomplete
    ├── movies
    └── tv
```

- **Movies / TV Shows** → Plex libraries
- **Downloads** → Temporary staging (not scanned by Plex)

---

## Docker Stack Location

```
~/docker/servarr
```

Start stack (manual):

```bash
colima start
cd ~/docker/servarr
docker-compose up -d
```

Stop stack:

```bash
docker-compose down
```

---

## Running Services

| Service      | Purpose                        | URL / Port             |
| ------------ | ------------------------------ | ---------------------- |
| Plex         | Media playback                 | http://<host-ip>:32400 |
| Overseerr    | Phone-friendly request UI      | http://<host-ip>:5055  |
| Radarr       | Movie automation               | http://<host-ip>:7878  |
| Sonarr       | TV automation                  | http://<host-ip>:8989  |
| Prowlarr     | Indexer management             | http://<host-ip>:9696  |
| qBittorrent  | Torrent client (VPN only)      | http://<host-ip>:8080  |
| Gluetun      | VPN container (NordVPN)        | internal               |
| FlareSolverr | Cloudflare solver for trackers | internal (8191)        |

---

## VPN Setup

- VPN provider: **NordVPN**
- VPN runs **only inside Docker**
- qBittorrent shares network with Gluetun
- Plex traffic is **not** routed through VPN

Verification:

```bash
docker exec -it gluetun wget -qO- https://ipinfo.io/ip
docker exec -it qbittorrent wget -qO- https://ipinfo.io/ip
curl https://ipinfo.io/ip
```

- Gluetun + qBittorrent → VPN IP
- Mac host → ISP IP

---

## Categories (qBittorrent)

- `movies` → `/downloads/movies`
- `tv` → `/downloads/tv`

Radarr uses category `movies`  
Sonarr uses category `tv`

---

## Recovery Checklist (after crash/reboot)

1. `colima start`
2. `cd ~/docker/servarr`
3. `docker-compose up -d`
4. Verify:
   - Overseerr loads
   - qBittorrent reachable
   - Gluetun status healthy
