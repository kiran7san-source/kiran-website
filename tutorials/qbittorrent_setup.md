---

# qBittorrent Installation Guide (Docker)

We'll use the **LinuxServer.io** image because it is stable, well documented, and widely used.

## Planned directory structure

Your downloads will be stored on the NVMe SSD:

```text
/srv/share
└── torrents
    ├── downloads
    ├── incomplete
    └── watch
```

Later, when downloads finish, you can either:

* manually move them into `/mnt/media`, or
* configure qBittorrent to move completed downloads automatically.

---

# Step 1: Create the directories

```bash
sudo mkdir -p /srv/share/torrents/downloads
sudo mkdir -p /srv/share/torrents/incomplete
sudo mkdir -p /srv/share/torrents/watch
```

Give yourself ownership:

```bash
sudo chown -R kiran:kiran /srv/share/torrents
```

Verify:

```bash
ls -R /srv/share/torrents
```

---

# Step 2: Create Docker directories

```bash
mkdir -p ~/docker-data/qbittorrent/config
```

---

# Step 3: Create the Docker stack directory

```bash
sudo mkdir -p /opt/stacks/qbittorrent
sudo chown -R kiran:kiran /opt/stacks/qbittorrent

cd /opt/stacks/qbittorrent
```

---

# Step 4: Create `compose.yml`

```bash
nano compose.yml
```

Paste:

```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent

    restart: unless-stopped

    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"

    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Kathmandu
      - WEBUI_PORT=8080

    volumes:
      - /home/kiran/docker-data/qbittorrent/config:/config
      - /srv/share/torrents:/downloads
```

Save and exit.

---

# Step 5: Start qBittorrent

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

You should see:

```
qbittorrent
```

---

# Step 6: Open the Web UI

Open a browser:

```
http://YOUR_SERVER_IP:8080
```

For example:

```
http://192.168.1.50:8080
```

---

# Step 7: Login

Default credentials:

```
Username:
admin

Password:
adminadmin
```

You'll be prompted to change the password immediately. Do so.

---

# Step 8: Configure download locations

Go to:

```
Settings
    Downloads
```

Set:

### Default Save Path

```
/downloads/downloads
```

### Keep incomplete torrents in

```
/downloads/incomplete
```

### Torrent Watch Folder

Enable it and set:

```
/downloads/watch
```

---

# Step 9: Configure Web UI

Go to:

```
Settings
    Web UI
```

Recommended settings:

* Change the username from `admin` if you prefer.
* Set a strong password.
* Enable **"Bypass authentication for clients on localhost"** if you'll access it only through a reverse proxy later (optional).
* Leave CSRF protection enabled.

---

# Step 10: Configure BitTorrent

Go to:

```
Settings
    BitTorrent
```

Recommended:

* ✅ Enable DHT
* ✅ Enable PeX
* ✅ Enable Local Peer Discovery
* ✅ Anonymous Mode: **Off** (unless you specifically need it)

---

# Step 11: Connection settings

Leave the listening port at:

```
6881
```

If your router supports UPnP/NAT-PMP and you want automatic port forwarding, enable it. Otherwise, if you use a VPN or prefer manual forwarding, configure that accordingly.

---

# Step 12: Test

Download a legal test torrent, such as an Ubuntu ISO, to verify everything works.

After it finishes, confirm the file exists on the server:

```bash
ls /srv/share/torrents/downloads
```

And verify you can see it over Samba from your Windows PC or Mac.

---

# Future integration with Jellyfin

One thing I'd change from your current plan is **where completed media lives**.

Instead of leaving movies permanently under `/srv/share/torrents/downloads`, I recommend:

```
/srv/share/torrents/downloads
        │
        ▼
Move completed movies
        │
        ▼
/mnt/media/Movies
```

That keeps:

* qBittorrent downloads separate,
* Jellyfin's library organized,
* and avoids accidentally deleting media when cleaning up torrents.

If you want this to happen automatically, we can configure qBittorrent to move completed downloads into your media library or use tools like Sonarr/Radarr later.

---

## One suggestion

Because your server has an **i5-10500T**, **16 GB RAM**, and you're using Docker, I recommend we next install:

1. qBittorrent (this step)
2. **Tailscale** (secure remote access)
3. **Immich**
4. Optionally **Sonarr**, **Radarr**, and **Prowlarr** for automated media management

This sequence builds a solid and maintainable home server.
