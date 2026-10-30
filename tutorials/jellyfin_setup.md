# Production Media Streaming: Jellyfin Media Server Deployment & Hardware Acceleration

## 1. Overview & Architectural Background
Jellyfin is a free, open-source media streaming server designed to manage, organize, and stream personal collections of movies, television shows, music, and live broadcasts across diverse client devices. 

### Architectural Comparison & Key Mechanics
* **FOSS vs. Proprietary Alternatives (Plex / Emby)**: Unlike Plex or Emby, Jellyfin has no premium tiers, paywalled features, third-party authentication dependencies, or remote telemetry tracking. Premium features such as hardware-accelerated transcoding, live TV DVR, and multi-user offline downloads are completely free and open-source.
* **Transcoding Pipeline**: Jellyfin leverages a custom FFmpeg binary (`jellyfin-ffmpeg`) optimized to handle direct stream remuxing and on-the-fly transcoding between incompatible source codecs (e.g., HEVC/H.265, AV1, VC-1, DTS-HD, TrueHD) and client-supported formats (e.g., H.264, AAC, Opus).
* **Hardware Acceleration (HWA)**: Direct GPU acceleration bypasses heavy CPU bottlenecks by routing video decoding, scaling, and encoding through dedicated ASIC silicon:
  * **Intel QuickSync Video (QSV)**: Low-power, high-throughput transcoding via integrated Intel GPUs utilizing the Linux `i915` or `xe` kernel drivers.
  * **NVIDIA NVENC/NVDEC**: High-density hardware transcoding across GeForce and Quadro cards via NVIDIA container toolkit passthrough.
  * **AMD AMF / VA-API**: Open hardware video acceleration via Linux Mesa drivers.
* **Direct Play vs. Transcoding**: The server determines playback topology dynamically based on client codec support, container capabilities, subtitle formats, and network bandwidth. Direct Play preserves 100% original stream quality with negligible host CPU usage (< 1%).

---

## 2. Production Homelab Use Cases
* **Centralized Home Cinema**: Stream 4K HDR/Dolby Vision media to smart TVs (Android TV, Apple TV, LG webOS, Roku) and mobile apps.
* **Remote Traveling Access**: Smoothly downscale high-bitrate remuxes over cellular connections or metered networks.
* **Network-Attached Storage (NAS) Integration**: Serve expansive libraries hosted across external disk pools, Samba/CIFS shares, or NFS exports via persistent mount paths.
* **Multi-User Isolation**: Granular parental controls, viewing restrictions, user profiles, and separate playback history trackers.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12) with Docker and Docker Compose installed.
* Dedicated host storage mount points for media libraries (e.g., `/mnt/storage/media/movies`, `/mnt/storage/media/tv`).
* Linux kernel user added to the `video` and `render` groups for GPU access.
* Accessible reverse proxy (Nginx Proxy Manager, Caddy, or Traefik) terminating with HTTPS for external streaming.

### Directory Layout
```text
/opt/homelab/jellyfin/
├── compose.yaml
├── config/
└── cache/
```

---

## 4. Hardware Acceleration Prerequisites (Intel QuickSync / VA-API)

To pass host GPU capabilities to the Jellyfin container, inspect your host's render devices and group permissions:

```bash
# 1. Verify existence of DRM render nodes
ls -l /dev/dri

# Expected output shows card0 and renderD128:
# crw-rw----+ 1 root video  226,   0 /dev/dri/card0
# crw-rw----+ 1 root render 226, 128 /dev/dri/renderD128

# 2. Add current user to video and render groups
sudo usermod -aG video,render $USER
```

---

## 5. Step-by-Step Implementation

### Step 5.1: Create Persistent Storage Paths
Prepare local persistent state directories:

```bash
sudo mkdir -p /opt/homelab/jellyfin/{config,cache}
sudo chown -R $USER:$USER /opt/homelab/jellyfin
cd /opt/homelab/jellyfin
```

