Based on the Docker stack structure we've built for your home server so far, I recommend keeping BookOrbit consistent with your existing layout.

Your current architecture is roughly:

* `/opt/stacks/` → Docker Compose files
* `/mnt/media/` → Movies, TV Shows, Music, Audiobooks
* `/mnt/cloud/` → Important files, Immich, backups
* Docker managed through Portainer

Since you already have **Kavita** installed for ebooks and **Audiobookshelf** for audiobooks, BookOrbit can either:

* **replace both** (if you like its interface better), or
* **run alongside them** using the exact same library folders (recommended initially while evaluating it). BookOrbit scans your existing library without moving files. ([BookOrbit][1])

# Step 1. Create the stack directory

```bash
sudo mkdir -p /opt/stacks/bookorbit
cd /opt/stacks/bookorbit
```

---

# Step 2. Create persistent folders

I would use the following structure.

```text
/opt/stacks/bookorbit
│
├── docker-compose.yml
├── .env
├── data/
│   ├── app
│   └── postgres
```

Create them:

```bash
mkdir -p data/app
mkdir -p data/postgres
```

---

# Step 3. Use your existing media folders

Since you already have libraries, don't duplicate them.

Example:

```text
/mnt/media
│
├── Books
│     ├── Fiction
│     ├── Non Fiction
│
├── Audiobooks
│
├── Movies
├── Music
└── TV Shows
```

BookOrbit will mount these folders read-only or read-write depending on your preference.

---

# Step 4. Create the environment file

Generate secure secrets:

```bash
openssl rand -hex 24
openssl rand -hex 32
openssl rand -hex 16
```

Create:

```bash
nano .env
```

Example:

```env
APP_IMAGE=ghcr.io/bookorbit/bookorbit:latest

APP_URL=http://192.168.1.20:3000

APP_PORT=3000

POSTGRES_USER=bookorbit
POSTGRES_DB=bookorbit
POSTGRES_PASSWORD=YOUR_RANDOM_PASSWORD

JWT_SECRET=YOUR_RANDOM_SECRET

SETUP_BOOTSTRAP_TOKEN=YOUR_RANDOM_TOKEN

BOOKS_HOST_PATH=/mnt/media

PUID=1000
PGID=1000
```

These are the minimum required settings. ([BookOrbit][2])

---

# Step 5. Create docker-compose.yml

```yaml
services:

  postgres:
    image: pgvector/pgvector:pg16
    container_name: bookorbit-db

    restart: unless-stopped

    env_file:
      - .env

    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}

    volumes:
      - ./data/postgres:/var/lib/postgresql/data

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5


  bookorbit:
    image: ${APP_IMAGE}

    container_name: bookorbit

    restart: unless-stopped

    depends_on:
      postgres:
        condition: service_healthy

    env_file:
      - .env

    ports:
      - "3000:3000"

    environment:
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}

    volumes:
      - ./data/app:/data
      - /mnt/media:/books
```

This follows BookOrbit's recommended Docker deployment using a bundled PostgreSQL database. ([BookOrbit][2])

---

# Step 6. Start the stack

```bash
docker compose pull

docker compose up -d
```

Verify:

```bash
docker ps
```

You should see

```
bookorbit
bookorbit-db
```

---

# Step 7. Open the web interface

```
http://192.168.1.20:3000
```

You'll be asked for:

* Administrator account
* Bootstrap token (from `.env`)

Complete the initial setup. ([BookOrbit][2])

---

# Step 8. Create your libraries

Create separate libraries rather than mixing everything together.

### Ebooks

```
Name:
Books

Folder:
/books/Books
```

### Audiobooks

```
Name:
Audiobooks

Folder:
/books/Audiobooks
```

When selecting folders in BookOrbit, use the **container path** (e.g. `/books/Books`), not the host path like `/mnt/media/Books`. ([BookOrbit][3])

---

# Step 9. Verify permissions

Run:

```bash
ls -ld /mnt/media
```

and

```bash
id
```

Ensure your user is UID/GID `1000` (or adjust `PUID`/`PGID` accordingly). BookOrbit needs read access to the mounted library and write access if you want it to upload or modify metadata. ([BookOrbit][2])

---

# Port Summary

| Service    | Port          |
| ---------- | ------------- |
| BookOrbit  | 3000          |
| PostgreSQL | Internal only |

---

## How this fits into your home server

Your Docker stack would become:

```
/opt/stacks
├── immich
├── jellyfin
├── navidrome
├── qbittorrent
├── audiobookshelf
├── kavita
├── bookorbit
└── portainer
```

and your media layout:

```
/mnt/media
├── Movies
├── TV Shows
├── Music
├── Books
├── Audiobooks
└── Comics
```

This integrates cleanly with the architecture we've been building for your server, keeps all media centralized under `/mnt/media`, and maintains the same `/opt/stacks/<service>` pattern for Docker Compose management. Once you've confirmed BookOrbit is working well for your workflow, we can also place it behind your reverse proxy (alongside your other services) and optionally enable HTTPS and remote access through Tailscale or your chosen proxy solution.

[1]: https://bookorbit.app/adding-books?utm_source=chatgpt.com "Adding Books | BookOrbit"
[2]: https://bookorbit.app/installation?utm_source=chatgpt.com "Installation | BookOrbit"
[3]: https://bookorbit.app/creating-a-library?utm_source=chatgpt.com "Creating a Library | BookOrbit"
