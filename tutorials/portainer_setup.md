Given your current Ubuntu Server setup (Docker, Jellyfin, qBittorrent, Samba, Tailscale, and your media mounted under `/mnt/media`), Portainer is an excellent addition. I recommend deploying it with Docker Compose so it fits the rest of your stack.

## Step 1: Create a directory for Portainer

```bash
mkdir -p ~/docker/portainer
cd ~/docker/portainer
```

(or `/opt/stacks/portainer` if you want to keep all Docker stacks together)

---

## Step 2: Create docker-compose.yml

```bash
nano docker-compose.yml
```

Paste the following:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped

    ports:
      - "9000:9000"
      - "9443:9443"

    volumes:
      - portainer_data:/data
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  portainer_data:
```

Save:

* **Ctrl+O**
* **Enter**
* **Ctrl+X**

---

## Step 3: Start Portainer

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

You should see something similar to:

```
portainer   Up
```

---

## Step 4: Open Portainer

From another computer on your network:

```
https://<server-ip>:9443
```

Example:

```
https://192.168.1.100:9443
```

Ignore the browser certificate warning the first time because Portainer uses a self-signed certificate by default.

---

## Step 5: Create the admin account

On first launch you'll be asked to:

* Create an administrator username
* Create a strong password

---

## Step 6: Connect to Docker

Choose:

```
Local
```

Then click

```
Connect
```

Portainer will immediately display your running containers.

---

# Verify everything

You should see at least:

* Jellyfin
* qBittorrent
* Portainer

along with their images, volumes, and networks.

---

# Optional: Organize your Docker stacks

Since you're building a long-term home server, I'd recommend this structure:

```
/opt/stacks
├── jellyfin
│   └── docker-compose.yml
├── qbittorrent
│   └── docker-compose.yml
├── portainer
│   └── docker-compose.yml
├── tailscale
│   └── docker-compose.yml
└── immich
    └── docker-compose.yml
```

Then you can manage each service independently:

```bash
cd /opt/stacks/portainer
docker compose pull
docker compose up -d
```

---

# Useful Portainer features

Once installed, you can:

* View container logs
* Start/stop/restart containers
* Update images
* Browse Docker volumes
* Open an interactive terminal inside containers
* Inspect networks
* Monitor CPU and memory usage
* Deploy new stacks directly from the web UI

---

## Next recommendation

Once Portainer is running, I recommend we:

1. Move your existing Jellyfin and qBittorrent Compose files into `/opt/stacks`.
2. Import them into Portainer as **Stacks**.
3. Later, deploy Immich (or another photo backup solution) and any future services directly through Portainer, so your entire home server can be managed from one interface.

If Portainer is asking for a **Setup Token** on the initial registration page, it's usually because the container was started in **agent mode** or it thinks it should connect to a remote environment instead of managing the local Docker engine.

Let's verify your deployment.

### 1. Check your running container

Run:

```bash
docker ps
```

Then inspect the container:

```bash
docker inspect portainer --format '{{.Config.Cmd}}'
```

and

```bash
docker inspect portainer --format '{{.Config.Image}}'
```

Please paste the output.

---

### 2. Check your compose file

Also show me the contents of your compose file:

```bash
cat ~/docker/portainer/docker-compose.yml
```

(or `/opt/stacks/portainer/docker-compose.yml` if you used that location).

---

### 3. If this is a fresh installation

If there is no data you need to keep, the quickest fix is to recreate Portainer from scratch:

```bash
docker compose down
docker volume rm portainer_portainer_data
docker compose up -d
```

**Only do this if Portainer has never been configured before**, because it deletes Portainer's configuration database (it does **not** delete your Docker containers, images, or application data).

---

### One question

When you browse to Portainer, is the URL:

```
https://<server-ip>:9443
```

And does the page say **"Create the first administrator user"**, or does it specifically say **"Enter setup token"**?

A screenshot of that page would also help pinpoint the issue quickly.

Your container looks normal. The fact that you're seeing a **Setup Token** page is not expected for a fresh standalone Portainer CE installation.

Let's inspect the configuration rather than guessing.

### Step 1: Show your Compose file

Run:

```bash
cat /opt/stacks/portainer/docker-compose.yml
```

Paste the output.

### Step 2: Check the Portainer logs

Run:

```bash
docker logs portainer --tail 100
```

Paste the output.

### Step 3: Check whether an old Portainer database exists

Run:

```bash
docker volume ls
```

and then:

```bash
docker volume inspect portainer_portainer_data
```

---

## One thing to check immediately

Are you opening:

```
https://<server-ip>:9443
```

or are you opening:

```
http://<server-ip>:9000
```

The recommended interface is **https://<server-ip>:9443**.

---

## If this is a brand-new installation

If you have **never configured Portainer before** on this server, we can completely reset it (this won't affect Jellyfin or qBittorrent):

```bash
docker compose down
docker volume rm portainer_portainer_data
docker image rm portainer/portainer-ce:latest
docker compose pull
docker compose up -d
```

Then browse to:

```
https://<server-ip>:9443
```

The first page should say **"Create the first administrator user"**, not ask for a setup token.

Please post:

1. `docker-compose.yml`
2. `docker logs portainer --tail 100`

Those two outputs should reveal why Portainer is presenting the setup token page.