### Step 5.2: Deploy Docker Compose Manifest (`compose.yaml`)
Create `/opt/homelab/jellyfin/compose.yaml`:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    user: "1000:1000" # Replace with your host UID:GID (id -u : id -g)
    network_mode: host # Recommended for DLNA discovery and native network performance
    environment:
      - JELLYFIN_PublishedServerUrl=[https://jellyfin.home.lab](https://jellyfin.home.lab)
    volumes:
      - /opt/homelab/jellyfin/config:/config
      - /opt/homelab/jellyfin/cache:/cache
      # Media Libraries (Adjust paths to match your disk mount points)
      - /mnt/storage/media/movies:/data/movies:ro
      - /mnt/storage/media/tv:/data/tv:ro
      - /mnt/storage/media/music:/data/music:ro
    devices:
      # GPU Passthrough for Intel QuickSync / AMD VA-API
      - /dev/dri/renderD128:/dev/dri/renderD128
      - /dev/dri/card0:/dev/dri/card0
```

> **Note on Network Mode**: Using `network_mode: host` allows DLNA devices and native smart TV auto-discovery to operate without port mapping hurdles. If using a Docker bridge network instead, map port `8096:8096` and UDP port `1900:1900` for DLNA.

Launch the service:

```bash
sudo docker compose up -d
```

---

## 6. Server Configuration & Transcoding Setup

### Step 6.1: Initial Web Wizard
1. Open your browser and navigate to `http://<SERVER_IP>:8096`.
2. Select your preferred display language.
3. Configure the initial administrative user and password.
4. Add your media libraries:
   * **Content type**: `Movies` ➔ Folder: `/data/movies`
   * **Content type**: `Shows` ➔ Folder: `/data/tv`
5. Set metadata downloaders (TheMovieDb, TheTVDB, Open Movie Database).

### Step 6.2: Enable Intel QuickSync Hardware Acceleration
1. Navigate to **Administration Dashboard** ➔ **Playback** ➔ **Transcoding**.
2. Set **Hardware acceleration** to **Intel QuickSync (QSV)** (or **Video Acceleration API (VAAPI)** for AMD systems).
3. Check all hardware decoding formats supported by your CPU generation:
   * H.264, HEVC, HEVC 10-bit, VP9, AV1 (if Intel 11th Gen+ / ARC).
4. Check **Enable Hardware Encoding**.
5. Enable **VPP Tone mapping** and **Tone mapping** to allow correct color gamut translation when transcoding 4K HDR/Dolby Vision files to 1080p SDR displays.
6. Scroll down and click **Save**.

---

## 7. Verification & Health Check

Verify container logs, hardware access, and real-time streaming operations:

```bash
# 1. Verify container runtime health
docker ps --filter "name=jellyfin"

# 2. Inspect active logs for hardware encoding initialization
docker logs --tail 40 jellyfin

# 3. Monitor real-time GPU utilization during playback (Intel GPUs)
sudo apt-get install -y intel-gpu-tools
sudo intel_gpu_top
```

---

## 8. Operational Troubleshooting Matrix

| Symptom / Vector | Root Cause & Resolution |
| :--- | :--- |
| **Playback Error: "Playback failed due to fatal player error"** | Incompatible audio or subtitle stream. Check `/opt/homelab/jellyfin/config/log/FFmpeg.Transcode.*.log` for exact decoder errors. Ensure PGS/ASS subtitle burn-in settings are supported. |
| **Transcoding Stutter / High Host CPU Usage** | Hardware acceleration is falling back to software CPU encoding. Run `ls -l /dev/dri/renderD128` and ensure the container's UID has read/write permissions to the device node. |
| **Permissions Denied on Media Folders** | Media volume is mounted with mismatched permissions. Set read permissions on host storage folders: `chmod -R o+rX /mnt/storage/media`. |
| **Reverse Proxy Buffering / Dropped Frames** | Ensure your reverse proxy buffer sizes are disabled for streaming: set `proxy_buffering off;` and `client_max_body_size 0;` in Nginx. |
