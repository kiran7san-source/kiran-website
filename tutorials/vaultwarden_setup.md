Given current `homeserver` architecture, One would deploy Vaultwarden as a **small, isolated Docker stack**, keep it **Tailscale/LAN-only**, put it behind your existing **Nginx Proxy Manager + AdGuard**, and include it in your existing **Restic configuration backup**.

The important principle is:

> **Do not expose Vaultwarden directly to the Internet.** Your password vault should be one of the most protected services on the server.

Vaultwarden's upstream documentation supports Docker Compose with persistent `/data` storage, and the current image is `vaultwarden/server`. ([GitHub][1])

## Target architecture

For your server, I'd use:

```text
                         YOUR DEVICES
                ┌──────────┼──────────┐
                │          │          │
             Windows     MacBook    Phone
                │          │          │
                └──────────┼──────────┘
                           │
                       Tailscale
                           │
                           ▼
                    100.83.21.64
                           │
                           ▼
                  AdGuard Home DNS
                           │
              vault.home.arpa
                           │
                           ▼
                Nginx Proxy Manager
                     HTTPS :443
                           │
                           ▼
                    Vaultwarden
                     Docker :80
                           │
                           ▼
             /opt/stacks/vaultwarden/data
                           │
                           ▼
                 Restic homeserver backup
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
        /mnt/cloud/...             External HDD
```

Your users will never need to remember a port.

You'll simply use:

**`https://vault.home.arpa`**

---

# 1. Before we start

There are three things I would deliberately **not** do:

### Don't expose port 80/443 from Vaultwarden itself

NPM already owns:

```text
80
81
443
```

So Vaultwarden doesn't need a host port.

### Don't put Vaultwarden's data under `/mnt/media`

Your media disk is for media.

### Don't put the Vaultwarden database on `/mnt/cloud`

Although `/mnt/cloud` is a good data disk, your existing Restic architecture already backs up configuration/application data from the root filesystem.

For this particular service, I'd keep:

```text
/opt/stacks/vaultwarden/
```

and make its persistent data:

```text
/opt/stacks/vaultwarden/data/
```

That also makes disaster recovery straightforward.

---

# 2. Create the stack

SSH into your homeserver:

```bash
ssh homeserver
```

Then:

```bash
sudo mkdir -p /opt/stacks/vaultwarden/data
cd /opt/stacks/vaultwarden
```

Check:

```bash
pwd
```

You should get:

```text
/opt/stacks/vaultwarden
```

---

# 3. Create the `.env`

We'll keep secrets out of the Compose file.

```bash
sudo nano /opt/stacks/vaultwarden/.env
```

Put:

```dotenv
DOMAIN=https://vault.home.arpa

SIGNUPS_ALLOWED=true

WEBSOCKET_ENABLED=true

TZ=Asia/Kathmandu
```

### Why `SIGNUPS_ALLOWED=true` initially?

Because you need to create your first account.

After creating your account, we'll change it to:

```dotenv
SIGNUPS_ALLOWED=false
```

This is important.

Otherwise anyone who can reach the Vaultwarden service could potentially create an account.

Vaultwarden supports `SIGNUPS_ALLOWED` specifically for controlling registration. ([GitHub][2])

---

# 4. Create Docker Compose

Create:

```bash
sudo nano /opt/stacks/vaultwarden/compose.yaml
```

Use:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped

    environment:
      DOMAIN: ${DOMAIN}
      SIGNUPS_ALLOWED: ${SIGNUPS_ALLOWED}
      WEBSOCKET_ENABLED: ${WEBSOCKET_ENABLED}
      TZ: ${TZ}

    volumes:
      - ./data:/data

    networks:
      - vaultwarden

networks:
  vaultwarden:
    name: vaultwarden
```

Notice something important:

```yaml
networks:
  - vaultwarden
