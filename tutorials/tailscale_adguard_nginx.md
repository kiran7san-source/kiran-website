# Homeserver — Tailscale + AdGuard Home + Nginx Proxy Manager

> [!summary] Goal
> Create a private, Tailscale-only homelab where services can be accessed using memorable names such as `qbit.home.arpa` instead of remembering Docker ports.
>
> No purchased domain name is required.
>
> Architecture:
>
> `Tailscale → AdGuard Home → DNS rewrite → Nginx Proxy Manager → Docker service`

---

## 1. Final Architecture

### Network

The homeserver is accessible through Tailscale.

**Homeserver Tailscale IP:**

```text
100.83.21.64
````

**Tailnet:**

```text
phoenix-kingsnake.ts.net
```

Therefore the homeserver's MagicDNS name is:

```text
homeserver.phoenix-kingsnake.ts.net
```

The ProDesk is another Tailscale device.

### DNS

Tailscale DNS is configured to use AdGuard Home:

```text
Tailscale device
       │
       ▼
AdGuard Home
100.83.21.64:53
```

AdGuard handles:

* DNS resolution
* DNS rewrites for homelab names
* ad/tracker blocking
* DNS query logging
* DNS caching
* upstream DNS resolution

### Friendly service names

The private namespace used is:

```text
*.home.arpa
```

Examples:

```text
qbit.home.arpa
drive.home.arpa
jellyfin.home.arpa
photos.home.arpa
music.home.arpa
books.home.arpa
audio.home.arpa
metube.home.arpa
portainer.home.arpa
```

`.home.arpa` is intended for private/home networks and does not require purchasing a domain.

---

# 2. Important Concept

There are three separate layers.

## Tailscale

Provides the private, encrypted network:

```text
Remote device
     │
     │ Tailscale
     ▼
Homeserver
100.83.21.64
```

The services do not need to be exposed to the public Internet.

## AdGuard Home

Provides DNS:

```text
qbit.home.arpa
       ↓
100.83.21.64
```

It also provides ad/tracker blocking and DNS logging.

## Nginx Proxy Manager

Provides reverse proxying:

```text
qbit.home.arpa
       ↓
Nginx Proxy Manager
       ↓
100.83.21.64:8080
       ↓
qBittorrent
```

This removes the need to remember application ports.

---

# 3. Tailscale Configuration

Check Tailscale status:

```bash
tailscale status
```

Check the Tailscale IP:

```bash
tailscale ip
```

Check DNS configuration:

```bash
tailscale dns status
```

Expected important information:

```text
MagicDNS: enabled tailnet-wide
suffix = phoenix-kingsnake.ts.net
```

and:

```text
Resolvers (in preference order):
  - 100.83.21.64
```

This confirms Tailscale is using AdGuard Home as its DNS resolver.

The homeserver should appear as:

```text
homeserver.phoenix-kingsnake.ts.net
```

---

# 4. Renaming the Tailscale Tailnet

Originally the MagicDNS suffix was:

```text
tail4cae4b.ts.net
```

The tailnet was renamed to:

```text
phoenix-kingsnake.ts.net
```

This is recommended when setting up the homelab because it makes the Tailscale hostnames easier to remember.

After renaming, verify:

```bash
tailscale status
```

and:

```bash
tailscale dns status
```

Expected:

```text
MagicDNS: enabled tailnet-wide
suffix = phoenix-kingsnake.ts.net
```

The rename did not affect:

```text
100.83.21.64
```

or the AdGuard DNS rewrites.

---

# 5. AdGuard Home Installation

AdGuard Home was installed in:

```text
/opt/stacks/adguardhome
```

Check the directory:

```bash
cd /opt/stacks/adguardhome
pwd
```

Expected:

```text
/opt/stacks/adguardhome
```

## Docker Compose

The Compose file was:

```yaml
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped

    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf

    ports:
      - "3001:3000/tcp"
      - "53:53/tcp"
      - "53:53/udp"
