

# Seafile Self-Hosted Setup – Homeserver

> [!summary]
> Seafile Community Edition 13 deployed on the Ubuntu homeserver using Docker Compose.
> Persistent Seafile data is stored on `/mnt/cloud`.
> Docker Compose files are maintained under `/opt/stacks/seafile`.
> Access is intended through the local hostname `drive.home.arpa`, with Nginx Proxy Manager and Tailscale providing network access.

---

## 1. Purpose

Seafile is used as the homeserver's private cloud/document synchronization platform.

Recommended uses:

- Personal documents
- Study and learning material
- PDFs
- Receipts
- Certificates
- Projects
- Server documentation
- Scripts
- Other files that need synchronization between computers and mobile devices

Do **not** use Seafile as the primary backend for:

- Jellyfin media
- Navidrome music
- Audiobookshelf audiobooks
- BookOrbit/Kavita book libraries
- Docker volumes
- Databases

Those applications should continue using normal filesystem directories.

---

# 2. Homeserver Environment

## Host

- OS: Ubuntu Server 24.04.4 LTS
- Docker installed
- Docker Compose available
- Docker stacks stored under:

```text
/opt/stacks/
````

## Storage

Seafile uses:

```text
/mnt/cloud
```

for persistent storage.

The Seafile application stack itself is:

```text
/opt/stacks/seafile/
```

---

# 3. Final Seafile Storage Architecture

The intended directory structure is:

```text
/mnt/cloud/Seafile/
├── data/
└── mysql/
```

### Seafile application data

```text
/mnt/cloud/Seafile/data
```

is mounted inside the Seafile container as:

```text
/shared
```

### MariaDB data

```text
/mnt/cloud/Seafile/mysql
```

is mounted inside the MariaDB container as:

```text
/var/lib/mysql
```

> [!warning]
> Never manually create, rename, move, or delete files inside `/mnt/cloud/Seafile/data`.
>
> Seafile uses an internal object/block storage structure rather than storing libraries as ordinary human-readable directories.

---

# 4. Create Directories

Run:

```bash
sudo mkdir -p /opt/stacks/seafile
sudo mkdir -p /mnt/cloud/Seafile
sudo mkdir -p /mnt/cloud/Seafile/data
sudo mkdir -p /mnt/cloud/Seafile/mysql
```

Set ownership:

```bash
sudo chown -R $USER:$USER /opt/stacks/seafile
sudo chown -R $USER:$USER /mnt/cloud/Seafile
```

Verify:

```bash
ls -ld /opt/stacks/seafile
ls -ld /mnt/cloud/Seafile
ls -ld /mnt/cloud/Seafile/data
ls -ld /mnt/cloud/Seafile/mysql
```

---

# 5. Download Official Seafile Docker Files

Change directory:

```bash
cd /opt/stacks/seafile
```

Download the official Seafile Community Edition environment file:

```bash
wget -O .env https://manual.seafile.com/13.0/repo/docker/ce/env
```

Download the official Compose file:

```bash
wget https://manual.seafile.com/13.0/repo/docker/ce/seafile-server.yml
```

Rename the Compose file:

```bash
mv seafile-server.yml docker-compose.yml
```

Final structure:

```text
/opt/stacks/seafile/
├── docker-compose.yml
└── .env
```

> [!important]
> Always prefer the current official Seafile documentation when reinstalling because Seafile environment variables and deployment files can change between releases.

---

# 6. Seafile Docker Compose Configuration

The deployment uses:

* Seafile Community Edition 13
* MariaDB 10.11
* Redis
* Seafile Go File Server
* Seafile internal Nginx

The important storage mappings are:

```yaml
services:

  db:
    volumes:
      - /mnt/cloud/Seafile/mysql:/var/lib/mysql

  seafile:
    volumes:
      - /mnt/cloud/Seafile/data:/shared
```

---

# 7. Port Configuration

The Seafile container's internal web server listens on port `80`.

The host publishes it on port `8083`.

```yaml
ports:
  - "8083:80"
```

The Seafile Go File Server listens internally on:

```text
8082
```

The internal Seafile Nginx is normally responsible for handling `/seafhttp` requests and forwarding them to the Go File Server.

Therefore, the preferred configuration is:

```yaml
ports:
  - "8083:80"
