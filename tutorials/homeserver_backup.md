# Enterprise Homelab Disaster Recovery: Automated Backup Strategies with Restic

## 1. Overview & Architectural Background
Data integrity and recovery readiness form the foundation of dependable homelab infrastructure. While container manifests and configurations can be redeployed within minutes, unrecoverable data loss—stemming from drive degradation, accidental deletions, or database state corruptions—is irreversible without a verified backup regimen.

This guide details a production-grade backup pipeline implementing the industry-standard **3-2-1 backup strategy** powered by **Restic**, a modern, cryptographically secure, and deduplicated backup framework.

### The 3-2-1 Backup Strategy
* **3 Copies of Data**: Maintain the production operational dataset plus at least two independent backup datasets.
* **2 Different Storage Media**: Diversify physical storage targets (e.g., primary NVMe/SATA internal disk array and an external dedicated backup pool/HDD).
* **1 Offsite Replica**: Maintain at least one replica geographically separated or physically isolated from the local network (e.g., encrypted cloud target or off-site NAS) to defend against fire, theft, or catastrophic local network compromise.

### Architectural Advantages of Restic
Unlike traditional archive mechanisms (`tar`, `cp`, or basic `rsync`) that copy full filesystem trees or naive incremental deltas:
* **Content-Addressable Block Deduplication**: Restic segments files into variable-sized chunks using dynamic Rabin fingerprints. Only novel, non-redundant blocks are committed to the repository, drastically reducing storage consumption across recurring snapshots.
* **Zero-Knowledge End-to-End Encryption**: Every repository chunk, index, and header is authenticated and encrypted using AES-256 in Galois/Counter Mode (GCM) with Poly1305 MACs prior to leaving the host machine.
* **Point-in-Time Snapshot Isolation**: Each backup execution generates an atomic snapshot representing the exact state of the filesystem tree without relying on complex, fragile incremental dependency chains.
* **Integrity Auditing**: Built-in verification utilities audit repository cryptographic hashes against silent data corruption (bit rot).

---

## 2. Core Homelab Backup Topologies
* **Container Volume Persistence**: Back up `/opt/homelab` configuration manifests, Compose definitions, and application volume bindings.
* **Database State Freezing**: Safely extract consistent point-in-time database dumps (PostgreSQL, MariaDB, SQLite) before running storage snapshots.
* **Immutable Target Separation**: Mount dedicated backup targets with restricted service permissions to protect against rogue workloads or user-level errors.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12).
* Administrative `sudo` privileges.
* Primary backup destination: An external dedicated storage partition or NAS mount point (e.g., `/mnt/backup-drive/restic-repo`).
* `restic` binary installed on the host.

### Directory Layout
```text
/opt/homelab/backups/
├── restic-backup.sh
├── restic-password.txt
├── backup.exclude
└── logs/
```

---

## 4. Step-by-Step Implementation

### Step 4.1: Install Restic
Install the latest stable Restic release from the official distribution repositories:

```bash
sudo apt-get update
sudo apt-get install -y restic
```

Verify binary functionality and version:

```bash
restic version
```

### Step 4.2: Prepare the Storage Target & Credentials
1. Create the dedicated backup script and configuration directory:

```bash
sudo mkdir -p /opt/homelab/backups/logs
sudo chmod 700 /opt/homelab/backups
```

2. Generate a high-entropy encryption passphrase and save it to an access-restricted file:

```bash
openssl rand -base64 32 | sudo tee /opt/homelab/backups/restic-password.txt > /dev/null
sudo chmod 600 /opt/homelab/backups/restic-password.txt
```

> **Warning**: Back up this password and your repository keys separately offsite. If you lose this encryption passphrase, all repository data is mathematically irrecoverable.

3. Initialize the Restic repository on your target storage mount:

```bash
sudo restic init \
  --repo /mnt/backup-drive/restic-repo \
  --password-file /opt/homelab/backups/restic-password.txt
```

### Step 4.3: Define Exclusion Rules (`backup.exclude`)
Certain files (ephemeral caches, transient runtime sockets, system swap files) waste storage space and should be excluded.

Create `/opt/homelab/backups/backup.exclude`:

```text
# Cache & Ephemeral Files
*/cache/*
*/Cache/*
*/.cache/*
*.tmp
*.log

# Docker Temporary Sockets
*/docker.sock

# Jellyfin / Media Transcode Buffers
/opt/homelab/jellyfin/cache/*
/opt/homelab/immich/model-cache/*
```

---

## 5. Automated Backup & Maintenance Pipeline

Because active databases can change while files are read, always dump databases to disk before invoking Restic snapshots.