```

---

# 6. AdGuard Port 53 Conflict

Initially, starting AdGuard failed because port 53 was already being used by `systemd-resolved`.

Error:

```text
failed to bind host port 0.0.0.0:53/tcp:
address already in use
```

Check port 53:

```bash
sudo ss -lntup | grep ':53'
```

The important point is that `systemd-resolved` normally listens on:

```text
127.0.0.53:53
127.0.0.54:53
```

AdGuard was ultimately configured to bind DNS specifically to the Tailscale IP:

```text
100.83.21.64:53
```

This allowed both services to coexist.

Verify:

```bash
sudo ss -lntup | grep ':53'
```

Expected:

```text
100.83.21.64:53
```

for AdGuard, alongside the local `systemd-resolved` listeners.

---

# 7. Start AdGuard Home

From:

```text
/opt/stacks/adguardhome
```

start it with:

```bash
docker compose up -d
```

Check:

```bash
docker ps --filter name=adguardhome
```

The important part is that AdGuard is running and DNS is exposed as:

```text
100.83.21.64:53
```

The web interface was initially accessed through:

```text
http://homeserver.phoenix-kingsnake.ts.net:3001/
```

---

# 8. Test AdGuard DNS

From another Tailscale device, such as the ProDesk:

```bash
nslookup example.com 100.83.21.64
```

and:

```bash
nslookup google.com 100.83.21.64
```

A successful response confirms AdGuard is resolving external DNS queries.

---

# 9. Configure Tailscale to Use AdGuard

In the Tailscale DNS configuration, add:

```text
100.83.21.64
```

as the nameserver/resolver.

On the homeserver:

```bash
tailscale dns status
```

Expected:

```text
Resolvers (in preference order):
  - 100.83.21.64
```

This makes Tailscale clients use AdGuard Home for DNS.

---

# 10. Android DNS Configuration

Android initially could not resolve:

```text
qbit.home.arpa
```

The solution was to enable:

**Tailscale → Settings → Use Tailscale DNS**

After enabling this, Android could resolve the private names correctly.

If Android cannot resolve `*.home.arpa`, first check that:

* Tailscale is connected
* **Use Tailscale DNS** is enabled
* Android is not overriding DNS with an incompatible custom Private DNS provider

---

# 11. AdGuard DNS Rewrites

DNS rewrites are used to make all friendly names point to the homeserver.

Example:

```text
qbit.home.arpa → 100.83.21.64
```

In AdGuard Home:

**Filters → DNS rewrites**

Add:

```text
Domain:
qbit.home.arpa

IP address:
100.83.21.64
```

Repeat for other services.

Example structure:

```text
qbit.home.arpa       → 100.83.21.64
drive.home.arpa      → 100.83.21.64
jellyfin.home.arpa   → 100.83.21.64
photos.home.arpa     → 100.83.21.64
music.home.arpa      → 100.83.21.64
books.home.arpa      → 100.83.21.64
audio.home.arpa      → 100.83.21.64
metube.home.arpa     → 100.83.21.64
portainer.home.arpa  → 100.83.21.64
```

All names resolve to the same Tailscale IP.

The **port is not part of the DNS record**.

---

# 12. Verify DNS Rewrites

From the ProDesk:

```bash
nslookup qbit.home.arpa
```

Expected:

```text
Name:    qbit.home.arpa
Address: 100.83.21.64
```

This confirms:

```text
qbit.home.arpa
       ↓
AdGuard Home
       ↓
100.83.21.64
```

---

# 13. Nginx Proxy Manager

Nginx Proxy Manager was installed in:

```text
/opt/stacks/nginx-proxy-manager
```

Check:

```bash
cd /opt/stacks/nginx-proxy-manager
pwd
```

Expected:

```text
/opt/stacks/nginx-proxy-manager
```

---

# 14. Nginx Proxy Manager Data Directories

Create:

```bash
mkdir -p data letsencrypt
```

Verify:

```bash
ls -ld data letsencrypt
```

Expected ownership was:

```text
kiran kiran
```

This avoids permission problems when Docker writes its data.

---

# 15. Nginx Proxy Manager Compose File

The Compose file:

```yaml
services:
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped

    ports:
      - "80:80"
      - "81:81"
      - "443:443"

    extra_hosts:
      - "host.docker.internal:host-gateway"

    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

---

# 16. Check Ports Before Starting Nginx Proxy Manager

Nginx Proxy Manager requires:

```text
80
81
443
```

Check that they are free:

```bash
sudo ss -lntup | grep -E ':(80|81|443)\b'
```

If there is no output, the ports are free.

Do not stop an existing service blindly if one of these ports is already occupied.

---

# 17. Start Nginx Proxy Manager

From:

```text
/opt/stacks/nginx-proxy-manager
```

run:

```bash
docker compose up -d
```