```

rather than unnecessarily exposing:

```yaml
- "8082:8082"
```

> [!important]
> During troubleshooting we considered exposing `8082:8082`, but the preferred final architecture is to keep the Go File Server internal and let Seafile's Nginx handle `/seafhttp`.

---

# 8. Important `.env` Configuration

The important variables are:

```env
TIME_ZONE=Asia/Kathmandu

SEAFILE_SERVER_PROTOCOL=http

SEAFILE_SERVER_HOSTNAME=drive.home.arpa

INIT_SEAFILE_ADMIN_EMAIL=YOUR_ADMIN_EMAIL

INIT_SEAFILE_ADMIN_PASSWORD=YOUR_STRONG_PASSWORD

SEAFILE_MYSQL_DB_PASSWORD=YOUR_DATABASE_PASSWORD

INIT_SEAFILE_MYSQL_ROOT_PASSWORD=YOUR_ROOT_DATABASE_PASSWORD

REDIS_PASSWORD=YOUR_REDIS_PASSWORD

JWT_PRIVATE_KEY=YOUR_GENERATED_JWT_KEY
```

Generate the JWT key with:

```bash
openssl rand -hex 32
```

Copy the generated value into:

```env
JWT_PRIVATE_KEY=
```

---

# 9. Seafile Hostname

The final preferred hostname is:

```text
drive.home.arpa
```

The important setting is:

```env
SEAFILE_SERVER_HOSTNAME=drive.home.arpa
```

The reason for using a single canonical hostname is important.

Initially Seafile was configured with:

```text
192.168.1.20:8083
```

while remote access was being performed using:

```text
100.83.21.64:8083
```

This creates two different addresses for the same Seafile installation.

Seafile generates URLs for file operations such as uploads based on its configured server hostname.

Therefore, using a single canonical hostname avoids problems where:

```text
Browser address
        ≠
Seafile generated upload address
```

---

# 10. Local DNS

`drive.home.arpa` should resolve to:

```text
192.168.1.20
```

using the existing local DNS/AdGuard Home setup.

Test from a client:

```bash
nslookup drive.home.arpa
```

Expected:

```text
Name:    drive.home.arpa
Address: 192.168.1.20
```

---

# 11. Nginx Proxy Manager

The intended architecture is:

```text
Client
  |
  | http://drive.home.arpa
  |
  v
Nginx Proxy Manager
  |
  | 192.168.1.20:8083
  |
  v
Seafile
  |
  | internal /seafhttp
  |
  v
Go File Server :8082
```

Create a Proxy Host in Nginx Proxy Manager:

```text
Domain Names:
drive.home.arpa

Scheme:
http

Forward Hostname/IP:
192.168.1.20

Forward Port:
8083
```

Do not expose the internal Go File Server unnecessarily.

---

# 12. Tailscale Access

Tailscale is used for remote access to the homeserver.

The Tailscale IP observed during setup was:

```text
100.83.21.64
```

Initially Seafile was accessed remotely using:

```text
http://100.83.21.64:8083/
```

However, the preferred setup is to use:

```text
http://drive.home.arpa
```

as the canonical address.

Tailscale should provide access to the homeserver while DNS resolution continues to resolve:

```text
drive.home.arpa
```

to:

```text
192.168.1.20
```

The goal is that the same hostname works whether connected locally or remotely through Tailscale.

---

# 13. Deploying with Docker Compose

Unlike creating the stack manually in Portainer, this installation uses the existing Compose files directly.

This is consistent with the homeserver's existing structure:

```text
/opt/stacks/
```

Deploy with:

```bash
cd /opt/stacks/seafile
docker compose pull
docker compose up -d
```

Check containers:

```bash
docker compose ps
```

Expected containers:

```text
seafile
seafile-mysql
seafile-redis
```

---

# 14. Monitor Logs

View all logs:

```bash
docker compose logs -f
```

View only Seafile:

```bash
docker compose logs -f seafile
```

Exit log viewing with:

```text
Ctrl+C
```

This does not stop the containers.

---

# 15. Verify Containers

Run:

```bash
docker ps
```

or:

```bash
docker compose ps
```

The Seafile container should be running.

MariaDB should also be running and healthy.

Redis should be running.

---

# 16. First Login

Once initialization is complete, access:

```text
http://drive.home.arpa
```

Log in using:

```text
INIT_SEAFILE_ADMIN_EMAIL
```

and:

```text
INIT_SEAFILE_ADMIN_PASSWORD
```

from `.env`.

---

# 17. Recommended Seafile Library Structure

Separate libraries were created rather than placing everything into one giant library.

Suggested structure:

```text
Personal
├── Identity
├── Certificates
├── Finance
├── Insurance
├── Receipts
└── Other