Create the orchestration script `/opt/homelab/backups/restic-backup.sh`:

```bash
sudo tee /opt/homelab/backups/restic-backup.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

# --- Configuration ---
export RESTIC_REPOSITORY="/mnt/backup-drive/restic-repo"
export RESTIC_PASSWORD_FILE="/opt/homelab/backups/restic-password.txt"
EXCLUDE_FILE="/opt/homelab/backups/backup.exclude"
BACKUP_PATHS="/opt/homelab"
DB_DUMP_DIR="/opt/homelab/backups/db-dumps"
DATE_TAG=$(date +"%Y%m%d_%H%M%S")

mkdir -p "${DB_DUMP_DIR}"

echo "[$(date)] === Starting Database Pre-Backup Dumps ==="

# 1. Immich PostgreSQL Dump (if running)
if docker ps --format '{{.Names}}' | grep -q "^immich_postgres$"; then
    echo "Dumping Immich database..."
    docker exec -t immich_postgres pg_dumpall -c -U postgres > "${DB_DUMP_DIR}/immich_dump_${DATE_TAG}.sql"
fi

# 2. Vaultwarden SQLite Online Safe Backup (if running)
if docker ps --format '{{.Names}}' | grep -q "^vaultwarden$"; then
    echo "Snapshotting Vaultwarden database..."
    docker exec -t vaultwarden sqlite3 /data/db.sqlite3 ".backup '/data/db_dump_${DATE_TAG}.sqlite3'"
    mv /opt/homelab/vaultwarden/data/db_dump_${DATE_TAG}.sqlite3 "${DB_DUMP_DIR}/"
fi

echo "[$(date)] === Executing Restic Snapshot ==="
restic backup \
    --exclude-file="${EXCLUDE_FILE}" \
    --tag "homelab-automated" \
    --verbose \
    ${BACKUP_PATHS}

echo "[$(date)] === Cleaning Up Pre-Backup Dumps ==="
rm -rf "${DB_DUMP_DIR:?}"/*

echo "[$(date)] === Applying Snapshot Retention Policy (Pruning) ==="
# Retain last 7 daily, 4 weekly, and 6 monthly snapshots
restic forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 6 \
    --prune

echo "[$(date)] === Auditing Repository Integrity Check ==="
restic check --read-data-subset=5%

echo "[$(date)] === Backup Pipeline Finished Successfully ==="
EOF

sudo chmod 700 /opt/homelab/backups/restic-backup.sh
```

---

## 6. Automation via Cron Scheduling

Schedule the backup pipeline to run nightly during off-peak hours (e.g., 03:00 AM) with automatic log redirection:

```bash
(sudo crontab -l 2>/dev/null; echo "0 3 * * * /opt/homelab/backups/restic-backup.sh >> /opt/homelab/backups/logs/backup_\$(date +\%Y\%m).log 2>&1") | sudo crontab -
```

---

## 7. Restoration & Disaster Recovery Drills

An untested backup is not a valid backup. Regularly rehearse recovery procedures to verify pipeline integrity.

```bash
# 1. List all available snapshots
sudo restic -r /mnt/backup-drive/restic-repo \
  --password-file /opt/homelab/backups/restic-password.txt snapshots

# 2. Inspect contents of the latest snapshot
sudo restic -r /mnt/backup-drive/restic-repo \
  --password-file /opt/homelab/backups/restic-password.txt ls latest

# 3. Restore files to an isolated verification target
sudo mkdir -p /tmp/restore-test
sudo restic -r /mnt/backup-drive/restic-repo \
  --password-file /opt/homelab/backups/restic-password.txt \
  restore latest --target /tmp/restore-test

# 4. Verify restored tree structure
ls -la /tmp/restore-test/opt/homelab
```

---

## 8. Operational Troubleshooting Matrix

| Symptom / Event | Underlying Cause | Corrective Action |
| :--- | :--- | :--- |
| **`unable to create lock in backend: repository is already locked`** | A prior backup process terminated abnormally or was interrupted by a host reboot. | Verify no backup processes are actively running with `pgrep restic`, then execute `sudo restic unlock --password-file /path/to/password.txt`. |
| **`check found pack files containing unused data`** | Snapshots were forgotten without running prune operations. | Run `restic prune` to physically purge abandoned chunks from disk. |
| **High Memory Usage on Large Repositories** | Deduplication index lookups consuming RAM during large tree parsing. | Set `GOGC=20` before the restic command in the bash script to force aggressive garbage collection. |
| **`file has changed during read` Warnings** | Active application writing to an un-dumped file while Restic is indexing. | Ensure the affected path is either excluded in `backup.exclude` or handled via an offline application hook prior to running the snapshot. |