Check:

```bash
docker ps --filter name=nginx-proxy-manager
```

Expected:

```text
nginx-proxy-manager
Up
```

Ports:

```text
80 → 80
81 → 81
443 → 443
```

Check logs:

```bash
docker logs --tail 50 nginx-proxy-manager
```

A successful startup ends with something similar to:

```text
Backend PID ... listening on port 3000
```

Initial migration messages are normal.

---

# 18. Nginx Proxy Manager Administration

The Nginx Proxy Manager admin interface is on port 81.

Initial access:

```text
http://homeserver.phoenix-kingsnake.ts.net:81
```

Create the administrator account through the initial setup page.

After login, the Nginx Proxy Manager dashboard should appear.

---

# 19. First Reverse Proxy — qBittorrent

qBittorrent is exposed by Docker on:

```text
100.83.21.64:8080
```

Create an Nginx Proxy Manager:

**Hosts → Proxy Hosts → Add Proxy Host**

Use:

```text
Domain Names:
qbit.home.arpa

Scheme:
http

Forward Hostname / IP:
100.83.21.64

Forward Port:
8080
```

Save.

---

# 20. How the qBittorrent Proxy Works

The browser requests:

```text
http://qbit.home.arpa
```

DNS:

```text
qbit.home.arpa
       ↓
100.83.21.64
```

Nginx Proxy Manager receives the request:

```text
100.83.21.64:80
```

and forwards it to:

```text
100.83.21.64:8080
```

Therefore:

```text
http://qbit.home.arpa
```

works without:

```text
:8080
```

---

# 21. Important Reverse Proxy Concept — HTTP vs HTTPS

The **Scheme** in Nginx Proxy Manager must match the protocol spoken by the application on its forwarded port.

For qBittorrent:

```text
Scheme: http
Port: 8080
```

For Portainer:

```text
Scheme: https
Port: 9443
```

This distinction is important.

---

# 22. Portainer Special Case

Portainer exposes:

```text
9000 → HTTP
9443 → HTTPS
```

The Portainer proxy initially produced:

```text
Client sent an HTTP request to an HTTPS server
```

because Nginx Proxy Manager was configured as:

```text
Scheme: http
Forward Port: 9443
```

Port 9443 expects HTTPS.

## Correct configuration

```text
Domain Names:
portainer.home.arpa

Scheme:
https

Forward Hostname / IP:
100.83.21.64

Forward Port:
9443
```

After changing the Scheme to `https`, Portainer worked correctly.

---

# 23. Current Service Port Reference

Based on the Docker configuration:

| Service        | Docker host port | Backend protocol |
| -------------- | ---------------: | ---------------- |
| qBittorrent    |             8080 | HTTP             |
| Seafile        |             8083 | HTTP             |
| MeTube         |             8081 | HTTP             |
| BookOrbit      |             3000 | HTTP             |
| Audiobookshelf |            13378 | HTTP             |
| Navidrome      |             4533 | HTTP             |
| Immich         |             2283 | HTTP             |
| Jellyfin       |             8096 | HTTP             |
| Portainer      |             9443 | HTTPS            |

Portainer is the notable exception because its preferred web interface is HTTPS on 9443.

---

# 24. Recommended Naming Scheme

Use simple service names under:

```text
home.arpa
```

Recommended:

```text
adguard.home.arpa
nginx.home.arpa

qbit.home.arpa
drive.home.arpa
jellyfin.home.arpa
photos.home.arpa
music.home.arpa
books.home.arpa
audio.home.arpa
metube.home.arpa
portainer.home.arpa
```

This is easier to remember than ports.

For example:

```text
http://qbit.home.arpa
http://drive.home.arpa
http://jellyfin.home.arpa
http://photos.home.arpa
```

---

# 25. Giving AdGuard and Nginx Proxy Manager Friendly Names

It is good practice to give the management applications their own names.

Recommended:

```text
adguard.home.arpa
nginx.home.arpa
```

These names should resolve to:

```text
100.83.21.64
```

### Important

AdGuard's DNS service itself remains:

```text
100.83.21.64:53
```

The friendly name is only for the AdGuard web interface.

Similarly, Nginx Proxy Manager's admin interface is:

```text
100.83.21.64:81
```

The friendly name is for its web interface.

---

# 26. AdGuard Home Uses Beyond DNS Rewrites

AdGuard Home can provide:

### DNS filtering

Block advertising and tracking domains at DNS level.

### Query logging

See which DNS domains devices are requesting.

### Statistics

View:

* total DNS queries
* blocked queries
* blocking percentage
* frequently requested domains
* frequently blocked domains
* client activity

### DNS caching

Repeated DNS requests can be answered from AdGuard's cache.

### Upstream DNS

AdGuard forwards domains it doesn't know to configured upstream DNS servers.

### Per-device filtering

Different clients can have different filtering rules.

### Allowlisting

If a legitimate service is accidentally blocked, allow the required domain rather than disabling filtering globally.

---

# 27. AdGuard Filtering — Recommended Approach

Start conservatively.

Do not immediately add dozens of third-party blocklists.

Recommended initial approach:

1. Use AdGuard's built-in/default general-purpose filtering.
2. Verify normal websites work.
3. Check Query Log.
4. Look at blocked requests.
5. Add allowlist entries only when necessary.
6. Avoid overly aggressive filtering on the homeserver.

DNS blocking is not a complete replacement for browser ad blockers.

---

# 28. Testing AdGuard Filtering

Use:

**AdGuard Home → Query Log**

Browse normally from the ProDesk or Android device.

You should see queries marked as allowed or blocked.

DNS-level blocking can affect:

* browsers
* applications
* smart devices
* telemetry
* trackers
* advertising domains

If an application breaks:

1. Check AdGuard Query Log.
2. Find the blocked domain.
3. Determine whether it is actually required.
4. Add a targeted allowlist rule if appropriate.

Do not immediately disable all filtering.

---

# 29. Testing the Complete System

## Test Tailscale

```bash
tailscale status
```

## Test MagicDNS

```bash
ping -c 4 homeserver.phoenix-kingsnake.ts.net
```

## Test DNS

```bash
nslookup qbit.home.arpa
```

Expected:

```text
100.83.21.64
```

## Test AdGuard directly

```bash
nslookup example.com 100.83.21.64
```

## Test qBittorrent

```text
http://qbit.home.arpa
```

## Test Portainer

```text
http://portainer.home.arpa
```

## Test Nginx Proxy Manager

```text
http://nginx.home.arpa
```

## Test AdGuard

```text
http://adguard.home.arpa
```

---

# 30. Troubleshooting Checklist

## Friendly hostname doesn't resolve

Check:

```bash
tailscale dns status
```

Make sure:

```text
Resolvers:
100.83.21.64
```

Then:

```bash
nslookup service.home.arpa
```

Expected:

```text
100.83.21.64
```

On Android, make sure:

**Tailscale → Settings → Use Tailscale DNS**

is enabled.

---

## DNS works but web page doesn't open

Test the backend directly:

```bash
curl -I http://100.83.21.64:PORT
```

For example:

```bash
curl -I http://100.83.21.64:8080
```

If the backend responds but the friendly name doesn't work, inspect Nginx Proxy Manager.

---

## Nginx gives 502 Bad Gateway

Check:

```bash
docker ps
```

Then test the backend from the homeserver:

```bash
curl -I http://100.83.21.64:PORT
```

For an HTTPS backend:

```bash
curl -k -I https://100.83.21.64:9443
```

Also verify the Nginx Proxy Manager Scheme matches the application:

```text
HTTP backend → http
HTTPS backend → https
```

---

## Error:

```text
Client sent an HTTP request to an HTTPS server
```

The reverse proxy is using:

```text
http
```

against an HTTPS backend.

Change Nginx Proxy Manager's:

```text
Scheme
```

to:

```text
https
```

Portainer on 9443 is an example.

---

# 31. Useful Docker Commands

List all containers:

```bash
docker ps
```

