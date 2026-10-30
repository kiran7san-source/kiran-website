# Production Self-Hosted Password Management: Vaultwarden Deployment & Hardening

## 1. Overview & Architectural Background
Vaultwarden is an alternative implementation of the Bitwarden server API written in Rust. It provides full compatibility with official Bitwarden mobile apps, browser extensions, and desktop clients while significantly reducing operational overhead compared to the upstream enterprise Bitwarden stack.

### Upstream Bitwarden vs. Vaultwarden Mechanics
* **Resource Footprint**: Upstream Bitwarden is an enterprise-oriented suite composed of a dozen ASP.NET microservices requiring Microsoft SQL Server and 3–4 GB of base memory. Vaultwarden rewrites the API layer in asynchronous Rust (Actix-web / Rocket), executing as a single lightweight binary consuming roughly 30–50 MB of RAM.
* **Database Backend**: Vaultwarden defaults to an embedded SQLite database using Write-Ahead Logging (WAL) for high concurrency, with native support for external MariaDB/PostgreSQL databases when scaling across enterprise nodes.
* **Cryptographic Guarantees**: All cryptographic operations (Argon2id master key derivation, AES-256 vault encryption, and asymmetric RSA key generation) happen entirely client-side. The server strictly stores zero-knowledge encrypted ciphertext, blind hashes, and initialization vectors.
* **Mandatory TLS Requirement**: WebCrypto APIs enforce that browser extensions and modern web vaults will only function over secure HTTPS contexts (or `localhost`).

---

## 2. Production Homelab Use Cases
* **Centralized Credential Vaulting**: Secure passwords, software licenses, SSH keys, identity records, and secure notes across all personal and family devices.
* **Integrated Two-Factor Authentication (2FA)**: Built-in TOTP authenticator engine, eliminating the need for a secondary mobile authenticator app.
* **Secure File Attachments & Sends**: End-to-end encrypted file sharing via Bitwarden Send.
* **Emergency Access**: Designated emergency fallback contacts for family succession planning.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12) with Docker and Docker Compose installed.
* Fully Qualified Domain Name (FQDN) or local domain (e.g., `vault.home.lab` or `vault.kiranpsankhi.com.np`) terminating with a valid HTTPS certificate (via Nginx Proxy Manager, Caddy, or Traefik).
* Dedicated directory for persistent vault data and automated backups.

### Directory Layout
```text
/opt/homelab/vaultwarden/
├── compose.yaml
├── .env
├── backups/
└── data/
```

---

## 4. Step-by-Step Implementation

### Step 4.1: Prepare Host Directories
Create the directory structure and set proper ownership permissions:

```bash
sudo mkdir -p /opt/homelab/vaultwarden/{data,backups}
cd /opt/homelab/vaultwarden
```

### Step 4.2: Generate an Administrative Token
Vaultwarden provides a web-based `/admin` management panel. Protect it with an Argon2-hashed secret token using `openssl`:

```bash
# Generate a strong 32-byte secret token
ADMIN_TOKEN=$(openssl rand -hex 32)
echo "Generated Admin Token: ${ADMIN_TOKEN}"
```

> **Note**: Store this output string safely; you will paste it into the environment configuration.

### Step 4.3: Create Environment Configuration (`.env`)
Create `/opt/homelab/vaultwarden/.env`:

```bash
cat <<EOF> /opt/homelab/vaultwarden/.env
# Vaultwarden Core Configuration
DOMAIN=[https://vault.home.lab](https://vault.home.lab)
ADMIN_TOKEN=${ADMIN_TOKEN}
SIGNUPS_ALLOWED=false
INVITATIONS_ALLOWED=true
SHOW_PASSWORD_HINT=false

# WebSocket Push Notifications
WEBSOCKET_ENABLED=true

# Database Tuning
DATABASE_URL=data/db.sqlite3
DATABASE_MAX_CONNS=10

# Logging & Storage
LOG_LEVEL=warn
EXTENDED_LOGGING=true
LOG_FILE=/data/vaultwarden.log
EOF
```

### Step 4.4: Deploy Docker Compose Manifest (`compose.yaml`)
Create `/opt/homelab/vaultwarden/compose.yaml`:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - ./data:/data
    ports:
      - "127.0.0.1:8088:80"    # HTTP API / Web Vault (reverse proxy target)
    environment:
      - WEBSOCKET_PORT=3012
    networks:
      - ingress-net