```

There is **no `ports:` section**.

That is intentional.

Vaultwarden is not directly accessible from your LAN.

NPM will eventually connect to it through Docker networking.

---

# 5. Check the Compose configuration

Before starting anything:

```bash
cd /opt/stacks/vaultwarden
sudo docker compose config
```

Look for errors.

Then:

```bash
sudo docker compose pull
```

Start it:

```bash
sudo docker compose up -d
```

Check:

```bash
sudo docker ps
```

You should see something like:

```text
vaultwarden   vaultwarden/server:latest   Up ...
```

Check the logs:

```bash
sudo docker compose logs --tail=50
```

You want to see Vaultwarden starting normally without repeated errors.

---

# 6. Connect Vaultwarden to Nginx Proxy Manager

This is the one part that requires a small change to your existing NPM arrangement.

Your NPM container needs to be able to communicate with the Vaultwarden container.

First inspect your NPM stack:

```bash
docker inspect nginx-proxy-manager
```

Look for its Docker networks.

You can simplify this by connecting NPM to the Vaultwarden network:

```bash
docker network connect vaultwarden nginx-proxy-manager
```

But **check the actual NPM container name first**:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}'
```

If your NPM container is named something different, substitute that name.

For example:

```bash
docker network connect vaultwarden nginx-proxy-manager
```

Verify:

```bash
docker inspect nginx-proxy-manager \
  --format '{{json .NetworkSettings.Networks}}'
```

You should now see both NPM's existing network and:

```text
vaultwarden
```

### Why do this?

Now:

```text
NPM
 │
 │ Docker network
 ▼
vaultwarden:80
```

No host port is required.

---

# 7. Configure AdGuard

You already use AdGuard DNS rewrites.

Add:

```text
vault.home.arpa → 100.83.21.64
```

In AdGuard Home:

**Filters → DNS rewrites**

Add:

```text
Domain:
vault.home.arpa

IP address:
100.83.21.64
```

Then test from a client.

On Windows:

```powershell
nslookup vault.home.arpa
```

You want:

```text
Name:    vault.home.arpa
Address: 100.83.21.64
```

On Linux/macOS:

```bash
nslookup vault.home.arpa
```

---

# 8. Configure Nginx Proxy Manager

Open your existing NPM:

```text
https://npm.home.arpa
```

or whatever hostname you're currently using for NPM.

Create:

**Proxy Hosts → Add Proxy Host**

### Details

**Domain Names**

```text
vault.home.arpa
```

**Scheme**

```text
http
```

**Forward Hostname/IP**

```text
vaultwarden
```

**Forward Port**

```text
80
```

Enable:

```text
Block Common Exploits
```

and:

```text
Websockets Support
```

Vaultwarden's current configuration supports WebSockets through the reverse proxy; modern deployments don't need the old separate `3012` host-port arrangement. ([GitHub][3])

---

# 9. SSL certificate

You already have:

```text
Kiran Homeserver Local CA
```

and:

```text
*.home.arpa
```

So select your existing wildcard certificate.

Use:

```text
SSL Certificate:
*.home.arpa
```

Then enable:

```text
Force SSL
HTTP/2 Support
```

Don't request a Let's Encrypt certificate for this.

This is an internal `.home.arpa` service.

Your final NPM configuration should effectively be:

```text
vault.home.arpa
       │
       │ HTTPS
       ▼
NPM :443
       │
       │ HTTP
       ▼
vaultwarden:80
```

---

# 10. Test Vaultwarden

From your computer:

```text
https://vault.home.arpa
```

You should get the Vaultwarden web interface.

You should **not** get:

```text
http://192.168.1.20:xxxx
```

and you shouldn't need any port number.

---

# 11. Create your account

Now register your first account.

Use a strong, unique master password.

This password is extremely important.

Your master password should **not** be reused anywhere else.

Once your account is created, immediately go back to:

```bash
sudo nano /opt/stacks/vaultwarden/.env
```

Change:

```dotenv
SIGNUPS_ALLOWED=true
```

to:

```dotenv
SIGNUPS_ALLOWED=false
```

Then:

```bash
cd /opt/stacks/vaultwarden
sudo docker compose up -d
```

Check:

```bash
docker compose ps
```

Now new registrations are disabled.

---

# 12. Do we need the Vaultwarden admin page?

For your setup, **I'd initially say no**.

Vaultwarden has an `/admin` interface, but this is another sensitive attack surface.

You don't need it for normal password-manager operation.

So I would **not configure `ADMIN_TOKEN` initially**.

If later you need advanced server configuration, we can add it and restrict access appropriately.

---

# 13. Configure your Bitwarden clients

This is one of the biggest advantages of Vaultwarden.

