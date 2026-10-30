# Production Photo & Video Management: Immich Self-Hosted Server Deployment

## 1. Overview & Architectural Background
Immich is a high-performance, self-hosted photo and video management solution engineered as a direct replacement for proprietary cloud services like Google Photos and Apple iCloud. It provides automated mobile background syncing, multi-user asset libraries, and server-side machine learning inference pipelines.

### Architectural Breakdown & Core Mechanics
Unlike traditional PHP or monolithic photo platforms, Immich utilizes a distributed microservices design:
* **Immich Server**: Built with Node.js and TypeScript, handling client API requests, album organization, metadata indexing, user authorization, and job dispatching.
* **Immich Machine Learning (`immich-machine-learning`)**: A dedicated Python execution environment running on-nx and PyTorch models to perform local facial detection, recognition, automated tagging, and CLIP semantic vector search (e.g., searching for natural phrases like *"mountain sunsets with dogs"* without manual tagging).
* **PostgreSQL + `pgvector`**: The primary relational persistence layer. Immich uses the `vector` extension (`pgvecto.rs` / `pgvector`) to store high-dimensional image embeddings directly inside the database, enabling sub-millisecond nearest-neighbor similarity lookups.
* **Redis**: Acts as the message broker and cache layer, managing asynchronous job queues (transcoding, facial clustering, thumbnail generation, reverse geocoding) without blocking client API traffic.

---

## 2. Production Homelab Use Cases
* **Automated Mobile Backup**: Continuous, incremental background photo and video uploads from Android and iOS devices.
* **AI-Powered Local Semantic Search**: Search personal collections using natural language, face clustering, and geolocation maps without third-party cloud processing.
* **External Media Library Support**: Ingest existing photo collections stored on local storage or external NAS shares in-place without duplicating files.
* **Multi-User Family Management**: Independent private vaults with shared albums, partner sharing, and permission boundaries.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12) with Docker and Docker Compose installed.
* Hardware: Minimum 4 CPU cores and 4 GB RAM (8 GB+ recommended when machine learning models are active).
* Fast storage: Dedicated SSD/NVMe volume for database and thumbnail storage; bulk storage for original assets.
* Static local IP and reverse proxy configured with SSL and large request upload allowances (`client_max_body_size 0;`).

### Directory Layout
```text
/opt/homelab/immich/
├── compose.yaml
├── .env
├── postgres/
├── model-cache/
└── upload/
```

---

## 4. Step-by-Step Implementation

### Step 4.1: Create Persistent Host Directories
Prepare dedicated directories for Immich data tiers:

```bash
sudo mkdir -p /opt/homelab/immich/{postgres,model-cache,upload}
sudo chown -R $USER:$USER /opt/homelab/immich
cd /opt/homelab/immich
```

### Step 4.2: Configure Environment Variables (`.env`)
Generate a strong database password and create `/opt/homelab/immich/.env`:

```bash
DB_PASSWORD=$(openssl rand -hex 16)

cat <<EOF> /opt/homelab/immich/.env
# Deployment Paths
UPLOAD_LOCATION=/opt/homelab/immich/upload
DB_DATA_LOCATION=/opt/homelab/immich/postgres
MODEL_CACHE_LOCATION=/opt/homelab/immich/model-cache

# General Settings
IMMICH_VERSION=release
TZ=UTC

# Database Configuration
DB_PASSWORD=${DB_PASSWORD}
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
DB_HOSTNAME=immich-postgres
EOF
```

### Step 4.3: Deploy Docker Compose Manifest (`compose.yaml`)
Create `/opt/homelab/immich/compose.yaml`:

```yaml
services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    ports:
      - "2283:2283"
    depends_on:
      - redis
      - database
    restart: unless-stopped
    networks:
      - immich-net

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
    volumes:
      - ${MODEL_CACHE_LOCATION}:/cache
    env_file:
      - .env
    restart: unless-stopped
    networks:
      - immich-net

  redis:
    container_name: immich_redis
    image: docker.io/redis:6.2-alpine@sha256:eaba718fecd1196188580e8249aa90a09331996715b0883b378f2323c08d6037
    healthcheck:
      test: [ "CMD-SHELL", "redis-cli ping || exit 1" ]
    restart: unless-stopped
    networks:
      - immich-net

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14-pgvector-v0.1.1
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}" ]
    restart: unless-stopped
    networks:
      - immich-net

networks:
  immich-net:
    driver: bridge
```

Bring the stack online:

```bash
sudo docker compose up -d
```

---

## 5. Reverse Proxy & Network Considerations

Because mobile devices upload multi-gigabyte video files, reverse proxies (Nginx, Traefik, NPM) require specific payload adjustments.

### Nginx Proxy Manager Ingress Parameters
1. Add a **Proxy Host** forwarding to `immich-server:2283`.
2. Under the **SSL** tab, enforce **Force SSL** and **HTTP/2 Support**.
3. Under the **Advanced** tab, append the following snippet to eliminate upload buffers and request timeouts:

```nginx
client_max_body_size 0;
proxy_read_timeout 600s;
proxy_send_timeout 600s;
send_timeout 600s;
```

---

## 6. Verification & Health Diagnostics

Verify container orchestration and database health:

```bash
# 1. Inspect running status of the 4 microservice containers
docker compose ps

# 2. Check Immich server logs for successful DB connection and migration
docker logs --tail 30 immich_server

# 3. Test API ping endpoint
curl [http://127.0.0.1:2283/api/server-info/ping](http://127.0.0.1:2283/api/server-info/ping)
```

Open `http://<SERVER_IP>:2283` in your browser to run through the initial administrative setup wizard and pair mobile apps.

---

## 7. Troubleshooting & Operational Matrix

| Symptom / Failure | Diagnostic & Root Cause | Resolution |
| :--- | :--- | :--- |
| **Mobile Sync "Upload Failed (413 Payload Too Large)"** | The upstream reverse proxy is blocking large uploads. | Set `client_max_body_size 0;` in your reverse proxy (Nginx / NPM). |
| **High Host CPU During Initial Ingestion** | Machine learning and video transcoding jobs processing simultaneously. | In **Administration** ➔ **Settings** ➔ **Job Settings**, throttle maximum concurrency for ML and transcoding jobs to 1 or 2 workers. |
| **Facial Recognition Stalled** | The `immich-machine-learning` container cannot download Hugging Face models due to egress restrictions. | Ensure the host has outbound internet access on port 443; check container logs with `docker logs immich_machine_learning`. |
| **Database Corruption / WAL Lock** | Unexpected host shutdown while writing vector embeddings. | Ensure `data-checksums` are enabled and maintain daily dumps using `docker exec -t immich_postgres pg_dumpall -U postgres > backup.sql`. |
