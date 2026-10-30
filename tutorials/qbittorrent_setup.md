# Production Headless P2P Infrastructure: Containerized qBittorrent-nox Deployment & Hardening

## 1. Overview & Architectural Background
BitTorrent remains an efficient, decentralized peer-to-peer (P2P) protocol for distributing large datasets, open-source Linux distributions, and multimedia archives. On dedicated servers and homelab hosts, running a desktop graphical torrent client is inefficient and unnecessary.

**qBittorrent-nox** is a lightweight, headless variant of the open-source qBittorrent client written in C++ (Qt6 and `libtorrent-rasterbar`). It strips out all X11/Wayland desktop dependencies while providing a full-featured web-based graphical user interface (WebUI) and a REST API that matches the desktop application feature for feature.

### Architectural Breakdown & Core Mechanics
* **`libtorrent-rasterbar` Engine**: Rather than relying on custom P2P socket handling, qBittorrent is powered by `libtorrent-rasterbar`, a high-performance C++ library offering asynchronous I/O (Boost.Asio), multi-core piece verification, peer-exchange (PEX), distributed hash tables (DHT), and local peer discovery (LSD).
* **Headless Daemon Execution**: The `qbittorrent-nox` binary executes as a continuous daemon with no display server requirements, running efficiently on low-memory micro-instances or background server nodes.
* **Granular Network Interface Binding**: Built-in support for socket binding ensures that P2P traffic strictly routes through a specified network interface (e.g., `tun0`, `wg0`, or a dedicated VPN container gateway), automatically preventing unencrypted data leaks if a secure tunnel drops.
* **REST API & Ecosystem Integration**: Provides a documented HTTP REST API utilized by automated media managers and companion tooling (e.g., Radarr, Sonarr, Lidarr, and automated clean-up daemons) to orchestrate downloads, seed ratios, and file relocation programmatically.

---

## 2. Production Homelab Use Cases
* **Centralized Automated Ingestion**: Serve as the persistent high-throughput ingestion engine for automated homelab media pipelines.
* **Bandwidth & Connection Throttling**: Enforce strict upload and download limits, global peer caps, and scheduled alternative rate limits during peak working hours.
* **Automated Torrent Lifecycle Management**: Implement automated post-download scripts, Category-based subfolder routing, and automated seeding ratio boundaries.
* **Secure Web Administration**: Monitor and control download queues from desktop browsers or dedicated mobile web clients without requiring direct SSH shell access.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12) with Docker and Docker Compose installed.
* Dedicated high-capacity persistent storage mount for downloads (e.g., `/mnt/storage/downloads`).
* Reverse proxy (Nginx Proxy Manager, Caddy, or Traefik) terminating with SSL.
* Internal hostname or local FQDN (e.g., `torrent.home.lab` or `qbittorrent.kiranpsankhi.com.np`).

### Directory Layout
```text
/opt/homelab/qbittorrent/
├── compose.yaml
├── .env
├── config/
└── downloads/ (or pointed to external storage /mnt/storage/downloads)
```

---

## 4. Step-by-Step Implementation

### Step 4.1: Prepare Host Directories
Create the directory structure for configuration states and download volumes:

```bash
sudo mkdir -p /opt/homelab/qbittorrent/config
sudo mkdir -p /mnt/storage/downloads/{complete,incomplete,torrents}
sudo chown -R $USER:$USER /opt/homelab/qbittorrent
cd /opt/homelab/qbittorrent
```

### Step 4.2: Configure Environment Variables (`.env`)
Create `/opt/homelab/qbittorrent/.env`:

```bash
cat <<EOF> /opt/homelab/qbittorrent/.env
# UID/GID Matching Host Non-Root User
PUID=1000
PGID=1000
TZ=UTC

# Port Mapping
WEBUI_PORT=8080
TORRENTING_PORT=6881

# Storage Paths
DOWNLOADS_PATH=/mnt/storage/downloads
EOF
```

### Step 4.3: Deploy Docker Compose Manifest (`compose.yaml`)
Create `/opt/homelab/qbittorrent/compose.yaml`:

```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=${WEBUI_PORT}
      - TORRENTING_PORT=${TORRENTING_PORT}
    volumes:
      - ./config:/config
      - ${DOWNLOADS_PATH}:/downloads
    ports:
      - "${WEBUI_PORT}:8080"
      - "${TORRENTING_PORT}:6881/tcp"
      - "${TORRENTING_PORT}:6881/udp"
    networks:
      - ingress-net

networks:
  ingress-net:
    external: true
```

