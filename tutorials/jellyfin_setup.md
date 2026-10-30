Since you've already installed Docker and mounted your media share, let's deploy Jellyfin using Docker Compose. This approach makes updates and backups much easier.

## Step 1: Create a directory for the stack

```bash
sudo mkdir -p /opt/stacks/jellyfin
sudo chown -R $USER:$USER /opt/stacks/jellyfin
cd /opt/stacks/jellyfin
```

---

## Step 2: Create directories for Jellyfin configuration

I recommend storing the container configuration on your SSD rather than on the network share.

```bash
mkdir -p ~/docker-data/jellyfin/config
mkdir -p ~/docker-data/jellyfin/cache
```

You should end up with:

```
~/docker-data/
└── jellyfin
    ├── config
    └── cache
```

---

## Step 3: Create the Compose file

Create `compose.yml`:

```bash
nano compose.yml
```

Paste the following:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin

    network_mode: bridge

    ports:
      - "8096:8096"

    volumes:
      - ~/docker-data/jellyfin/config:/config
      - ~/docker-data/jellyfin/cache:/cache
      - /mnt/jelly:/media

    restart: unless-stopped

    environment:
      - TZ=Asia/Kathmandu

    devices:
      - /dev/dri:/dev/dri
```

Save and exit.

---

## Step 4: Start Jellyfin

```bash
docker compose up -d
```

Check that it's running:

```bash
docker ps
```

You should see a container named `jellyfin`.

---

## Step 5: Open the web interface

From a browser on your network:

```
http://<YOUR_SERVER_IP>:8096
```

For example:

```
http://192.168.1.20:8096
```

---

## Step 6: Complete the setup

During the initial setup:

* Create an administrator account.
* Skip metadata settings for now if you like.
* Add a media library.
* Browse to:

```
/media
```

or a subfolder such as:

```
/media/Movies
/media/TV Shows
```

depending on how your SMB share is organized.

---

## Step 7: Verify hardware acceleration

Since you're using an Intel CPU, Jellyfin can use Intel Quick Sync for hardware transcoding.

Check that Ubuntu detects the GPU:

```bash
ls /dev/dri
```

You should see something like:

```
by-path
card0
renderD128
```

Also verify that the container can see it:

```bash
docker exec jellyfin ls /dev/dri
```

If it lists the same devices, you're ready to enable hardware acceleration later in the Jellyfin dashboard.

---

## Step 8: Enable automatic startup

Your Compose file already includes:

```yaml
restart: unless-stopped
```

This ensures Jellyfin starts automatically whenever Docker starts after a reboot.

---

## One improvement I'd recommend

Instead of using `~` in the volume mappings, use absolute paths. Docker Compose expands `~` inconsistently depending on how it's invoked.

For example:

```yaml
volumes:
  - /home/kiran/docker-data/jellyfin/config:/config
  - /home/kiran/docker-data/jellyfin/cache:/cache
  - /mnt/jelly:/media
```

Replace `kiran` with your actual username if different.

### Before proceeding

To help you configure hardware acceleration correctly, could you tell me which CPU your Ubuntu server is running? From our previous discussions I know you've considered several systems (e.g. HP ProDesk G4 with an i5-9500T, newer 12th/14th gen Intel systems), but I want to make sure the Jellyfin configuration matches the machine you're actually using.

Excellent. Your server hardware is actually very well suited for Jellyfin.

### Your hardware

* **CPU:** Intel Core i5-10500T (6 cores / 12 threads)
* **GPU:** Intel UHD Graphics 630 (Comet Lake)
* **RAM:** 16 GB
* **Ubuntu:** 24.04 LTS

This CPU supports **Intel Quick Sync Video (QSV)**, which means Jellyfin can perform hardware-accelerated transcoding. It can easily handle multiple 1080p transcodes and even a few 4K HEVC transcodes depending on bitrate.

## I recommend using this compose file

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin

    restart: unless-stopped

    ports:
      - "8096:8096"

    environment:
      - TZ=Asia/Kathmandu

    volumes:
      - /home/kiran/docker-data/jellyfin/config:/config
      - /home/kiran/docker-data/jellyfin/cache:/cache
      - /mnt/jelly:/media

    devices:
      - /dev/dri:/dev/dri

    group_add:
      - "44"

    security_opt:
      - no-new-privileges:true
```