Studies
├── Courses
├── Notes
├── Books
├── Assignments
├── Research
└── Learning

Projects
├── Homeserver
├── Docker
├── Scripts
└── Documentation

Archive
```

The exact libraries can be adjusted according to actual usage.

---

# 18. Why Separate Libraries?

A Seafile library is more than a normal folder.

Separate libraries make it easier to:

* synchronize only selected data
* share selected collections
* manage permissions
* use encryption selectively
* manage data independently
* keep personal and study material separate

For example:

```text
Laptop
  ├── Studies
  └── Projects

Phone
  └── Personal

Desktop
  ├── Personal
  ├── Studies
  └── Projects
```

---

# 19. Existing Files on `/mnt/cloud`

Existing files should **not** be copied directly into:

```text
/mnt/cloud/Seafile/data
```

This directory is Seafile's internal storage.

Do not do:

```bash
cp -r /mnt/cloud/oldfolder/* /mnt/cloud/Seafile/data/
```

Do not manually manipulate Seafile's internal data structure.

Instead, use:

* Seafile web interface
* Seafile desktop client
* Seafile Drive client
* Seafile mobile app
* supported Seafile import/API mechanisms

For smaller collections, the web interface is easiest.

For large collections, the desktop client or a proper server-side migration method is preferable.

---

# 20. Seafile File Storage Model

If a library contains:

```text
My Library
└── Test Folder
    └── example.pdf
```

you should NOT expect to find:

```text
/mnt/cloud/Seafile/data/My Library/Test Folder/example.pdf
```

Instead, Seafile stores data internally as objects/blocks and uses its database and metadata to reconstruct the library.

Therefore:

```text
Seafile library
        ↓
Seafile internal storage
        ↓
Database + object/block storage
```

rather than:

```text
Seafile library
        ↓
ordinary filesystem folder
```

---

# 21. Upload Failure Troubleshooting

## Problem encountered

The web interface worked:

* login worked
* libraries could be created
* folders could be created

but uploading a file resulted in:

```text
Upload failed
Connection refused
```

The browser showed a URL similar to:

```text
http://192.168.1.20:8083/seafhttp/upload-aj/...
```

---

# 22. Investigating the Go File Server

Seafile uses a Go File Server for file transfers.

Check its configuration:

```bash
docker exec seafile cat /shared/seafile/conf/seafile.conf
```

The following was observed:

```ini
[fileserver]
port=8082
```

This is normal.

The important distinction is:

```text
Host port:
8083

Seafile internal Go File Server:
8082
```

The Go File Server does not necessarily need to be exposed directly to clients.

---

# 23. Check Seafile's Upload Endpoint

Check:

```bash
docker exec seafile grep FILE_SERVER_ROOT /shared/seafile/conf/seahub_settings.py
```

This helps determine what URL Seafile is generating for file uploads.

The canonical hostname should correspond to:

```text
drive.home.arpa
```

rather than:

```text
192.168.1.20
```

or:

```text
100.83.21.64
```

when using the final hostname-based architecture.

---

# 24. Check Listening Ports

Inside the container:

```bash
docker exec seafile ss -tlnp
```

Expected relevant listeners include:

```text
:80
:8082
```

Port `80` is the internal web server.

Port `8082` is the internal Go File Server.

---

# 25. Check Seafile Logs

Search for file-server errors:

```bash
docker logs seafile | grep -Ei "error|failed|fileserver|seafhttp"
```

---

# 26. Test Seafile Internally

Test the internal web endpoint:

```bash
docker exec seafile curl -I http://127.0.0.1/seafhttp/
```

A response such as `404` or `403` can still indicate that the service is reachable.

The important failure is:

```text
Connection refused
```

or an inability to connect.

---

# 27. YAML Troubleshooting

During configuration, a YAML indentation error occurred.

The incorrect configuration was:

```yaml
  seafile:
    image: ...
    container_name: seafile
    restart: unless-stopped

ports:
      - "8083:80"
      - "8082:8082"
volumes:
      - /mnt/cloud/Seafile/data:/shared
```

`ports:` and `volumes:` were outside the `seafile:` service.

Correct structure:

```yaml
  seafile:
    image: ...
    container_name: seafile
    restart: unless-stopped

    ports:
      - "8083:80"
      - "8082:8082"

    volumes:
      - /mnt/cloud/Seafile/data:/shared

    environment:
      ...
```

Always validate the Compose file before deployment:

```bash
docker compose config
```

If validation succeeds:

```bash
docker compose up -d
```

---

# 28. Recreating the Stack After Configuration Changes

If `.env` or Compose configuration changes after initial deployment:

```bash
cd /opt/stacks/seafile
docker compose down
docker compose up -d
```

> [!danger]
> Do NOT use:
>
> ```bash
> docker compose down -v
> ```
>
> unless you deliberately intend to remove Docker volumes.
>
> Existing Seafile data is stored under `/mnt/cloud/Seafile`, so preserving the host-mounted directories is critical.

---

# 29. Recommended Final Networking Architecture

The clean target architecture is:

```text
                         ┌──────────────────────┐
                         │      Internet        │
                         └──────────┬───────────┘
                                    │
                                 Tailscale
                                    │
                         ┌──────────▼───────────┐
                         │     Homeserver       │
                         │    192.168.1.20      │
                         └──────────┬───────────┘
                                    │
                           drive.home.arpa
                                    │
                         ┌──────────▼───────────┐
                         │ Nginx Proxy Manager  │
                         │                      │
                         │ → 192.168.1.20:8083 │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │       Seafile        │
                         │      port 80         │
                         └──────────┬───────────┘
                                    │
                            internal /seafhttp
                                    │
                         ┌──────────▼───────────┐
                         │   Go File Server     │
                         │      port 8082       │
                         └──────────────────────┘
```

Canonical client URL:

```text
http://drive.home.arpa
```

---

# 30. Backup Considerations

Seafile data consists of both:

1. application/library data
2. database data

The important locations in this deployment are:

```text
/mnt/cloud/Seafile/data
/mnt/cloud/Seafile/mysql
/opt/stacks/seafile
```

The Compose files and `.env` are stored under:

```text
/opt/stacks/seafile
```

The `.env` contains credentials and secrets.

> [!warning]
> Treat `.env` as sensitive. Do not publish it or store it in a public Git repository.

For a robust backup strategy, database dumps should also be considered rather than relying solely on copying a live MariaDB data directory.

---

# 31. Seafile WebDAV

Seafile can expose files through WebDAV.

Useful applications include:

* document managers
* backup/synchronization tools
* compatible office applications
* Obsidian synchronization workflows
* Joplin
* PDF/document applications
* other software supporting WebDAV

WebDAV should generally be treated as a **client access mechanism**, not as a replacement for a local filesystem.

---

# 32. BookOrbit and Seafile

Do not use Seafile WebDAV as the primary filesystem backend for BookOrbit.

BookOrbit should preferably read books from a normal local filesystem directory.

Recommended architecture:

```text
/mnt/cloud/Books
        ↓
     BookOrbit
```

while:

```text
Seafile
   ↓
Personal documents
Study material
Projects
Notes
```

Reading progress is normally managed by the book-management application's own database rather than being synchronized by merely putting EPUB/PDF files on WebDAV.

---

# 33. Recommended Use of Seafile in This Homeserver

Seafile becomes the private cloud/document synchronization layer:

```text
                         Homeserver

       ┌──────────────────────────────────────┐
       │                                      │
       │  Immich       → Photos               │
       │  Jellyfin     → Movies/TV            │
       │  Navidrome    → Music                │
       │  Audiobookshelf → Audiobooks         │
       │  BookOrbit    → Books                │
       │  Seafile      → Documents/Files      │
       │                                      │
       └──────────────────────────────────────┘
```

This separation is intentional.

Each application manages the type of content it is designed for.

---

# 34. Useful Docker Commands

## Enter Seafile directory

```bash
cd /opt/stacks/seafile
```

## Check configuration

```bash
docker compose config
```

## Pull updated images

```bash
docker compose pull
```

## Start

```bash
docker compose up -d
```

## Stop

```bash
docker compose down
```

## Restart

```bash
docker compose restart
```

## View status

```bash
docker compose ps
```

## View logs

```bash
docker compose logs -f
```

## Seafile logs only

```bash
docker compose logs -f seafile
```

## Database logs

```bash
docker compose logs -f db
```

## Redis logs

```bash
docker compose logs -f redis
```

## View all Docker containers

```bash
docker ps
```

---

# 35. Useful Diagnostic Commands

Check Seafile configuration:

```bash
docker exec seafile cat /shared/seafile/conf/seafile.conf
```

Check upload root:

```bash
docker exec seafile grep FILE_SERVER_ROOT /shared/seafile/conf/seahub_settings.py
```

Check processes:

```bash
docker exec seafile ps -ef
```

Check ports:

```bash
docker exec seafile ss -tlnp
```

Check Seafile errors:

```bash
docker logs seafile | grep -Ei "error|failed|fileserver|seafhttp"
```

Check storage:

```bash
df -h /mnt/cloud
```

Check Seafile storage usage:

```bash
du -sh /mnt/cloud/Seafile
```

Check individual directories:

```bash
du -sh /mnt/cloud/Seafile/*
```

---

# 36. Important Lessons

## Use one canonical hostname

Prefer:

```text
drive.home.arpa
```

rather than switching between:

```text
192.168.1.20:8083
```

and:

```text
100.83.21.64:8083
```

This is especially important for Seafile upload URLs and other generated links.

---

## Don't manipulate Seafile's internal storage

Never treat:

```text
/mnt/cloud/Seafile/data
```

as an ordinary shared folder.

Use Seafile itself to add and manage files.

---

## Keep Docker configuration separate from data

```text
/opt/stacks/seafile
        ↓
Compose configuration

/mnt/cloud/Seafile
        ↓
Persistent Seafile data
```

This makes the deployment easier to recreate.

---

## Validate YAML before deploying

Always run:

```bash
docker compose config
```

before:

```bash
docker compose up -d
```

---

## Don't expose unnecessary ports

Preferred:

```yaml
ports:
  - "8083:80"
```

The internal Go File Server on `8082` should normally remain behind Seafile's internal Nginx.

---

# 37. Reinstallation Checklist

For a future reinstall:

```bash
# 1. Confirm storage
df -h /mnt/cloud

# 2. Create stack directory
sudo mkdir -p /opt/stacks/seafile

# 3. Create persistent storage
sudo mkdir -p /mnt/cloud/Seafile/data
sudo mkdir -p /mnt/cloud/Seafile/mysql

# 4. Enter stack
cd /opt/stacks/seafile

# 5. Download official Seafile files
wget -O .env https://manual.seafile.com/13.0/repo/docker/ce/env
wget https://manual.seafile.com/13.0/repo/docker/ce/seafile-server.yml

# 6. Rename compose file
mv seafile-server.yml docker-compose.yml

# 7. Edit .env
nano .env

# 8. Edit compose file
nano docker-compose.yml

# 9. Validate
docker compose config

# 10. Pull images
docker compose pull

# 11. Start
docker compose up -d

# 12. Verify
docker compose ps

# 13. View logs
docker compose logs -f seafile
```

Configure:

```env
TIME_ZONE=Asia/Kathmandu
SEAFILE_SERVER_PROTOCOL=http
SEAFILE_SERVER_HOSTNAME=drive.home.arpa
```

and ensure the Compose volume mappings point to:

```text
/mnt/cloud/Seafile/mysql
/mnt/cloud/Seafile/data
```

---

# 38. Final Target State

```text
/opt/stacks/seafile/
├── docker-compose.yml
└── .env


/mnt/cloud/Seafile/
├── data/
└── mysql/
```

Access:

```text
http://drive.home.arpa
```

Docker services:

```text
seafile
seafile-mysql
seafile-redis
```

Network path:

```text
LAN/Tailscale
      ↓
drive.home.arpa
      ↓
Nginx Proxy Manager
      ↓
192.168.1.20:8083
      ↓
Seafile
      ↓
internal Go File Server :8082
```

Primary purpose:

```text
Personal files
Study material
Documents
Projects
Notes
Archives
```

Keep media and application-specific libraries outside Seafile.

```

This note reflects the **deployment and troubleshooting decisions from our conversation**, including the hostname transition to `drive.home.arpa`, the `/mnt/cloud` storage layout, Docker Compose deployment, and the Seafile upload issue.
```