You can use the normal Bitwarden clients.

On the client, choose:

**Self-hosted**

Then set the server URL to:

```text
https://vault.home.arpa
```

Do this on:

* Windows
* MacBook
* Android/iPhone
* Browser extensions

The client should synchronize with your Vaultwarden server.

---

# 14. VERY IMPORTANT — Tailscale access

Your current Tailscale address is:

```text
100.83.21.64
```

and your AdGuard rewrite points:

```text
vault.home.arpa
        ↓
100.83.21.64
```

That means devices connected through your Tailscale network can access the service.

This is exactly what I want for your password manager.

You can therefore have:

```text
Home LAN
   │
   └── vault.home.arpa
           │
           ▼
     100.83.21.64


Away from home
   │
   └── Tailscale
           │
           ▼
     vault.home.arpa
           │
           ▼
     100.83.21.64
```

No public port forwarding.

No public DNS.

No Cloudflare tunnel required.

No Internet-facing Vaultwarden.

---

# 15. Verify there is no exposed Vaultwarden port

Run:

```bash
docker ps
```

The Vaultwarden line should **not** show something like:

```text
0.0.0.0:8080->80/tcp
```

That's important.

You want:

```text
vaultwarden/server:latest
```

with no published host port.

Check explicitly:

```bash
docker port vaultwarden
```

Ideally it returns nothing.

---

# 16. Check your firewall situation

You can also verify listening ports:

```bash
sudo ss -tulpn
```

You should see your existing:

```text
80
443
81
```

from NPM, but there should not be a new Vaultwarden host port.

---

# 17. Vaultwarden data

Your important data is now:

```text
/opt/stacks/vaultwarden/data/
```

Check it:

```bash
sudo ls -lah /opt/stacks/vaultwarden/data
```

You'll eventually see things such as:

```text
db.sqlite3
attachments/
sends/
rsa_key.pem
rsa_key.pub.der
...
```

The data directory contains the SQLite database, server keys and other important Vaultwarden data. For a complete disaster-recovery backup, the entire data directory should be preserved rather than only the database. ([GitHub][4])

---

# 18. Backup strategy for YOUR server

This is the most important section.

You already have:

```text
Restic
    ↓
/mnt/cloud/homeserverbkp/restic
```

and an external:

```text
/mnt/backup-inspect
```

So I would integrate Vaultwarden into the backup system you already built rather than introduce another backup application.

Your hierarchy becomes:

```text
Vaultwarden
    │
    ▼
/opt/stacks/vaultwarden
    │
    ├── compose.yaml
    ├── .env
    └── data/
          │
          ├── db.sqlite3
          ├── attachments/
          ├── sends/
          └── rsa keys
                    │
                    ▼
                 Restic
                    │
                    ▼
             /mnt/cloud/
             homeserverbkp/
             restic
                    │
                    ▼
             External HDD
```

---

# 19. Check your existing Restic backup

Because your existing backup plan already includes `/opt/stacks`, Vaultwarden should naturally fall inside it.

Check your backup script:

```bash
sudo grep -n "/opt/stacks" /usr/local/sbin/backup-homeserver
```

If your script uses:

```text
/opt/stacks
```

as a recursive backup source, you're covered.

But there's one thing I want you to verify:

```bash
sudo grep -n "vaultwarden" /usr/local/sbin/backup-homeserver
```

If nothing appears, that's okay — it may be covered by `/opt/stacks`.

---

# 20. I would add an explicit Vaultwarden backup

Because this is your password manager, I would make the backup intention obvious.

Create:

```bash
sudo mkdir -p /opt/stacks/vaultwarden/backups
```

But **don't actually use this as your primary backup**.

Your Restic repository should remain the real backup.

The local directory can be useful for a short-term recovery snapshot.

---

# 21. Test Restic

Run your existing backup:

```bash
sudo /usr/local/sbin/backup-homeserver
```

Then check:

```bash
sudo restic \
  -r /mnt/cloud/homeserverbkp/restic \
  snapshots
```

You should see a new snapshot.

---

# 22. Verify Vaultwarden exists inside the snapshot

Use Restic:

```bash
sudo restic \
  -r /mnt/cloud/homeserverbkp/restic \
  snapshots
```

Find the latest snapshot ID.

