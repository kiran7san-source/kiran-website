# Enterprise Private Cloud Storage: Seafile Community Edition Deployment with Docker Compose

## 1. Overview & Architectural Background
Seafile is an open-source, high-performance file sync-and-share platform designed for reliability, speed, and cross-platform synchronization. Unlike traditional web-based file managers that handle files as single static units on disk, Seafile operates on a distributed, content-addressable block storage architecture.

### Architectural Comparison: Seafile vs. Nextcloud / ownCloud
* **Block-Level Storage vs. Direct Filesystem**: Platforms like Nextcloud write files directly onto a standard filesystem hierarchy. Seafile segments files into variable-sized cryptographic data blocks (similar to Git's object model) managed by dedicated internal daemons (`seafile-server` and `ccnet-server`). 
* **Differential Delta Syncing**: When modifying a 5 GB archive or database file, Seafile uploads and recalculates only the changed blocks rather than re-transmitting the entire multi-gigabyte file, resulting in dramatically faster synchronization speeds and lower bandwidth consumption.
* **Separation of Metadata and Storage**: File and library metadata are indexed inside a relational database (MariaDB/MySQL), while actual raw data blocks reside under a structured storage tree (`seafile-data/storage`).
* **Caching & Session State**: Integrates natively with Memcached to cache directory hierarchies, user sessions, and block tokens, delivering instantaneous navigation across extensive file libraries.

---

## 2. Production Homelab Use Cases
* **High-Speed Workstation Sync**: Bidirectional, background delta-sync across macOS, Linux, and Windows clients with selective folder synchronization.
* **Encrypted Private Libraries**: Client-side encrypted folders where encryption keys never leave the local user device, ensuring zero-knowledge privacy on untrusted infrastructure.
* **Direct File Sharing & External Ingress**: Secure public share links with expiration limits, download passwords, and upload-only dropboxes.
* **Low-Overhead Cloud Architecture**: Operates efficiently on low-power homelab servers without the CPU or database contention typical of heavier PHP-based cloud stacks.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12) with Docker and Docker Compose installed.
* Static host IP or configured reverse proxy (e.g., Nginx Proxy Manager) terminating with valid SSL.
* Host domain or local FQDN (e.g., `cloud.home.lab` or `seafile.kiranpsankhi.com.np`).
* Dedicated storage volume with ample capacity for block persistence.

### Directory Layout
```text
/opt/homelab/seafile/
├── compose.yaml
├── .env
├── seafile-data/
└── db-data/
```

---

## 4. Step-by-Step Implementation

### Step 4.1: Prepare Host Directories
Create the directory structure for persistent database volumes and Seafile block storage:

```bash
sudo mkdir -p /opt/homelab/seafile/{seafile-data,db-data}
sudo chown -R $USER:$USER /opt/homelab/seafile
cd /opt/homelab/seafile
```

### Step 4.2: Configure Environment Variables (`.env`)
Generate strong random passwords for the database root account and Seafile service user, then define the environment variables:

```bash
MYSQL_ROOT_PW=$(openssl rand -hex 16)
DB_PW=$(openssl rand -hex 16)
ADMIN_PW=$(openssl rand -base64 12)

cat <<EOF> /opt/homelab/seafile/.env
# Database Credentials
MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PW}
DB_PASSWORD=${DB_PW}

# Seafile Administrative Account
SEAFILE_ADMIN_EMAIL=admin@home.lab
SEAFILE_ADMIN_PASSWORD=${ADMIN_PW}

# Host FQDN & Timezone
SEAFILE_SERVER_HOSTNAME=cloud.home.lab
TIME_ZONE=UTC
EOF

echo "Generated Admin Password: ${ADMIN_PW}"
```

> **Note**: Record the output admin password securely to access the initial web dashboard.

### Step 4.3: Deploy Docker Compose Manifest (`compose.yaml`)
Create `/opt/homelab/seafile/compose.yaml`:

```yaml
services:
  db:
    image: mariadb:10.11
    container_name: seafile-mariadb
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_LOG_CONSOLE=true
      - MARIADB_AUTO_UPGRADE=1
    volumes:
      - ./db-data:/var/lib/mysql
    networks:
      - seafile-net

  memcached:
    image: memcached:1.6-alpine
    container_name: seafile-memcached
    restart: unless-stopped
    entrypoint: memcached -m 256
    networks:
      - seafile-net

  seafile:
    image: seafileltd/seafile-mc:latest
    container_name: seafile
    restart: unless-stopped
    ports:
      - "8082:80"   # Seahub Web Frontend
    volumes:
      - ./seafile-data:/shared
    environment:
      - DB_HOST=db
      - DB_ROOT_PASSWD=${MYSQL_ROOT_PASSWORD}
      - TIME_ZONE=${TIME_ZONE}
      - SEAFILE_ADMIN_EMAIL=${SEAFILE_ADMIN_EMAIL}
      - SEAFILE_ADMIN_PASSWORD=${SEAFILE_ADMIN_PASSWORD}
      - SEAFILE_SERVER_LETSENCRYPT=false
      - SEAFILE_SERVER_HOSTNAME=${SEAFILE_SERVER_HOSTNAME}
    depends_on:
      - db
      - memcached
    networks:
      - seafile-net

networks:
  seafile-net:
    driver: bridge
```

Launch the stack:

```bash
sudo docker compose up -d
```

---

## 5. Reverse Proxy & Service URLs Configuration

Seafile requires strict alignment between client-facing URLs and internal block server endpoints (`seaf-server`) to prevent file upload failures.

### Step 5.1: Adjust Seahub Settings (`seahub_settings.py`)
After the initial container spin-up finishes, edit `/opt/homelab/seafile/seafile-data/seafile/conf/seahub_settings.py` to match your reverse proxy domain:

```python
SERVICE_URL = '[https://cloud.home.lab](https://cloud.home.lab)'
FILE_SERVER_ROOT = '[https://cloud.home.lab/seafhttp](https://cloud.home.lab/seafhttp)'
```

### Step 5.2: Reverse Proxy Configuration (Nginx / Nginx Proxy Manager)
1. Direct ingress domain `cloud.home.lab` to forward port `8082` on your Seafile host.
2. In your Nginx server block or advanced configuration, configure custom locations to pass web traffic and the raw block file server correctly:

```nginx
client_max_body_size 0;

location / {
    proxy_pass [http://127.0.0.1:8082](http://127.0.0.1:8082);
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 1200s;
}

location /seafhttp {
    rewrite ^/seafhttp(.*)$ $1 break;
    proxy_pass [http://127.0.0.1:8082](http://127.0.0.1:8082);
    client_max_body_size 0;
    proxy_connect_timeout 36000s;
    proxy_read_timeout 36000s;
    proxy_send_timeout 36000s;
    send_timeout 36000s;
}
```

Restart the Seafile service to load URL modifications:

```bash
sudo docker compose restart seafile
```

---

## 6. Verification & Health Checks

```bash
# 1. Verify container runtime states
docker compose ps

# 2. Check Seahub initialization logs
docker logs --tail 30 seafile

# 3. Test WebUI responsiveness
curl -kI [https://cloud.home.lab](https://cloud.home.lab)
```

Open `https://cloud.home.lab` in your browser and log in with your configured `SEAFILE_ADMIN_EMAIL` and `SEAFILE_ADMIN_PASSWORD`.

---

## 7. Operational Troubleshooting Matrix

| Symptom / Error | Root Cause | Remediation Procedure |
| :--- | :--- | :--- |
| **"Network Error" or 0-byte file uploads fail in WebUI** | Mismatch in `FILE_SERVER_ROOT` pointing to incorrect HTTP scheme or missing `/seafhttp` proxy rewrite. | Verify `FILE_SERVER_ROOT = 'https://cloud.home.lab/seafhttp'` in `seahub_settings.py` and ensure the proxy forwards `/seafhttp` without request timeouts. |
| **`Permission Denied` on container startup** | Internal container user UID 1000 lacks read/write ownership over mounted volume paths. | Set permissions recursively: `sudo chown -R 1000:1000 /opt/homelab/seafile/seafile-data`. |
| **Database connection refused during initialization** | MariaDB container healthcheck was not ready before Seafile attempted database schema migration. | Restart the main Seafile container once MariaDB stabilizes: `docker compose restart seafile`. |
| **Database Growth / Stale Data Blocks** | Deleted files remain in library history garbage collection buffers. | Run the built-in Seafile garbage collector: `docker exec -it seafile /opt/seafile/seafile-server-latest/seaf-gc.sh run`. |
