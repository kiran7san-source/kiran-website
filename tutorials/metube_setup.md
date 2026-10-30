Since you've standardized your homelab by giving each service its own Docker stack under `/opt/stacks`, we'll follow the same approach for **MeTube**. This will keep it consistent with your Jellyfin, Navidrome, Audiobookshelf, Kavita, BookOrbit, and other services. ([Reddit][1])

## What is MeTube?

MeTube is a self-hosted web interface for **yt-dlp**. It lets you download:

* YouTube videos
* Music
* Playlists
* Videos from hundreds of supported websites

Downloads are saved directly to a folder on your server, making it a perfect companion to your media library.

---

# Step 1: Create the Stack Directory

SSH into your server:

```bash
sudo mkdir -p /opt/stacks/metube
cd /opt/stacks/metube
```

---

# Step 2: Create Download Folder

Since your media drive is mounted at `/mnt/media`, I recommend creating a dedicated downloads directory.

```bash
sudo mkdir -p /mnt/media/downloads
sudo chown -R 1000:1000 /mnt/media/downloads
```

You can later organize it as:

```
/mnt/media/
│
├── Movies
├── TV Shows
├── Music
├── Audiobooks
├── Books
└── Downloads
```

---

# Step 3: Create docker-compose.yml

```bash
nano docker-compose.yml
```

Paste:

```yaml
services:
  metube:
    image: ghcr.io/alexta69/metube:latest
    container_name: metube

    restart: unless-stopped

    ports:
      - "8081:8081"

    environment:
      UID: 1000
      GID: 1000
      TZ: Asia/Kathmandu

    volumes:
      - /mnt/media/downloads:/downloads
```

Save:

```
CTRL+O
ENTER
CTRL+X
```

This mounts your host download directory into the container so downloaded files persist outside the container. ([Reddit][2])

---

# Step 4: Deploy with Portainer

1. Open Portainer.
2. Go to **Stacks**.
3. Click **Add stack**.
4. Name the stack:

```
metube
```

5. Either:

   * Paste the compose file into the web editor, or
   * Load the compose file from `/opt/stacks/metube/docker-compose.yml` if using Git/file-based workflows.
6. Click **Deploy the stack**.

---

# Step 5: Open MeTube

Visit:

```
http://192.168.1.20:8081
```

(Replace `192.168.1.20` with your server IP if it changes.)

You should see the MeTube web interface.

---

# Step 6: Test

Paste a YouTube URL, choose a format, click **Download**, and the file should appear in:

```
/mnt/media/downloads
```

---

# Recommended Enhancements

You can customize MeTube with additional environment variables:

```yaml
environment:
  UID: 1000
  GID: 1000
  TZ: Asia/Kathmandu

  DOWNLOAD_DIR: /downloads
  MAX_CONCURRENT_DOWNLOADS: 2
  CREATE_CUSTOM_DIRS: false
  DEFAULT_THEME: dark
```

These options control the default download location, the number of simultaneous downloads, directory creation behavior, and UI theme. ([Reddit][3])

---

# Optional Reverse Proxy

If you're already using Nginx Proxy Manager, you can expose MeTube at something like:

```
https://metube.yourdomain.com
```

instead of accessing it by port.

---

# Final Folder Structure

Your homelab would then look like:

```
/opt/stacks
├── immich
├── jellyfin
├── navidrome
├── qbittorrent
├── audiobookshelf
├── kavita
├── bookorbit
└── metube
    └── docker-compose.yml

/mnt/media
├── Movies
├── TV Shows
├── Music
├── Books
├── Audiobooks
└── Downloads
```

### Suggested next improvements

Once MeTube is running, I recommend two enhancements that fit well with your existing media server:

1. **Automatic media sorting** (using tools like FileFlows or custom scripts) so downloaded music, videos, and playlists are moved into the appropriate Jellyfin or Navidrome libraries.
2. **Browser integration** using the MeTube browser extension or bookmarklet, allowing you to send videos directly from your browser to your home server with a single click.

[1]: https://www.reddit.com/r/selfhosted/comments/1qyaedu/real_newbie_docker_compose_help_multiple_services/?utm_source=chatgpt.com "Real newbie - docker_compose help: multiple services?"
[2]: https://www.reddit.com/r/truenas/comments/1ofalds?utm_source=chatgpt.com "TrueNas Scale Dockge MeTube"
[3]: https://www.reddit.com/r/selfhosted/comments/1hh0w0f/guide_metube_selfhosted_youtube_downloader/?utm_source=chatgpt.com "[Guide]  MeTube — Self-hosted YouTube downloader"