Then:

```bash
sudo restic \
  -r /mnt/cloud/homeserverbkp/restic \
  ls <SNAPSHOT_ID> \
  | grep vaultwarden
```

You want to see paths containing:

```text
opt/stacks/vaultwarden
```

This is an **extremely important test**.

Don't assume your backup contains something just because the backup script ran successfully.

---

# 23. External HDD copy

Your existing disaster-recovery strategy is even better because you have:

```text
homeserver
    │
    ▼
/mnt/cloud/homeserverbkp/restic
    │
    ▼
external HDD
```

Keep doing this.

For Vaultwarden, I'd consider the external copy **mandatory**.

The reason is simple:

If the homeserver dies and the 2TB `/mnt/cloud` disk also dies, your password manager backup should still exist somewhere else.

---

# 24. Recovery test

Once everything works, I would perform a **non-destructive restore test**.

Don't immediately restore over the live Vaultwarden.

Instead:

```text
Restic
   │
   ▼
temporary restore directory
   │
   ▼
inspect:
   db.sqlite3
   attachments/
   rsa_key.*
```

For example:

```bash
mkdir -p /tmp/vaultwarden-restore-test
```

Then:

```bash
sudo restic \
  -r /mnt/cloud/homeserverbkp/restic \
  restore latest \
  --target /tmp/vaultwarden-restore-test
```

Then:

```bash
sudo find /tmp/vaultwarden-restore-test/opt/stacks/vaultwarden \
  -maxdepth 2 \
  -type f
```

You should be able to see your Vaultwarden data.

---

# 25. Actual disaster recovery

If your Vaultwarden installation is destroyed but the rest of your homeserver survives:

```bash
cd /opt/stacks
```

Remove only the Vaultwarden installation if necessary:

```bash
sudo rm -rf /opt/stacks/vaultwarden
```

Restore it from Restic.

Then:

```bash
cd /opt/stacks/vaultwarden
```

Check:

```bash
ls -lah
```

Make sure:

```text
compose.yaml
.env
data/
```

exist.

Then:

```bash
sudo docker compose up -d
```

Your Vaultwarden database, keys and attachments should return.

The Vaultwarden project specifically recommends preserving the entire data directory for full disaster recovery, not just SQLite. ([GitHub][4])

---

# 26. One subtle but VERY important issue

Your `.env` contains configuration/secrets.

Therefore:

```text
/opt/stacks/vaultwarden/.env
```

must also be included in your Restic backup.

Check:

```bash
sudo ls -la /opt/stacks/vaultwarden/
```

You should have:

```text
.env
compose.yaml
data/
```

Your Restic backup needs all three.

---

# 27. Permissions

Don't start changing ownership unnecessarily.

Check:

```bash
sudo ls -ld /opt/stacks/vaultwarden
sudo ls -ld /opt/stacks/vaultwarden/data
```

The Docker container must be able to write to `data`.

If Vaultwarden starts and creates:

```text
db.sqlite3
```

you're fine.

---

# 28. Updating Vaultwarden

Don't blindly run:

```bash
docker compose down -v
```

**Never use `-v` here.**

That can remove Docker volumes, and although our data is bind-mounted, making destructive commands a habit around a password manager is dangerous.

Normal update:

```bash
cd /opt/stacks/vaultwarden
sudo docker compose pull
sudo docker compose up -d
```

Then:

```bash
sudo docker compose logs --tail=100
```

And test:

```text
https://vault.home.arpa
```

---

# 29. Before every major update

I'd make a Restic snapshot first:

```bash
sudo /usr/local/sbin/backup-homeserver
```

Then update:

```bash
cd /opt/stacks/vaultwarden
sudo docker compose pull
sudo docker compose up -d
```

This gives you:

```text
Backup
  ↓
Update
  ↓
Test
  ↓
Keep/restore
```

rather than:

```text
Update
  ↓
Something breaks
  ↓
Where's my backup?
```

---

# 30. Security settings I recommend

For your particular installation:

