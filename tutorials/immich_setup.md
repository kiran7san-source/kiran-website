A **clean Immich installation** will be much simpler, and you'll avoid carrying over any hidden database issues.

Since we're starting fresh, we'll completely remove the old Immich stack, volumes, images, and recovered database while **leaving Jellyfin and qBittorrent untouched**.

---

# Step 1: Stop the Immich stack

```bash
cd /opt/stacks/immich

docker compose down
```

Verify nothing is running:

```bash
docker ps -a
```

---

# Step 2: Remove the Immich containers

```bash
docker rm -f immich_server immich_machine_learning immich_postgres immich_redis 2>/dev/null
```

---

# Step 3: Remove the Immich images

```bash
docker image rm -f \
ghcr.io/immich-app/immich-server:release \
ghcr.io/immich-app/immich-machine-learning:release \
ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0 \
docker.io/valkey/valkey:9
```

This **will not** affect Jellyfin or qBittorrent.

---

# Step 4: Delete the old stack

```bash
sudo rm -rf /opt/stacks/immich
```

---

# Step 5: Delete the old data

Since you're starting over:

```bash
sudo rm -rf /mnt/cloud/immich
```

---

# Step 6: Create a clean directory structure

I recommend storing everything under your 2 TB data drive.

```text
/mnt/cloud/
└── immich/
    ├── library
    ├── postgres
    └── model-cache
```

Create it:

```bash
sudo mkdir -p /mnt/cloud/immich/{library,postgres,model-cache}
```

Set ownership:

```bash
sudo chown -R 1000:1000 /mnt/cloud/immich
```

---

# Step 7: Create the stack directory

```bash
sudo mkdir -p /opt/stacks/immich

sudo chown -R $USER:$USER /opt/stacks/immich

cd /opt/stacks/immich
```

---

# Step 8: Download a fresh compose file

Download the latest `docker-compose.yml` from the official Immich installation guide and save it in `/opt/stacks/immich`.

Then create a new `.env`.

Example:

```env
UPLOAD_LOCATION=/mnt/cloud/immich/library

DB_DATA_LOCATION=/mnt/cloud/immich/postgres

IMMICH_VERSION=release

DB_PASSWORD=choose_a_strong_password

DB_USERNAME=postgres

DB_DATABASE_NAME=immich
```

---

# Step 9: Start Immich

```bash
docker compose pull

docker compose up -d
```

Wait about a minute.

Then verify:

```bash
docker ps
```

All four containers should be **healthy**.

---

# Step 10: Open Immich

Visit

```
http://<server-ip>:2283
```

Create your administrator account.

---

# Step 11: Import your existing photos

Since your photos are already organized elsewhere, I recommend **using an External Library** instead of uploading everything again.

For example, if your photos are stored in:

```text
/mnt/media/Photos
```

or

```text
/mnt/cloud/Photos
```

you can mount that folder read-only into the Immich server:

```yaml
volumes:
  - ${UPLOAD_LOCATION}:/data
  - /mnt/media/Photos:/external/photos:ro
  - /etc/localtime:/etc/localtime:ro
```

Then, inside Immich:

1. Go to **Administration → External Libraries**.
2. Create a new library.
3. Point it to `/external/photos`.
4. Start a scan.

This avoids duplicating your photos on disk while still letting Immich index and manage them.

## One recommendation before you proceed

Given the unusual startup issues we encountered with the latest `release` image, I recommend **using a specific Immich version instead of the floating `release` tag** once we've confirmed a stable version. Pinning the version makes future upgrades predictable and prevents unexpected changes.

I can also help you build a production-ready Docker Compose setup for your server (Immich + Jellyfin + qBittorrent + Tailscale) with proper directory layout, health checks, and backup strategy so future migrations are much easier.