*Note: The torrenting port (`6881` TCP/UDP) is forwarded to ensure healthy incoming peer connections. If operating behind an external router or firewall, ensure port 6881 is forwarded or routed through a dedicated VPN.*

Launch the container:

```bash
sudo docker compose up -d
```

---

## 5. Security Hardening & WebUI Access

Recent releases of qBittorrent generate a **temporary random administrator password** on initial startup rather than defaulting to static legacy credentials (`admin/adminadmin`).

### Step 5.1: Retrieve the Initial Temporary Password
Inspect the initial container startup logs to retrieve your unique temporary password:

```bash
docker logs qbittorrent 2>&1 | grep "temporary password"
```

Look for the line:
`A temporary password is provided for this session: <RANDOM_STRING>`

### Step 5.2: Change Password & Configure WebUI
1. Navigate to `http://<SERVER_IP>:8080` in your web browser.
2. Log in using username `admin` and the generated temporary password.
3. Open **Tools** ➔ **Options** (gear icon) ➔ **Web UI**:
   * Change your username and set a strong, permanent password.
   * Under **Security**, check **Enable Cross-Site Request Forgery (CSRF) protection**.
   * Under **Security**, check **Enable Clickjacking protection**.
4. Scroll to the bottom and click **Save**.

---

## 6. Reverse Proxy Configuration & Host Header Verification

When accessing qBittorrent behind a reverse proxy (such as Nginx Proxy Manager), qBittorrent enforces strict host header validation to prevent DNS rebinding attacks.

### Step 6.1: Allow Reverse Proxy Host Headers in qBittorrent
1. Inside the WebUI, navigate to **Tools** ➔ **Options** ➔ **Web UI**.
2. Under **Web User Interface**, uncheck **Enable Host header validation** (or add your reverse proxy domain `torrent.home.lab` under **Server domains**).
3. Click **Save**.

### Step 6.2: Nginx Proxy Manager Ingress Parameters
1. In NPM, navigate to **Hosts** ➔ **Proxy Hosts** ➔ **Add Proxy Host**:
   * **Domain Names**: `torrent.home.lab`
   * **Scheme**: `http`
   * **Forward Hostname / IP**: `qbittorrent` (or host IP)
   * **Forward Port**: `8080`
   * Check **Block Common Exploits** and **Websockets Support**.
2. Under the **SSL** tab:
   * Select your SSL certificate.
   * Check **Force SSL** and **HTTP/2 Support**.
3. Under the **Advanced** tab, pass standard proxy headers to ensure the client IP addresses log accurately:

```nginx
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Server $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

---

## 7. Storage Path & Network Optimization

For optimal disk performance and organized libraries, configure qBittorrent’s internal storage hierarchy:

1. In the WebUI, go to **Tools** ➔ **Options** ➔ **Downloads**:
   * **Default Save Path**: `/downloads/complete`
   * Check **Keep incomplete torrents in**: `/downloads/incomplete`
   * Check **Copy .torrent files to**: `/downloads/torrents`
2. Under **Connection**:
   * Set **Peer connection protocol**: `TCP and μTP`
   * Verify **Port used for incoming connections**: `6881`
3. Under **Speed**:
   * Configure global rate limits to ensure torrent throughput does not saturate WAN egress bandwidth needed for other self-hosted services.

---

## 8. Operational Troubleshooting Matrix

| Symptom / Event | Root Cause | Remediation Procedure |
| :--- | :--- | :--- |
| **"Unauthorized" error on WebUI login behind proxy** | Host header validation rejected the reverse proxy domain name. | In `config/qBittorrent/qBittorrent.conf`, set `WebUI\HostHeaderValidation=false` or configure your domain under `WebUI\ServerDomains=torrent.home.lab`, then restart the container. |
| **Torrents stalled with zero upload/download speed** | Incoming port `6881` is blocked by a firewall, or ISP is filtering BitTorrent protocol traffic. | Ensure port `6881` TCP/UDP is forwarded in UFW (`sudo ufw allow 6881`) and router port forwarding rules, or route traffic via a privacy VPN. |
| **"I/O Error: Permission Denied" when downloading** | The host volume mounted at `/downloads` has mismatched POSIX permissions for UID `1000`. | Run `sudo chown -R 1000:1000 /mnt/storage/downloads` on the host to ensure the container daemon can write blocks to disk. |
| **High disk I/O / System sluggishness during downloads** | `libtorrent` cache settings causing excessive direct physical disk writes. | In **Options** ➔ **Advanced**, increase the disk write cache size (e.g., set **Disk cache** to `512 MiB`) and set disk cache expiry to `60` seconds. |