Perfect — this output tells us exactly what is happening.

Your **MeTube installation is working**, and your yt-dlp version `2026.07.04` is reasonably current. The problem is **YouTube is rate-limiting/blocking the server's requests**:

```text
HTTP Error 429: Too Many Requests
```

and then:

```text
Sign in to confirm you're not a bot
```

The `Missing required Visitor Data` / PO Token warning is also part of YouTube's newer anti-bot system. yt-dlp's current documentation confirms that YouTube is increasingly requiring PO Tokens for some clients and that cookies can be needed when YouTube presents bot verification. ([GitHub][1])

### I recommend we fix this with YouTube cookies first

Don't change your MeTube container yet. We'll first prove that cookies solve the problem.

## 1. Export your YouTube cookies

Since you're using Windows/Edge (based on your browser), I'd recommend using **Firefox if you have it available**. yt-dlp's documentation specifically notes that cookie extraction tends to work better with Firefox than some Chromium setups. ([GitHub][2])

The important thing is that the exported file is in Netscape format and begins with something like:

```text
# Netscape HTTP Cookie File
```

**Important:** Your cookies are effectively authentication credentials. Don't paste the contents of `cookies.txt` into this chat or upload it here.

The yt-dlp documentation also recommends exporting cookies from a fresh private/incognito browser session because YouTube can rotate cookies on normal open tabs. ([GitHub][3])

---

# 2. Put the cookie file on your server

Let's create a dedicated directory:

```bash
sudo mkdir -p /opt/stacks/metube/cookies
```

Then transfer your `cookies.txt` from your Windows PC to:

```text
/opt/stacks/metube/cookies/cookies.txt
```

You can use **SFTP/WinSCP** for this.

Your directory should ultimately look like:

```text
/opt/stacks/metube/
├── docker-compose.yml
└── cookies/
    └── cookies.txt
```

Then:

```bash
sudo chmod 600 /opt/stacks/metube/cookies/cookies.txt
```

---

# 3. Modify your MeTube stack

Your current compose probably has:

```yaml
volumes:
  - /mnt/media/downloads:/downloads
```

Change it to:

```yaml
volumes:
  - /mnt/media/downloads:/downloads
  - /opt/stacks/metube/cookies/cookies.txt:/cookies.txt:ro
```

And add this under `environment`:

```yaml
YTDL_OPTIONS: '{"cookiefile":"/cookies.txt"}'
```

So your complete compose should be:

```yaml
services:
  metube:
    image: ghcr.io/alexta69/metube:latest
    container_name: metube

    restart: unless-stopped

    ports:
      - "8081:8081"

    environment:
      UID: 1000
      GID: 1000
      TZ: Asia/Kathmandu
      YTDL_OPTIONS: '{"cookiefile":"/cookies.txt"}'

    volumes:
      - /mnt/media/downloads:/downloads
      - /opt/stacks/metube/cookies/cookies.txt:/cookies.txt:ro
```

---

# 4. Redeploy in Portainer

In Portainer:

**Stacks → metube → Editor**

Update the compose configuration and click:

**Update the stack**

If Portainer doesn't recreate the container, run:

```bash
cd /opt/stacks/metube
docker compose down
docker compose up -d
```

Then check:

```bash
docker ps
```

You should see:

```text
metube    ...    Up ...
```

---

# 5. Test cookies directly

Before testing through MeTube, let's test yt-dlp itself.

Run:

```bash
docker exec -it metube yt-dlp --cookies /cookies.txt -F "https://www.youtube.com/watch?v=6TI07doT1ow"
```

### What we want to see

Instead of:

```text
HTTP Error 429
```

and:

```text
Sign in to confirm you're not a bot
```

you should see something like:

```text
ID  EXT   RESOLUTION ...
18  mp4   360p
...
137 mp4   1080p
140 m4a   audio only
```

If you get a list of formats, **we've fixed the main problem**.

Then try the same video through MeTube.

---

## One important thing about PO Tokens

You may still see a warning similar to:

```text
Unable to fetch GVS PO Token
```

Don't immediately worry about that.

YouTube's current PO-token enforcement is evolving. yt-dlp currently recommends using a **PO Token provider plugin** rather than manually extracting tokens, because many PO tokens are tied to individual videos and expire/change. ([GitHub][1])

So our troubleshooting order should be:

**First:**

```text
Cookies → test
```

If that works, we're done.

**If cookies still fail:**

```text
Cookies
   ↓
PO Token provider
   ↓
MeTube
```

rather than manually copying PO tokens for every video.

---

### One more important consideration

You currently have:

