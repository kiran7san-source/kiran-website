# Production Web Media Ingestion: Headless MeTube & yt-dlp Deployment

## 1. Overview & Architectural Background
Video and audio stream extraction from digital platforms often requires running command-line tools across local workstations. When managing a headless home server or centralized NAS, managing these downloads through scattered terminal commands becomes inefficient and hard to track.

**MeTube** is a self-hosted, web-based graphical interface for **yt-dlp** (the actively maintained, feature-complete fork of `youtube-dl`). It provides a lightweight, asynchronous download server capable of pulling high-resolution video streams, complete playlists, and extracted audio tracks directly onto centralized storage pools.

### Architectural Breakdown & Core Mechanics
* **yt-dlp Extraction Core**: Instead of maintaining fragile custom scraping logic, MeTube embeds `yt-dlp` directly as its extraction engine. This provides broad compatibility across hundreds of streaming sites, dynamic format selection (e.g., best video stream merged with highest bitrate audio via FFmpeg), and automatic geographic cookie passing.
* **Asynchronous Task Queue & WebSockets**: MeTube is written in Python using modern asynchronous frameworks. Web clients interact over persistent WebSockets, providing real-time download progress, throughput metrics, ETA indicators, and status updates without aggressive client polling.
* **Automatic FFmpeg Post-Processing**: Once raw audio and video chunks are downloaded from source CDNs, MeTube automatically runs internal FFmpeg routines to remux formats into standard, widely supported containers (such as `.mp4` or `.mkv`), embed video thumbnails, inject album/creator metadata tags, or extract audio tracks directly into `.mp3` or `.opus` files.
* **Persistent Ingestion Topology**: Downloads write directly to target server mounts or Samba/NFS paths. This allows newly fetched media to be immediately discovered, indexed, and served by streaming platforms like Jellyfin or Navidrome with zero manual file transfers.

---

## 2. Production Homelab Use Cases
* **Centralized Media Archiving**: Archive educational tutorials, webinars, conferences, and documentaries directly to your homelab archive with a single URL paste.
* **Audio-Only Music Extraction**: Convert live acoustic sets, rare audio recordings, and podcast streams into tagged audio files routed straight into Navidrome libraries.
* **Mobile & Multi-Device Drop Queue**: Access a mobile-friendly browser interface to queue multi-gigabyte downloads remotely while traveling without running client-side battery down.
* **Automated Playlist Synchronization**: Ingest entire playlists and channel series sequentially with structured file and folder naming templates.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12) with Docker and Docker Compose installed.
* Dedicated persistent storage mount for ingested media (e.g., `/mnt/storage/downloads/metube` or unified NAS pools).
* Reverse proxy (Nginx Proxy Manager, Caddy, or Traefik) terminating with SSL.
* Host domain or local FQDN (e.g., `metube.home.lab` or `metube.kiranpsankhi.com.np`).

### Directory Layout
```text
/opt/homelab/metube/
├── compose.yaml
├── .env
├── cookies/ (optional for authenticated streaming platforms)
└── downloads/ (or pointed to external storage /mnt/storage/downloads)
```

---

## 4. Step-by-Step Implementation

### Step 4.1: Prepare Host Directories
Create the directory structure for MeTube configurations, cookies, and target download volumes:

```bash
sudo mkdir -p /opt/homelab/metube/cookies
sudo mkdir -p /mnt/storage/downloads/metube
sudo chown -R $USER:$USER /opt/homelab/metube
cd /opt/homelab/metube
```

### Step 4.2: Configure Environment Variables (`.env`)
Create `/opt/homelab/metube/.env` to customize output formats, concurrency limits, and directory sorting:

```bash
cat <<EOF> /opt/homelab/metube/.env
# Application Defaults
UID=1000
GID=1000
DOWNLOAD_DIR=/downloads
AUDIO_DOWNLOAD_DIR=/downloads/audio

# Performance & Concurrency Tuning
MAX_CONCURRENT_DOWNLOADS=3
DELETE_FILE_ON_TRASHCAN=true

# Output File Templating (yt-dlp syntax)
OUTPUT_TEMPLATE=%(title)s.%(ext)s
OUTPUT_TEMPLATE_CHAPTER=%(title)s/%(section_number)03d - %(section_title)s.%(ext)s

# Ingestion Presets
DEFAULT_OPTION_FORMAT=any
EOF
```

