# Production Personal Music Streaming: Navidrome Music Server Deployment

## 1. Overview & Architectural Background
Navidrome is an open-source, lightweight streaming music server and audio metadata indexer written in Go. It allows users to self-host and stream personal music libraries across any browser or native mobile client without subscription paywalls or cloud vendor lock-in.

### Architectural Breakdown & Core Mechanics
* **Subsonic / OpenSubsonic API Standard**: Rather than forcing users into a proprietary client ecosystem, Navidrome natively implements the open Subsonic/OpenSubsonic API standard. This makes it instantly compatible with dozens of third-party clients across iOS (Amperfy, play:Sub), Android (Symfonium, DSub), desktop (Feishin, Substreamer), and CLI players.
* **Go + SQLite Performance Profile**: Unlike legacy Java-based music servers (e.g., Airsonic, Madsonic) that demand heavy JVM heaps and significant idle CPU, Navidrome compiles to a compact Go binary with an embedded SQLite engine. It indexes hundreds of thousands of audio tracks with negligible memory usage (often under 50 MB RAM).
* **Direct File Tag Extraction**: Navidrome does not modify or alter the host filesystem or write metadata directly to audio files. It reads embedded ID3, Vorbis, and FLAC tags using an internal parser, populating an indexed database cache for instantaneous search and playlist generation.
* **On-the-Fly Dynamic Transcoding**: Through seamless FFmpeg integration, Navidrome dynamically transcodes lossless audio formats (FLAC, ALAC, WAV) into bandwidth-efficient codecs (Opus, MP3) when bandwidth constraints or client hardware limitations require downsampling.

---

## 2. Production Homelab Use Cases
* **Lossless Audiophile Streaming**: Stream bit-perfect FLAC/ALAC collections locally and remotely across mobile devices and home DACs.
* **Low-Bandwidth Mobile Playback**: Automatically downsample high-resolution audio to modern, bandwidth-efficient Opus streams when connected over cellular networks.
* **Offline Caching & Smart Playlists**: Sync playlists, favorites, and albums for offline listening via Subsonic mobile clients.
* **Multi-User Library Isolation**: Maintain separate listening stats, favorite tracks, playback queues, and playlists for different family members on a single shared music repository.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12) with Docker and Docker Compose installed.
* Dedicated music directory containing tagged audio files (e.g., `/mnt/storage/media/music`).
* Reverse proxy (Nginx Proxy Manager, Caddy, or Traefik) terminating with SSL.
* Host domain or local FQDN (e.g., `music.home.lab` or `music.kiranpsankhi.com.np`).

### Directory Layout
```text
/opt/homelab/navidrome/
├── compose.yaml
├── .env
└── data/
```

---

## 4. Step-by-Step Implementation

### Step 4.1: Prepare Host Directories
Create the persistent state directory for Navidrome:

```bash
sudo mkdir -p /opt/homelab/navidrome/data
sudo chown -R $USER:$USER /opt/homelab/navidrome
cd /opt/homelab/navidrome
```

### Step 4.2: Configure Environment Variables (`.env`)
Create `/opt/homelab/navidrome/.env` to define operational defaults:

```bash
cat <<EOF> /opt/homelab/navidrome/.env
# Server Ingress & Logging
ND_BASEURL=[https://music.home.lab](https://music.home.lab)
ND_LOGLEVEL=info
ND_SESSIONTIMEOUT=24h

# Library Scanning
ND_SCANINTERVAL=1h
ND_AUTOUPDATEINDEX=true

# Transcoding Options
ND_DEFAULTTRANSCODINGFORMAT=opus
ND_TRANSCODINGCACHESIZE=2GB

# Security & UI
ND_ENABLESHARING=true
ND_ENABLECOVERANIMATION=true
EOF
```

### Step 4.3: Deploy Docker Compose Manifest (`compose.yaml`)
Create `/opt/homelab/navidrome/compose.yaml`:

```yaml
services:
  navidrome:
    image: deluan/navidrome:latest
    container_name: navidrome
    restart: unless-stopped
    user: "1000:1000" # Replace with host UID:GID (id -u : id -g)
    ports:
      - "4533:4533"
    env_file:
      - .env
    volumes:
      - ./data:/data
      - /mnt/storage/media/music:/music:ro # Path to host music library (read-only)
    networks:
      - ingress-net

networks:
  ingress-net:
    external: true
```

*Note: Mounting the host music directory as `:ro` (read-only) ensures that Navidrome cannot accidentally modify, overwrite, or delete source audio assets.*

Launch the container:

```bash
sudo docker compose up -d
```

---

## 5. Reverse Proxy Configuration & Subsonic Ingress

Because mobile clients communicate with Navidrome via HTTP REST calls, standard reverse proxy configurations work seamlessly without custom socket handlers.

### Nginx Proxy Manager Ingress Parameters
1. Create a new **Proxy Host** in NPM:
   * **Domain Names**: `music.home.lab`
   * **Scheme**: `http`
   * **Forward Hostname / IP**: `navidrome` (or your host IP)
   * **Forward Port**: `4533`
   * Check **Block Common Exploits** and **Websockets Support**.
2. Under the **SSL** tab:
   * Select your SSL Wildcard or Domain Certificate.
   * Check **Force SSL** and **HTTP/2 Support**.
3. Save the proxy host.

---

## 6. Initial Configuration & Client Pairing

### Step 6.1: Initial Administrative Account Setup
1. Open your browser and navigate to `https://music.home.lab` (or `http://<SERVER_IP>:4533`).
2. Create your primary administrative username and password.
3. Navidrome will automatically initiate an asynchronous index scan of your mounted `/music` directory.

### Step 6.2: Connecting Mobile & Desktop Clients (Subsonic API)
To stream on mobile devices using applications such as **Symfonium** (Android), **Amperfy** (iOS), or **Feishin** (Desktop):
1. In the client, add a new server and select **Subsonic / OpenSubsonic API**.
2. **Server URL**: `https://music.home.lab`
3. Enter your Navidrome username and password.
4. Save and perform an initial sync to fetch playlists, artists, and covers.

---

## 7. Verification & Operational Matrix

```bash
# 1. Verify container runtime health
docker ps --filter "name=navidrome"

# 2. Check active indexing logs
docker logs --tail 30 navidrome

# 3. Test API ping endpoint directly
curl -kI [https://music.home.lab/ping](https://music.home.lab/ping)
```

### Operational Troubleshooting Matrix

| Symptom / Vector | Root Cause | Remediation Procedure |
| :--- | :--- | :--- |
| **Transcoding Fails / Playback Stalls on Mobile** | FFmpeg binary execution failure or insufficient write space in transcode cache. | Ensure the container user has write permissions to `./data` where the transcode cache directory resides. |
| **New Albums Not Appearing in Library** | Scan interval has not elapsed or embedded tags are malformed. | Click **Activity** ➔ **Quick Scan** in the web UI, or run `docker exec -it navidrome kill -s USR1 1` to trigger an immediate rescan. |
| **Cover Art Missing** | Embedded cover images missing or album art file is named abnormally. | Ensure cover images inside music directories are named standardly: `cover.jpg`, `folder.jpg`, or `front.jpg`. |
| **`Permission Denied` Reading Music Files** | Host filesystem permissions restrict user ID `1000`. | Run `sudo chmod -R o+rX /mnt/storage/media/music` on the host to ensure all audio directories permit read traversal. |