| Setting                      | Recommendation                  |
| ---------------------------- | ------------------------------- |
| Internet exposure            | **No**                          |
| Tailscale                    | **Yes**                         |
| NPM                          | **Yes**                         |
| HTTPS                        | **Yes**                         |
| `.home.arpa`                 | **Yes**                         |
| Public DNS                   | **No**                          |
| Port forwarding              | **No**                          |
| Signup                       | **Disable after first account** |
| 2FA                          | **Enable**                      |
| Strong master password       | **Mandatory**                   |
| Admin page                   | **Don't enable initially**      |
| Database                     | **SQLite**                      |
| Database backup              | **Yes**                         |
| Entire data directory backup | **Yes**                         |
| External backup              | **Yes**                         |
| Backup restore test          | **Yes**                         |

---

# 31. Your final homeserver layout

This fits nicely into the architecture you've already established:

```text
/opt/stacks/
│
├── adguardhome/
├── nginx-proxy-manager/
├── seafile/
├── bookorbit/
├── navidrome/
├── jellyfin/
├── qbittorrent/
├── metube/
├── portainer/
├── immich/
│
└── vaultwarden/
    ├── compose.yaml
    ├── .env
    └── data/
        ├── db.sqlite3
        ├── attachments/
        ├── sends/
        ├── rsa_key.pem
        └── ...
```

And access:

```text
adguard.home.arpa
npm.home.arpa
drive.home.arpa
books.home.arpa
music.home.arpa
...
vault.home.arpa   ← password manager
```

---

# 32. One change I'd make to your overall security model

Once Vaultwarden is working, I would **not store the Vaultwarden master password in Vaultwarden**.

Instead, keep the master password available through a separate recovery mechanism.

For example:

```text
Vaultwarden master password
        │
        ├── Memorized
        │
        └── Emergency offline copy
                  │
                  ▼
             Secure location
```

And your recovery information should include:

```text
Vaultwarden URL:
https://vault.home.arpa

Homeserver:
192.168.1.20

Tailscale:
100.83.21.64

Vaultwarden stack:
 /opt/stacks/vaultwarden

Backup repository:
 /mnt/cloud/homeserverbkp/restic
```

**Do not put the master password itself in your normal homeserver documentation or Obsidian notes.**

---

## The deployment sequence I'd actually follow

Don't try to do everything at once. Follow this order:

```text
1. Create /opt/stacks/vaultwarden
              ↓
2. Create .env
              ↓
3. Create compose.yaml
              ↓
4. docker compose up -d
              ↓
5. Connect NPM to Vaultwarden Docker network
              ↓
6. Add vault.home.arpa to AdGuard
              ↓
7. Configure NPM proxy host
              ↓
8. Configure *.home.arpa certificate
              ↓
9. Open https://vault.home.arpa
              ↓
10. Create your account
              ↓
11. Disable SIGNUPS_ALLOWED
              ↓
12. Enable 2FA
              ↓
13. Configure Bitwarden clients
              ↓
14. Run backup-homeserver
              ↓
15. Verify Vaultwarden appears in Restic
              ↓
16. Copy backup to external HDD
              ↓
17. Perform a test restore
```

That gives you a **properly integrated password-management service rather than simply another Docker container**.

One current upstream detail worth keeping in mind: Vaultwarden's official repository recommends the `vaultwarden/server` image and persistent `/data`; its documentation also points users to the Wiki for detailed configuration. ([GitHub][1])

If you want to proceed interactively, **start with steps 1–5 only**. Paste the output of `docker ps` and your existing NPM container/network information, and I can adapt the network connection commands to your *actual* current Docker setup before you touch NPM.

[1]: https://github.com/dani-garcia/vaultwarden/blob/main/README.md?plain=1&utm_source=chatgpt.com "vaultwarden/README.md at main · dani-garcia/vaultwarden · GitHub"
[2]: https://github.com/MNLaugh/vaultwarden?utm_source=chatgpt.com "GitHub - MNLaugh/vaultwarden: Deploy Unofficial Bitwarden compatible server written in Rust in a Docker container with Traefik as a reverse proxy and HTTP → HTTPS redirection · GitHub"
[3]: https://github.com/oehrn/Vaultwarden?utm_source=chatgpt.com "GitHub - oehrn/Vaultwarden: Self-hosted Vaultwarden compose stack · GitHub"
[4]: https://github.com/phi0x/vaultwarden/blob/main/README.md?utm_source=chatgpt.com "vaultwarden/README.md at main · phi0x/vaultwarden · GitHub"