networks:
  ingress-net:
    external: true
```

*Note: If your reverse proxy runs on the same Docker host, attaching Vaultwarden to your shared `ingress-net` bridge lets your proxy resolve `http://vaultwarden:80` directly.*

Launch the container:

```bash
sudo docker compose up -d
```

---

## 5. Reverse Proxy Configuration & WebSocket Setup

To support live real-time sync across open browser tabs without manual refreshing, route both standard HTTP traffic and WebSocket connections through your reverse proxy.

### Nginx / Nginx Proxy Manager Configuration:
1. **Details**:
   * **Forward Hostname / IP**: `vaultwarden` (or local IP)
   * **Forward Port**: `80` (or `8088`)
   * Check **Websockets Support**.
2. **SSL Tab**:
   * Select a valid Wildcard or Domain SSL certificate.
   * Check **Force SSL**, **HTTP/2 Support**, and **HSTS Enabled**.
3. **Custom Locations (If not using integrated WebSocket support)**:
   * Define path `/notifications/hub` forwarding to port `3012`.

---

## 6. Automated Backup Strategy (SQLite Snapshots)

Because SQLite uses continuous read/write locks, copying `db.sqlite3` while the container is running risks database corruption. Use the SQLite online backup API via the CLI to create safe, point-in-time snapshots.

Create an automated backup script `/opt/homelab/vaultwarden/backup-vault.sh`:

```bash
sudo tee /opt/homelab/vaultwarden/backup-vault.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/opt/homelab/vaultwarden/backups"
DATA_DIR="/opt/homelab/vaultwarden/data"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="${BACKUP_DIR}/vault_backup_${TIMESTAMP}.sqlite3"

mkdir -p "${BACKUP_DIR}"

# 1. Safe SQLite snapshot via sqlite3 CLI inside container
docker exec vaultwarden sqlite3 /data/db.sqlite3 ".backup '/data/db_snapshot.sqlite3'"

# 2. Move snapshot and compress attachments
mv "${DATA_DIR}/db_snapshot.sqlite3" "${BACKUP_FILE}"
tar -czf "${BACKUP_DIR}/attachments_${TIMESTAMP}.tar.gz" -C "${DATA_DIR}" attachments/

# 3. Prune snapshots older than 14 days
find "${BACKUP_DIR}" -type f -name "vault_backup_*.sqlite3" -mtime +14 -delete
find "${BACKUP_DIR}" -type f -name "attachments_*.tar.gz" -mtime +14 -delete

echo "[$(date)] Vaultwarden backup completed successfully."
EOF

sudo chmod +x /opt/homelab/vaultwarden/backup-vault.sh
```

Add a scheduled cron job to run nightly at 2:00 AM:

```bash
(crontab -l 2>/dev/null; echo "0 2 * * * /opt/homelab/vaultwarden/backup-vault.sh >> /var/log/vaultwarden_backup.log 2>&1") | crontab -
```

---

## 7. Verification & Operational Matrix

```bash
# 1. Verify container runtime health
docker ps --filter "name=vaultwarden"

# 2. Inspect active logs for database migrations
docker logs --tail 30 vaultwarden

# 3. Test API endpoint reachability
curl -kI [https://vault.home.lab/alive](https://vault.home.lab/alive)
```

### Operational Troubleshooting Matrix

| Symptom / Task | Diagnostic & Remediation |
| :--- | :--- |
| **"Cannot decrypt vault / WebCrypto error"** | Ensure the connection is strictly over HTTPS. Browsers disable WebCrypto APIs on plain HTTP (unencrypted) origins. |
| **Admin Panel 404** | Verify `ADMIN_TOKEN` is declared in `.env`. Access via `https://vault.home.lab/admin`. |
| **New Users Cannot Register** | `SIGNUPS_ALLOWED=false` is enforced for security. Log into the `/admin` portal and issue an invitation email or link. |
| **WebSocket Sync Fails** | Ensure reverse proxy forwards `/notifications/hub` with HTTP upgrade headers: `Upgrade $http_upgrade; Connection "upgrade";`. |