Show container names, images and ports:

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
```

Check a specific container:

```bash
docker ps --filter name=adguardhome
```

or:

```bash
docker ps --filter name=nginx-proxy-manager
```

View logs:

```bash
docker logs --tail 50 adguardhome
```

```bash
docker logs --tail 50 nginx-proxy-manager
```

Stop a Compose stack:

```bash
docker compose down
```

Start a Compose stack:

```bash
docker compose up -d
```

---

# 32. Useful Network Commands

Show listening TCP/UDP ports:

```bash
sudo ss -lntup
```

Check a particular port:

```bash
sudo ss -lntup | grep ':53'
```

Check HTTP response:

```bash
curl -I http://hostname:port
```

Check DNS:

```bash
nslookup hostname
```

Check a specific DNS server:

```bash
nslookup hostname 100.83.21.64
```

Check Tailscale:

```bash
tailscale status
```

Check Tailscale IP:

```bash
tailscale ip
```

Check Tailscale DNS:

```bash
tailscale dns status
```

---

# 33. Security Model

The intended architecture is:

```text
                    INTERNET
                       X
                       │
                 No public exposure
                       │
                       X

             ┌───────────────────┐
             │     TAILSCALE     │
             │ Private encrypted │
             │      network      │
             └─────────┬─────────┘
                       │
                       ▼
                100.83.21.64
                 HOMESERVER
                       │
              ┌────────┴────────┐
              │                 │
         AdGuard Home      Nginx Proxy
            :53              :80/:443
              │                 │
              │                 ▼
              │          Docker services
              │
              ▼
       DNS filtering +
       DNS rewrites
```

The objective is **not** to expose the applications to the public Internet.

Tailscale provides the private network path.

AdGuard provides DNS.

Nginx Proxy Manager provides friendly HTTP reverse-proxy names.

---

# 34. HTTPS Considerations

Currently the friendly service names use HTTP:

```text
http://qbit.home.arpa
```

This is acceptable for the current Tailscale-only architecture because traffic between Tailscale devices is protected by Tailscale's encrypted tunnel.

However, HTTP is still HTTP at the application layer.

If browser-visible HTTPS is desired later, a certificate strategy is needed.

A public Let's Encrypt certificate generally cannot simply be issued for:

```text
*.home.arpa
```

because `.home.arpa` is a private namespace.

Tailscale HTTPS certificates are designed around Tailscale `*.ts.net` hostnames, not arbitrary `*.home.arpa` names.

Possible future approach:

```text
Private CA
    ↓
*.home.arpa certificate
    ↓
Nginx Proxy Manager
    ↓
HTTPS
```

This can be done without purchasing a domain.

Do this as a separate step after the DNS/reverse-proxy architecture is stable.

---

# 35. Final Mental Model

The entire system can be remembered as:

```text
Tailscale
    ↓
"How do I securely reach my homeserver?"

AdGuard Home
    ↓
"What IP address does qbit.home.arpa mean?"

Nginx Proxy Manager
    ↓
"Which application should receive qbit.home.arpa?"

Docker
    ↓
"Run the actual application."
```

Example:

```text
Android
   │
   │ Tailscale
   ▼
AdGuard Home
   │
   │ qbit.home.arpa → 100.83.21.64
   ▼
Nginx Proxy Manager
   │
   │ HTTP → 100.83.21.64:8080
   ▼
qBittorrent
```

For Portainer:

```text
Android
   │
   │ Tailscale
   ▼
AdGuard
   │
   │ portainer.home.arpa → 100.83.21.64
   ▼
Nginx Proxy Manager
   │
   │ HTTPS → 100.83.21.64:9443
   ▼
Portainer
```

---

# 36. Current Working State

At the end of this setup:

* [x] Tailscale installed and working
* [x] MagicDNS enabled
* [x] Tailnet renamed to `phoenix-kingsnake.ts.net`
* [x] Homeserver Tailscale IP = `100.83.21.64`
* [x] AdGuard Home installed
* [x] AdGuard DNS available on `100.83.21.64:53`
* [x] Tailscale configured to use AdGuard DNS
* [x] DNS rewrites configured
* [x] Android configured to use Tailscale DNS
* [x] `.home.arpa` names resolve from Tailscale devices
* [x] Nginx Proxy Manager installed
* [x] Nginx Proxy Manager ports 80/81/443 available
* [x] qBittorrent reverse proxy working
* [x] Portainer reverse proxy working
* [x] HTTP vs HTTPS backend distinction verified
* [ ] AdGuard ad/tracker filtering to be configured/verified
* [ ] HTTPS for `*.home.arpa` can be added later if desired

## Core addresses

```text
Tailscale homeserver:
homeserver.phoenix-kingsnake.ts.net

Tailscale IP:
100.83.21.64

AdGuard DNS:
100.83.21.64:53

AdGuard Web UI:
http://homeserver.phoenix-kingsnake.ts.net:3001

Nginx Proxy Manager Admin:
http://homeserver.phoenix-kingsnake.ts.net:81

Private service namespace:
*.home.arpa
```