```text
MeTube → /mnt/media/downloads
```

That's good. I would **not** have MeTube download directly into your Jellyfin `Movies`, `TV Shows`, or Navidrome `Music` folders. Keeping:

```text
/mnt/media/downloads
```

as an incoming/download area is much safer. Later we can set up automatic organization if you want.

### Do this next

For now, **don't change anything else**.

Get the `cookies.txt` onto:

```text
/opt/stacks/metube/cookies/cookies.txt
```

update the compose as above, redeploy, and run:

```bash
docker exec -it metube yt-dlp --cookies /cookies.txt -F "https://www.youtube.com/watch?v=6TI07doT1ow"
```

**Send me the output of that command.** From that output we can determine whether cookies are sufficient or whether your current YouTube connection also needs a PO-token provider.

[1]: https://github.com/yt-dlp/yt-dlp/wiki/Po-Token-Guide?utm_source=chatgpt.com "PO Token Guide · yt-dlp/yt-dlp Wiki · GitHub"
[2]: https://github.com/yt-dlp/yt-dlp/wiki/FAQ?utm_source=chatgpt.com "FAQ · yt-dlp/yt-dlp Wiki · GitHub"
[3]: https://github.com/yt-dlp/yt-dlp/wiki/Extractors/619f9fd83f6ce39a5534b4fb1a076afcacb52038?utm_source=chatgpt.com "Extractors · yt-dlp/yt-dlp Wiki · GitHub"


Yes. Since you're using Firefox on your Windows PC, the easiest method is a Firefox cookie-export extension. yt-dlp officially supports cookies exported in Mozilla/Netscape format, and specifically mentions a Firefox `cookies.txt` exporter. ([GitHub][1])

### 1. Install a cookie exporter in Firefox

In Firefox, go to the Add-ons page and search for:

**cookies.txt**

Choose a reputable extension that exports cookies in **Netscape/Mozilla cookies.txt format**. yt-dlp's documentation specifically says the resulting file must start with either:

```text
# HTTP Cookie File
```

or

```text
# Netscape HTTP Cookie File
```

([GitHub][1])

### 2. Important: use a Private Window

For YouTube, I recommend doing this carefully because YouTube can rotate cookies when an existing tab is opened. yt-dlp recommends this procedure: ([GitHub][2])

1. Open **Firefox → New Private Window** (`Ctrl+Shift+P`).
2. Go to:

   [YouTube](https://www.youtube.com/?utm_source=chatgpt.com)
3. **Log in to your YouTube account.**
4. Open a YouTube video and make sure it plays normally.
5. Open a **new tab** in the same private window.
6. Close the YouTube tab.
7. Now use the cookie-export extension to export cookies.
8. Save the file as:

```text
cookies.txt
```

9. **Do not reopen YouTube in that private window.**
10. Close the entire private window after exporting.

The reason for this somewhat strange procedure is that YouTube may rotate authentication cookies when you continue using an open YouTube session, potentially making the exported cookies unusable. ([GitHub][2])

### 3. Check the file

Open `cookies.txt` with Notepad.

You should see something similar to:

```text
# Netscape HTTP Cookie File
.youtube.com    TRUE    /    TRUE    ...    ...
.youtube.com    TRUE    /    TRUE    ...    ...
```

**Don't send this file or its contents to me.** It contains authentication information and should be treated like a password.

### 4. Copy it to your homeserver

Using WinSCP, SFTP, or another method, copy it to:

```text
/opt/stacks/metube/cookies/cookies.txt
```

On the server:

```bash
sudo chmod 600 /opt/stacks/metube/cookies/cookies.txt
```

Then we'll test it **before changing MeTube's configuration**:

```bash
docker exec -it metube yt-dlp \
  --cookies /cookies.txt \
  -F "https://www.youtube.com/watch?v=6TI07doT1ow"
```

However, **one correction to my previous instructions:** before running that command, we need to make sure `/cookies.txt` is actually mounted into your container. So don't run the command until you've copied the file and added the volume mapping.

If you want, I can walk you through the **Firefox → export → WinSCP → Portainer → test** process one step at a time, using your existing `/opt/stacks/metube` setup.

[1]: https://github.com/yt-dlp/yt-dlp/wiki/FAQ?utm_source=chatgpt.com "FAQ · yt-dlp/yt-dlp Wiki · GitHub"
[2]: https://github.com/yt-dlp/yt-dlp/wiki/Extractors/619f9fd83f6ce39a5534b4fb1a076afcacb52038?utm_source=chatgpt.com "Extractors · yt-dlp/yt-dlp Wiki · GitHub"
