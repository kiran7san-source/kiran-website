Since I know your current setup, I'll tailor this guide to your server:

* **OS:** Ubuntu Server
* **Docker & Docker Compose:** Already installed
* **Media drive:** `/mnt/media`
* **Music library:** `/mnt/media/Music`
* **Existing containers:** Jellyfin, Immich, qBittorrent, Portainer
* **Goal:** Run Navidrome in Docker with your music stored on the host.

---

# Step 1: Create Navidrome folders

On your server, run:

```bash
sudo mkdir -p /opt/navidrome
sudo mkdir -p /mnt/media/Music
sudo mkdir -p /mnt/media/NavidromeData
```

Set ownership:

```bash
sudo chown -R $USER:$USER /opt/navidrome
sudo chown -R $USER:$USER /mnt/media/NavidromeData
```

---

# Step 2: Create a Docker Compose file

```bash
cd /opt/navidrome
nano docker-compose.yml
```

Paste:

```yaml
services:
  navidrome:
    image: deluan/navidrome:latest
    container_name: navidrome
    restart: unless-stopped

    ports:
      - "4533:4533"

    environment:
      ND_SCANSCHEDULE: "1h"
      ND_LOGLEVEL: "info"
      ND_SESSIONTIMEOUT: "24h"

      # Optional
      ND_LASTFM_ENABLED: "false"

    volumes:
      - /mnt/media/NavidromeData:/data
      - /mnt/media/Music:/music:ro
```

Save.

---

# Step 3: Start Navidrome

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

You should see:

```
navidrome
```

---

# Step 4: Open Navidrome

Open your browser:

```
http://YOUR_SERVER_IP:4533
```

Example:

```
http://192.168.1.100:4533
```

The first visit will ask you to create the administrator account.

Choose:

* Username
* Password
* Email (optional)

---

# Step 5: Import music

If your music is already in

```
/mnt/media/Music
```

Navidrome will automatically begin scanning.

You can watch progress under

```
Settings → Activity
```

or from the terminal:

```bash
docker logs -f navidrome
```

You'll see messages like:

```
Scanning library...
```

and

```
Imported 3000 tracks
```

---

# Step 6: Organize later

After you finish cleaning your music with MusicBrainz Picard, simply replace the contents of

```
/mnt/media/Music
```

Navidrome detects changes automatically every hour (because of `ND_SCANSCHEDULE: "1h"`).

To force an immediate rescan:

```
Settings
→ Library
→ Scan Library
```

---

# Step 7: Mobile apps

Because Navidrome implements the Subsonic API, many excellent clients work with it.

### Android

I recommend **Symfonium**.

Server:

```
http://YOUR_SERVER_IP:4533
```

Username:

```
your username
```

Password:

```
your password
```

---

# Step 8: Backup

Only two locations need backing up:

```
/mnt/media/Music
```

and

```
/mnt/media/NavidromeData
```

The data folder stores:

* users
* playlists
* favorites
* listening history
* settings

---

# Step 9: Reverse proxy (optional)

If you eventually expose services outside your home network using Nginx Proxy Manager or Caddy, you can publish Navidrome as:

```
music.yourdomain.com
```

without changing the Docker configuration.

---

# Step 10: Update

Updating Navidrome is straightforward:

```bash
cd /opt/navidrome
docker compose pull
docker compose up -d
```

---

## Recommended configuration for your server

I suggest slightly expanding the Compose file with a few quality-of-life settings:

```yaml
services:
  navidrome:
    image: deluan/navidrome:latest
    container_name: navidrome
    restart: unless-stopped

    ports:
      - "4533:4533"

    environment:
      ND_SCANSCHEDULE: "1h"
      ND_LOGLEVEL: "info"
      ND_SESSIONTIMEOUT: "24h"
      ND_BASEURL: ""
      ND_ENABLESHARING: "true"
      ND_ENABLEDOWNLOADS: "true"
      ND_LASTFM_ENABLED: "false"
      ND_DEFAULTTHEME: "Auto"

    volumes:
      - /mnt/media/NavidromeData:/data
      - /mnt/media/Music:/music:ro
```

## One improvement I'd make to your overall setup

Earlier, your Docker stacks were split across different directories. To keep things organized as your server grows, I'd recommend a structure like this:

```text
/opt/
├── immich/
├── jellyfin/
├── navidrome/
├── qbittorrent/
└── portainer/

/mnt/media/
├── Movies/
├── TV Shows/
├── Music/
├── MusicVideos/
└── Audiobooks/

/mnt/media/NavidromeData/
```

This keeps all Docker Compose files together under `/opt`, while all media remains on your large storage drive. It also makes backups and migrations much simpler.