---

## Before starting the container

Check that the GPU is available:

```bash
ls -l /dev/dri
```

You should see something similar to:

```
card0
renderD128
```

Now check the render group:

```bash
getent group render
```

It will probably show:

```
render:x:109:
```

or another number.

If it is **109** instead of **44**, change the compose file to:

```yaml
group_add:
  - "109"
```

Using the correct group ID is important for hardware transcoding.

---

## Start Jellyfin

```bash
cd /opt/stacks/jellyfin
docker compose up -d
```

Verify:

```bash
docker ps
```

---

## Configure Hardware Acceleration

After opening

```
http://<server-ip>:8096
```

Go to

```
Dashboard
    Playback
        Transcoding
```

Set:

* Hardware acceleration: **Intel Quick Sync (QSV)**
* Enable hardware decoding
* Enable hardware encoding
* Prefer OS native VA-API decoders
* Allow HEVC decoding
* Allow H264 decoding

Save.

---

## Verify it is working

While streaming something that requires transcoding, run:

```bash
docker logs jellyfin
```

or

```bash
intel_gpu_top
```

To install the monitoring tool:

```bash
sudo apt install intel-gpu-tools
```

Then:

```bash
sudo intel_gpu_top
```

If you see activity in the **Video** or **VideoEnhance** engines during playback, Quick Sync is working.

---

# My recommendation for your home server

Based on everything we've discussed over the last few days (Ubuntu Server, Docker, SMB-mounted media, Jellyfin, qBittorrent, and Immich), I would build the server in this order:

1. ✅ Ubuntu Server
2. ✅ Docker & Docker Compose
3. ✅ Mount SMB shares (`/mnt/jelly`, `/mnt/photos`, etc.)
4. ✅ Jellyfin
5. ⏳ qBittorrent
6. ⏳ Immich
7. ⏳ Samba (for sharing local storage)
8. ⏳ Tailscale (remote access)
9. ⏳ Automatic backups
10. ⏳ Docker auto-update (optional)

This order keeps permissions and storage layout straightforward.

Given your **i5-10500T**, **16 GB RAM**, and Intel Quick Sync support, your server has plenty of capacity for your intended workloads. You should be able to run Jellyfin, Immich, qBittorrent, and Samba simultaneously with good performance, especially if your media library is stored on fast local disks or a reliable gigabit network share.


For your **Intel i5-10500T (Comet Lake)** running **Ubuntu Server 24.04** with Jellyfin in Docker, the goal is to use **Intel Quick Sync (QSV)** for both hardware decoding and encoding. Intel recommends QSV on Linux for Broadwell (5th gen) and newer CPUs like yours, while VA-API serves as the underlying interface. ([Jellyfin][1])

Below are the settings I recommend.

---

# Dashboard → Playback → Transcoding

## Hardware Acceleration

### Hardware acceleration

✅ **Intel Quick Sync (QSV)**

Do **not** choose:

* None
* VA-API (unless you're troubleshooting)
* NVIDIA
* AMD

QSV provides the best performance on your CPU. ([Jellyfin][1])

---

### QSV Device

If shown, enter:

```text
/dev/dri/renderD128
```

If this field is absent, Jellyfin has already detected the device automatically.

---

# Hardware Decoding

Under **Enable hardware decoding for**, check **all codecs that your hardware supports**.

For a 10th Gen Intel CPU:

✅ H.264 / AVC

✅ HEVC / H.265

✅ MPEG-2

✅ VC-1

✅ VP8

✅ VP9

❌ AV1

The UHD 630 GPU in your i5-10500T cannot hardware-decode AV1, so leave it unchecked. ([Jellyfin][1])

---

### Prefer OS native decoders

✅ Check

**Prefer OS native DXVA or VA-API hardware decoders**

Even though you're using QSV, on Linux this allows Jellyfin to use the native VA-API decoder underneath, which is the recommended configuration. ([Jellyfin][1])

---

# Hardware Encoding

### Enable hardware encoding

✅ Check

Without this, Jellyfin will decode using the GPU but encode using the CPU.

---

### Allow encoding in HEVC

✅ Check

Allows Jellyfin to create HEVC streams when appropriate.

---

### Allow encoding in AV1

❌ Leave unchecked

Your GPU cannot encode AV1.

---

### Enable low-power encoder

❌ Leave unchecked

The low-power encoder is mainly useful on newer Intel generations or specific platforms. It isn't necessary for your Comet Lake system. ([Jellyfin][1])

---

### Enable low-power HEVC encoder

❌ Leave unchecked

Same reason.

---

# Tone Mapping

If you mainly watch:

* 1080p SDR movies
* 4K SDR movies

then:

❌ Leave all tone mapping options disabled.

---

If you have many **4K HDR** movies and will stream them to SDR devices:

Enable:

✅ Enable tone mapping

Then select:

* Tone mapping algorithm: **BT.2390** (good default)
* Tone mapping mode: **Auto**

You can leave HDR tone mapping disabled until you actually need it, as it adds some processing overhead. ([Jellyfin][2])

---

# Thread Settings

### Transcoding thread count

Leave at:

```text
0
```

This lets Jellyfin decide automatically.

---

### Encoding preset

Leave:

```text
Auto
```

---

# Audio

### Enable VBR audio encoding

✅ Check

Produces good audio quality while keeping file sizes reasonable.

---

# Muxing Queue Size

Leave:

```text
2048
```

This is the default and works well.

---

# Temporary Files

### Transcoding temporary path

If you have a fast SSD (which you do), use one.

For example:

```text
/home/kiran/docker-data/jellyfin/transcodes
```

Create it first:

```bash
mkdir -p /home/kiran/docker-data/jellyfin/transcodes
```

Then add it to your `compose.yml`:

```yaml
volumes:
  - /home/kiran/docker-data/jellyfin/transcodes:/transcodes
```

and set the path in Jellyfin to:

```text
/transcodes
```

This avoids writing temporary transcoding data to your media share and improves performance.

---

# Other Options

### Throttle transcodes

✅ Check

Reduces CPU/GPU usage when the client buffers ahead.

---

### Delete transcode segments

✅ Check

Automatically removes temporary files after playback.

---

### Allow subtitle extraction on the fly

✅ Check

Useful for MKV files with embedded subtitles.

---

# My recommended configuration

| Setting                   | Value                         |
| ------------------------- | ----------------------------- |
| Hardware acceleration     | ✅ Intel Quick Sync            |
| QSV Device                | `/dev/dri/renderD128`         |
| H.264 decoding            | ✅                             |
| HEVC decoding             | ✅                             |
| MPEG-2 decoding           | ✅                             |
| VC-1 decoding             | ✅                             |
| VP8 decoding              | ✅                             |
| VP9 decoding              | ✅                             |
| AV1 decoding              | ❌                             |
| Prefer OS native decoders | ✅                             |
| Hardware encoding         | ✅                             |
| HEVC encoding             | ✅                             |
| AV1 encoding              | ❌                             |
| Low-power encoder         | ❌                             |
| Low-power HEVC encoder    | ❌                             |
| Tone mapping              | ❌ (unless needed for HDR→SDR) |
| Encoding preset           | Auto                          |
| Thread count              | 0                             |
| VBR audio                 | ✅                             |
| Muxing queue              | 2048                          |
| Throttle transcodes       | ✅                             |
| Delete transcode segments | ✅                             |
| Subtitle extraction       | ✅                             |

## Verify hardware transcoding

After saving these settings, play a movie that **forces transcoding** (for example, reduce playback quality from 4K to 1080p). Then run:

```bash
sudo apt install intel-gpu-tools
sudo intel_gpu_top
```

If hardware acceleration is working, you'll see activity in the **Video** and/or **VideoEnhance** engines while the movie is transcoding. This is the most reliable way to confirm that Intel Quick Sync is being used. ([Jellyfin][1])

[1]: https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/?utm_source=chatgpt.com "Intel GPU | Jellyfin"
[2]: https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/?utm_source=chatgpt.com "Hardware Acceleration | Jellyfin"
