# Production Digital Library Management: Bookorbit / Calibre Server Deployment

## 1. Overview & Architectural Background
Managing personal collections of eBooks, technical documentation, PDFs, and comics often requires balancing powerful metadata organization on the desktop with accessible, multi-device web reading and over-the-air syncing. 

**Bookorbit** (and Calibre-Web ecosystems) provides an elegant, clean web front-end interface built on top of an existing **Calibre** database. It transforms static local book folders into an interactive personal cloud library accessible via browsers, dedicated mobile apps, and e-ink reading devices (such as Onyx Boox, Kindle, or Kobo).

### Architectural Breakdown & Core Mechanics
* **Calibre Database Model (`metadata.db`)**: Calibre organizes libraries around a structured SQLite database (`metadata.db`) paired with standardized file directory hierarchies (`Author Name/Book Title (ID)/`). Bookorbit interacts directly with this SQLite database schema to query titles, authors, tags, series ordering, and publishing metadata without altering the underlying storage convention.
* **In-Browser Reading Engines**: Bookorbit and modern web readers embed client-side EPUB/PDF rendering engines (such as EPUB.js and PDF.js). Readers can stream books directly in any web browser with instant pagination, font sizing adjustments, night mode styling, and persistent reading progress without requiring local client downloads.
* **OPDS (Open Publication Distribution System) Feeds**: Provides an integrated OPDS XML catalog endpoint. Any e-ink tablet or mobile reading app supporting the OPDS standard (e.g., KOReader, Moon+ Reader, NeoReader) can browse the remote library, search by tags, and download titles directly over the local network.
* **Over-the-Air Device Syncing**: Integrates SMTP mailers to dispatch converted `.mobi` or `.epub` files directly to Amazon Kindle or e-reader cloud accounts with a single click.

---

## 2. Production Homelab Use Cases
* **E-Ink Tablet Integration**: Seamlessly browse and pull technical PDFs, reference guides, and literature directly into e-ink devices (like Onyx Boox or Kobo) via OPDS.
* **Cross-Device Browser Reading**: Open and read books instantly across laptops, phones, and workstations while keeping bookmarks and progress synchronized.
* **Centralized Metadata Standardization**: Standardize author names, ISBNs, series rankings, and high-resolution cover art across extensive digital document collections.
* **Multi-User Library Profiles**: Create individual user accounts with dedicated reading lists, shelves, and granular restrictions (e.g., family-friendly shelf permissions).

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12) with Docker and Docker Compose installed.
* Existing or initialized Calibre library containing a valid `metadata.db` file.
* Dedicated persistent storage mount for the book repository (e.g., `/mnt/storage/books`).
* Accessible reverse proxy (Nginx Proxy Manager, Caddy, or Traefik) terminating with SSL.
* Host domain or local FQDN (e.g., `books.home.lab` or `books.kiranpsankhi.com.np`).

### Directory Layout
```text
/opt/homelab/bookorbit/
├── compose.yaml
├── config/
└── library/ (or mounted to external storage /mnt/storage/books)
```

---

## 4. Step-by-Step Implementation

### Step 4.1: Prepare Host Directories
Create the configuration directory and ensure the library storage path exists:

```bash
sudo mkdir -p /opt/homelab/bookorbit/config
sudo mkdir -p /mnt/storage/books
sudo chown -R $USER:$USER /opt/homelab/bookorbit
cd /opt/homelab/bookorbit
```

> **Note on Initial `metadata.db`**: Calibre-based web frontends require a valid `metadata.db` file inside the library directory. If you are starting with a completely empty directory, you can generate an empty initial Calibre database via the Calibre CLI (`calibredb`) or copy an existing `metadata.db` into `/mnt/storage/books/`.

### Step 4.2: Deploy Docker Compose Manifest (`compose.yaml`)
Create `/opt/homelab/bookorbit/compose.yaml`:

```yaml
services:
  bookorbit:
    image: lscr.io/linuxserver/calibre-web:latest
    container_name: bookorbit
    restart: unless-stopped
    environment:
      - PUID=1000 # Replace with host UID (id -u)
      - PGID=1000 # Replace with host GID (id -g)
      - TZ=UTC
      - DOCKER_MODS=linuxserver/mods:universal-calibre # Enables full ebook conversion tools
    volumes:
      - ./config:/config
      - /mnt/storage/books:/books
    ports:
      - "8083:8083"
    networks:
      - ingress-net

networks:
  ingress-net:
    external: true
```

*Note: The `DOCKER_MODS` variable installs underlying conversion binaries (such as `ebook-convert`), allowing the server to dynamically convert formats between EPUB, PDF, and MOBI on the fly.*

Launch the container:

```bash
sudo docker compose up -d
```

---

## 5. Reverse Proxy Configuration & OPDS Endpoint

### Step 5.1: Reverse Proxy Configuration (Nginx Proxy Manager)
1. In NPM, navigate to **Hosts** ➔ **Proxy Hosts** ➔ **Add Proxy Host**:
   * **Domain Names**: `books.home.lab`
   * **Scheme**: `http`
   * **Forward Hostname / IP**: `bookorbit` (or your host IP)
   * **Forward Port**: `8083`
   * Check **Block Common Exploits** and **Websockets Support**.
2. Under the **SSL** tab:
   * Select your SSL certificate.
   * Check **Force SSL** and **HTTP/2 Support**.
3. Under the **Advanced** tab, ensure large book upload operations (such as multi-megabyte technical PDF manuals) do not time out:

```nginx
client_max_body_size 500M;
```

---

## 6. Initial Setup & Device Synchronization

### Step 6.1: Initial Web Configuration
1. Open your browser and navigate to `https://books.home.lab` (or `http://<SERVER_IP>:8083`).
2. Log in with the default initial credentials:
   * **Username**: `admin`
   * **Password**: `admin123` *(Change this immediately after login!)*
3. On the initial database setup screen, set **Location of Calibre database** to:
   ```text
   /books
   ```
4. Click **Save**. The dashboard will load and display your indexed library collection.

### Step 6.2: Connecting E-Ink Readers via OPDS (Onyx Boox / KOReader)
1. Open your reader application (e.g., **KOReader** or **NeoReader** on Onyx Boox).
2. Navigate to the **OPDS Catalog** section and tap **Add Catalog**.
3. Configure the feed parameters:
   * **Catalog Name**: `Homelab Library`
   * **Catalog URL**: `https://books.home.lab/opds`
   * Enter your user credentials.
4. Save the catalog. You can now browse shelves and download books directly into your reading app over Wi-Fi.

---

## 7. Verification & Operational Matrix

```bash
# 1. Verify container runtime health
docker ps --filter "name=bookorbit"

# 2. Check active indexing and startup logs
docker logs --tail 30 bookorbit

# 3. Test OPDS feed accessibility
curl -kI [https://books.home.lab/opds](https://books.home.lab/opds)
```

### Operational Troubleshooting Matrix

| Symptom / Event | Root Cause | Remediation Procedure |
| :--- | :--- | :--- |
| **"DB location is not valid, please enter correct path"** | `/books` volume does not contain a valid `metadata.db` SQLite file. | Ensure a pre-existing Calibre library is copied into `/mnt/storage/books` or initialize an empty database using Calibre desktop/CLI first. |
| **"E-book converter failed to start"** | Missing conversion binaries or insufficient container memory. | Verify `DOCKER_MODS=linuxserver/mods:universal-calibre` is present in the Compose manifest, and confirm the container has at least 1 GB RAM available. |
| **Upload of large PDF returns 413 Payload Too Large** | Upstream reverse proxy enforces a low default body size. | Add `client_max_body_size 500M;` to the reverse proxy configuration. |
| **Permission Denied when uploading or editing books** | UID/GID mismatch between container environment variables and host filesystem permissions. | Run `sudo chown -R 1000:1000 /mnt/storage/books` and ensure `PUID`/`PGID` match your host user ID. |