### Step 4.3: Deploy Docker Compose Manifest (`compose.yaml`)
Create `/opt/homelab/metube/compose.yaml`:

```yaml
services:
  metube:
    image: ghcr.io/alexta69/metube:latest
    container_name: metube
    restart: unless-stopped
    env_file:
      - .env
    environment:
      - URL_PREFIX=/
    volumes:
      - /mnt/storage/downloads/metube:/downloads
      # Mount browser cookies file if downloading rate-limited or age-restricted content
      - ./cookies:/cookies:ro
    ports:
      - "8081:8081"
    networks:
      - ingress-net

networks:
  ingress-net:
    external: true
```

Launch the container:

```bash
sudo docker compose up -d
```

---

## 5. Reverse Proxy Configuration & WebSocket Ingress

Because MeTube streams live progress bars and socket events back to the client browser, ensure WebSocket connection upgrades are enabled in your reverse proxy.

### Nginx Proxy Manager Ingress Parameters
1. In NPM, navigate to **Hosts** ➔ **Proxy Hosts** ➔ **Add Proxy Host**:
   * **Domain Names**: `metube.home.lab`
   * **Scheme**: `http`
   * **Forward Hostname / IP**: `metube` (or host IP)
   * **Forward Port**: `8081`
   * Check **Block Common Exploits** and **Websockets Support**.
2. Under the **SSL** tab:
   * Select your SSL Wildcard or local certificate.
   * Check **Force SSL** and **HTTP/2 Support**.
3. Under the **Advanced** tab, configure large buffer allowances to prevent socket drops during heavy playlist downloads:

```nginx
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```

---

## 6. Authentication & Rate-Limit Bypass (Using Cookies)

Certain media platforms enforce CAPTCHAs, bot detections, or age verification that prevent unauthenticated instances of `yt-dlp` from fetching media.

To pass your personal session cookies safely into MeTube:
1. Install an open-source browser extension (such as *Get cookies.txt LOCALLY*) in your desktop browser.
2. Navigate to the target platform and export the current session cookies into a standardized Netscape-formatted file named `cookies.txt`.
3. Copy this file into your server's cookie directory:

```bash
sudo cp cookies.txt /opt/homelab/metube/cookies/cookies.txt
sudo chmod 644 /opt/homelab/metube/cookies/cookies.txt
```

4. Add the following environment variable to `/opt/homelab/metube/.env`:

```text
YTDL_OPTIONS={"cookiefile":"/cookies/cookies.txt"}
```

5. Restart the container:

```bash
sudo docker compose up -d
```

---

## 7. Verification & Health Check

Verify container status and review extraction engine logs:

```bash
# 1. Verify container runtime health
docker ps --filter "name=metube"

# 2. Inspect active execution and yt-dlp update logs
docker logs --tail 30 metube

# 3. Test HTTP connectivity
curl -kI [https://metube.home.lab](https://metube.home.lab)
```

Open `https://metube.home.lab` in your browser. Paste any media URL, select your target container format (e.g., `MP4` or `MP3`), and confirm that the download stream finishes and commits files cleanly to `/mnt/storage/downloads/metube`.

---

## 8. Operational Troubleshooting Matrix

| Symptom / Event | Root Cause | Remediation Procedure |
| :--- | :--- | :--- |
| **"Sign in to confirm you’re not a bot" / HTTP 429 Error** | Streaming CDN is rate-limiting or blocking requests originating from data centers or VPS IP blocks. | Export a fresh Netscape `cookies.txt` file from an active, verified browser session and mount it via `YTDL_OPTIONS`. |
| **Download completes but fails during post-processing** | Missing or corrupted FFmpeg binary execution inside the container, or insufficient disk space for temporary remuxing. | Check `docker logs metube` for FFmpeg exit codes; ensure the host filesystem partition has at least twice the storage capacity of the target video file. |
| **Real-time progress bar stuck or disconnected** | Reverse proxy is dropping the WebSocket handshake connection. | Ensure **Websockets Support** is toggled on in Nginx Proxy Manager, or verify `proxy_set_header Upgrade $http_upgrade;` is configured in Nginx. |
| **Permission Denied writing files to storage** | Container process UID/GID does not have write access to host directory. | Run `sudo chown -R 1000:1000 /mnt/storage/downloads/metube` and ensure `UID=1000` / `GID=1000` are exported in `.env`. |
