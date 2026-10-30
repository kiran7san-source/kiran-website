# Homeserver — HTTPS Setup with Wildcard `*.home.arpa` Certificate

> **Purpose:** Replicate the complete HTTPS setup used on the Homeserver so that all self-hosted services are accessed securely through HTTPS using a single local wildcard certificate and Nginx Proxy Manager (NPM).
>
> **Environment:** Ubuntu Server 24.04.x Homeserver + Docker + Nginx Proxy Manager + Tailscale + Linux Mint client + Firefox.
>
> **Final architecture:**
>
> `Firefox → HTTPS :443 → Nginx Proxy Manager → HTTP → Docker service`
>
> The applications themselves do **not** need individual HTTPS certificates.

---

## 1. Architecture and Design

The setup uses three important components:

1. **Local Certificate Authority (CA)**

   * `Kiran Homeserver Local CA`
   * Trusted by client devices.

2. **Wildcard server certificate**

   * `*.home.arpa`
   * Used by Nginx Proxy Manager for all web services.

3. **Nginx Proxy Manager**

   * Terminates HTTPS.
   * Uses the wildcard certificate.
   * Proxies requests to the individual Docker services over HTTP.

### Result

```text
Client device
    │
    │ HTTPS
    │
    ▼
*.home.arpa
    │
    ▼
Nginx Proxy Manager
    │
    │ HTTP
    ▼
Docker service
```

Example:

```text
https://photos.home.arpa
        │
        ▼
Nginx Proxy Manager :443
        │
        ▼
Immich :2283
```

---

# 2. Certificate Files

Certificate directory on the Homeserver:

```text
/opt/stacks/certificates/
```

Relevant files:

```text
/opt/stacks/certificates/ca/home-arpa-ca.crt
/opt/stacks/certificates/ca/home-arpa-ca.key

/opt/stacks/certificates/wildcard/home-arpa-wildcard.crt
/opt/stacks/certificates/wildcard/home-arpa-wildcard.key
/opt/stacks/certificates/wildcard/home-arpa-wildcard.cnf
```

### Important security rule

The `.key` files are **private keys**.

Do NOT distribute:

```text
home-arpa-ca.key
home-arpa-wildcard.key
```

to client computers.

Client devices only need:

```text
home-arpa-ca.crt
```

The CA private key should be backed up securely because losing it means certificates issued by that CA can no longer be reproduced.

---

# 3. Verify the Wildcard Certificate

Use:

```bash
sudo openssl x509 \
  -in /opt/stacks/certificates/wildcard/home-arpa-wildcard.crt \
  -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName
```

Expected:

```text
subject=CN = *.home.arpa
issuer=C = NP, O = Kiran Home Server, CN = Kiran Homeserver Local CA
notBefore=Aug  7 06:08:39 2026 GMT
notAfter=Nov  9 06:08:39 2028 GMT

X509v3 Subject Alternative Name:
    DNS:*.home.arpa
    DNS:home.arpa
```

The important part is:

```text
DNS:*.home.arpa
```

This allows the certificate to match:

```text
photos.home.arpa
jellyfin.home.arpa
music.home.arpa
books.home.arpa
qbit.home.arpa
...
```

---

# 4. Verify the Certificate Against the Local CA

Run:

```bash
sudo openssl verify \
  -CAfile /opt/stacks/certificates/ca/home-arpa-ca.crt \
  /opt/stacks/certificates/wildcard/home-arpa-wildcard.crt
```

Expected:

```text
/opt/stacks/certificates/wildcard/home-arpa-wildcard.crt: OK
```

If this says `OK`, the wildcard certificate was correctly signed by the local CA.

---

# 5. Verify the Wildcard Certificate Configuration

The configuration used to create the wildcard certificate was:

```ini
[req]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[dn]
CN = *.home.arpa

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.home.arpa
DNS.2 = home.arpa
```

File:

```text
/opt/stacks/certificates/wildcard/home-arpa-wildcard.cnf
```

### Important

The certificate must contain the wildcard hostname in **Subject Alternative Name (SAN)**.

Having only:

```text
CN = *.home.arpa
```

is not sufficient for modern browsers.

The required SAN is:

```text
DNS:*.home.arpa
```

---

# 6. Copy the CA Certificate to Client Computers

The CA certificate was copied to the shared folder so that remote client computers could access it.

Copy only:

```text
home-arpa-ca.crt
```

Do not copy the private keys.

For example:

```text
home-arpa-ca.crt       ← safe to distribute to trusted client devices
home-arpa-ca.key       ← KEEP PRIVATE
home-arpa-wildcard.crt ← normally not needed on clients
home-arpa-wildcard.key ← KEEP PRIVATE
```

---

# 7. Install the CA on Linux Mint

On the Linux Mint client, the CA was installed into the system trust store.

Copy:

```text
home-arpa-ca.crt
```

to:

```text
/usr/local/share/ca-certificates/
```

For example:

```bash
sudo cp ~/home-arpa-ca.crt \
  /usr/local/share/ca-certificates/home-arpa-ca.crt
```

Then:

```bash
sudo update-ca-certificates
```

Verify:

```bash
sudo openssl verify \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  /usr/local/share/ca-certificates/home-arpa-ca.crt
```

Expected:

```text
/usr/local/share/ca-certificates/home-arpa-ca.crt: OK
```

Also verify that the certificate was added to the hashed certificate directory:

```bash
ls -l /etc/ssl/certs/ | grep -i home-arpa
```

Example result:

```text
f943cec6.0 -> home-arpa-ca.pem
home-arpa-ca.pem -> /usr/local/share/ca-certificates/home-arpa-ca.crt
```

---

# 8. Firefox Certificate Trust

Firefox was configured to trust:

```text
Kiran Homeserver Local CA
```

After importing:

```text
home-arpa-ca.crt
```

Firefox successfully accessed:

```text
https://photos.home.arpa
```

without a certificate warning.

### Important

If Firefox displays a certificate warning after importing the CA:

* Verify that the correct CA certificate was imported.
* Do not import the `.key` file.
* Do not regenerate the wildcard certificate unnecessarily.
* Verify that the website is actually presenting the expected wildcard certificate.

---

# 9. Brave Browser — Important Lesson

Brave was installed as a **Flatpak**:

```text
com.brave.Browser
```

Version:

```text
1.92.140
```

The host system trusted the CA and:

```bash
curl https://photos.home.arpa
```

worked correctly.

Firefox also worked.

However, Flatpak Brave did not trust the local CA because the Flatpak sandbox had its own CA environment.

The following confirmed the difference:

```bash
flatpak run --command=sh com.brave.Browser -c \
'grep -a "Kiran Homeserver Local CA" /etc/ssl/certs/ca-certificates.crt >/dev/null && echo FOUND || echo NOT_FOUND'
```

Result:

```text
NOT_FOUND
```

Therefore:

```text
Linux system CA       → trusted
curl                  → works
Firefox               → works
Flatpak Brave         → does not trust local CA
```

### Final decision

Brave troubleshooting was abandoned.

**Firefox is the browser used for the `*.home.arpa` services.**

If reproducing this setup, there is no need to troubleshoot Flatpak Brave unless Brave is specifically required.

---

# 10. Verify DNS Resolution

The Homeserver services use Tailscale.

On the Linux Mint client:

```bash
resolvectl query photos.home.arpa
```

Example:

```text
photos.home.arpa: 100.83.21.64
```

The hostname must resolve to the Homeserver's Tailscale address.

In this setup:

```text
photos.home.arpa
        ↓
100.83.21.64
        ↓
Homeserver
```

---

# 11. Verify the HTTPS Certificate Being Served

From the Linux Mint client:

```bash
openssl s_client \
  -connect photos.home.arpa:443 \
  -servername photos.home.arpa \
  -showcerts </dev/null 2>/dev/null |
  openssl x509 -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName
```

Expected:

```text
subject=CN = *.home.arpa
issuer=C = NP, O = Kiran Home Server, CN = Kiran Homeserver Local CA
```

And:

```text
X509v3 Subject Alternative Name:
    DNS:*.home.arpa
    DNS:home.arpa
```

This proves that Nginx Proxy Manager is actually presenting the wildcard certificate.

---

# 12. Verify HTTPS with curl

Run:

```bash
curl -v https://photos.home.arpa
```

Important lines should include:

```text
subject: CN=*.home.arpa
subjectAltName: host "photos.home.arpa" matched cert's "*.home.arpa"
issuer: C=NP; O=Kiran Home Server; CN=Kiran Homeserver Local CA
SSL certificate verify ok.
```

And:

```text
HTTP/2 200
```

This verifies:

1. DNS works.
2. Port 443 is reachable.
3. TLS works.
4. The wildcard SAN matches the hostname.
5. The local CA is trusted.
6. Nginx Proxy Manager is responding.
7. The backend application is responding.

---

# 13. Nginx Proxy Manager Configuration

All web services are now accessed through Nginx Proxy Manager.

NPM listens on:

```text
HTTP  :80
HTTPS :443
Admin :81
```

The important configuration is:

```text
Client
   │
   │ HTTPS :443
   ▼
Nginx Proxy Manager
   │
   │ HTTP
   ▼
Application
```

### General Proxy Host Settings

For each service:

**Domain Names**

```text
service.home.arpa
```

**Scheme**

```text
http
```

**Forward Hostname / IP**

```text
100.83.21.64
```

**Forward Port**

Use the application's published host port.

Enable:

```text
Block Common Exploits
Websockets Support
```

### SSL

Select:

```text
*.home.arpa
```

Then enable:

```text
Force SSL
HTTP/2 Support
```

Save.

---

# 14. Current Service Mappings

Current Docker ports were verified using:

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Networks}}'
```

The verified mappings were:

| Service      | Hostname              | Forward Host/IP | Forward Port |
| ------------ | --------------------- | --------------: | -----------: |
| Immich       | `photos.home.arpa`    |  `100.83.21.64` |       `2283` |
| Jellyfin     | `jellyfin.home.arpa`  |  `100.83.21.64` |       `8096` |
| Navidrome    | `music.home.arpa`     |  `100.83.21.64` |       `4533` |
| BookOrbit    | `books.home.arpa`     |  `100.83.21.64` |       `3000` |
| qBittorrent  | `qbit.home.arpa`      |  `100.83.21.64` |       `8080` |
| MeTube       | `metube.home.arpa`    |  `100.83.21.64` |       `8081` |
| Seafile      | `drive.home.arpa`     |  `100.83.21.64` |       `8083` |
| AdGuard Home | `adguard.home.arpa`   |  `100.83.21.64` |       `3001` |
| Portainer    | `portainer.home.arpa` |  `100.83.21.64` |       `9443` |

### Important port correction

Navidrome uses:

```text
4533
```

not `4553`.

Seafile uses:

```text
Host 8083 → Container 80
```

for its web interface.

---

# 15. Jellyfin Example

NPM Proxy Host:

```text
Domain:
jellyfin.home.arpa

Scheme:
http

Forward Hostname/IP:
100.83.21.64

Forward Port:
8096
```

SSL:

```text
Certificate:
*.home.arpa

Force SSL:
ON

HTTP/2:
ON
```

Websockets:

```text
ON
```

The resulting connection is:

```text
https://jellyfin.home.arpa
        ↓
NPM :443
        ↓
http://100.83.21.64:8096
        ↓
Jellyfin
```

Jellyfin itself does **not** need to be configured for HTTPS.

---

# 16. Navidrome Example

NPM Proxy Host:

```text
Domain:
music.home.arpa

Scheme:
http

Forward Hostname/IP:
100.83.21.64

Forward Port:
4533
```

SSL:

```text
*.home.arpa
Force SSL: ON
HTTP/2: ON
```

Websockets:

```text
ON
```

Test:

```text
https://music.home.arpa
```

---

# 17. BookOrbit Example

```text
Domain:
books.home.arpa

Scheme:
http

Forward Hostname/IP:
100.83.21.64

Forward Port:
3000
```

SSL:

```text
*.home.arpa
Force SSL: ON
HTTP/2: ON
```

Test:

```text
https://books.home.arpa
```

---

# 18. General Configuration for Remaining Services

### qBittorrent

```text
Domain:
qbit.home.arpa

Scheme:
http

Forward Hostname/IP:
100.83.21.64

Forward Port:
8080
```

### MeTube

```text
Domain:
metube.home.arpa

Scheme:
http

Forward Hostname/IP:
100.83.21.64

Forward Port:
8081
```

### Seafile

```text
Domain:
drive.home.arpa

Scheme:
http

Forward Hostname/IP:
100.83.21.64

Forward Port:
8083
```

### AdGuard Home

```text
Domain:
adguard.home.arpa

Scheme:
http

Forward Hostname/IP:
100.83.21.64

Forward Port:
3001
```

### Portainer

Portainer's host mapping is:

```text
9443 → 9443
```

However, **verify whether Portainer is already serving HTTPS on its backend port** before configuring NPM.

If Portainer's `9443` endpoint is HTTPS, the NPM scheme should be:

```text
https
```

rather than:

```text
http
```

Do not blindly proxy HTTPS Portainer using HTTP.

---

# 19. Immich

Immich was already successfully configured:

```text
https://photos.home.arpa
```

The backend is:

```text
100.83.21.64:2283
```

The successful verification was:

```text
TLSv1.3
subject: CN=*.home.arpa
subjectAltName: photos.home.arpa matched *.home.arpa
issuer: Kiran Homeserver Local CA
SSL certificate verify ok.
HTTP/2 200
```

Therefore Immich is correctly behind NPM.

---

# 20. Do NOT Enable HTTPS Individually in Every Application

The preferred architecture is:

```text
Browser
   │
   │ HTTPS
   ▼
Nginx Proxy Manager
   │
   │ HTTP
   ▼
Docker application
```

Do **not** unnecessarily configure:

```text
Jellyfin → HTTPS
Navidrome → HTTPS
BookOrbit → HTTPS
Immich → HTTPS
...
```

NPM is the central TLS termination point.

This provides:

* One wildcard certificate.
* One place to manage HTTPS.
* One place to renew/replace certificates.
* Consistent HTTPS URLs.
* Less application-specific configuration.

---

# 21. Why the Wildcard Certificate Works

The certificate contains:

```text
DNS:*.home.arpa
```

Therefore it matches:

```text
photos.home.arpa
jellyfin.home.arpa
music.home.arpa
books.home.arpa
qbit.home.arpa
metube.home.arpa
drive.home.arpa
adguard.home.arpa
portainer.home.arpa
```

It does **not** automatically match deeper names such as:

```text
something.photos.home.arpa
```

because:

```text
*.home.arpa
```

only represents one hostname level.

---

# 22. Certificate Validation Troubleshooting

If Firefox reports:

```text
SEC_ERROR_UNKNOWN_ISSUER
```

check whether Firefox trusts:

```text
Kiran Homeserver Local CA
```

If Brave reports:

```text
ERR_CERT_COMMON_NAME_INVALID
```

first verify the actual certificate being served:

```bash
openssl s_client \
  -connect service.home.arpa:443 \
  -servername service.home.arpa \
  -showcerts </dev/null 2>/dev/null |
  openssl x509 -noout \
  -subject \
  -issuer \
  -ext subjectAltName
```

Make sure the SAN contains:

```text
DNS:*.home.arpa
```

and verify:

```bash
curl -v https://service.home.arpa
```

Look for:

```text
subjectAltName: host "service.home.arpa" matched cert's "*.home.arpa"
SSL certificate verify ok.
```

If `curl` works but a browser doesn't, the problem is probably the browser's trust store rather than NPM or the certificate.

---

# 23. Useful Diagnostic Commands

### Show all Docker containers and ports

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Networks}}'
```

### Test DNS

```bash
resolvectl query service.home.arpa
```

### Test certificate

```bash
openssl s_client \
  -connect service.home.arpa:443 \
  -servername service.home.arpa \
  -showcerts </dev/null 2>/dev/null |
  openssl x509 -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName
```

### Test HTTPS

```bash
curl -v https://service.home.arpa
```

### Test only the important TLS lines

```bash
curl -v https://service.home.arpa 2>&1 | \
grep -E 'Host|Trying|subject|issuer|subjectAltName|verify|HTTP/'
```

### Verify CA

```bash
sudo openssl verify \
  -CAfile /etc/ssl/certs/ca-certificates.crt \
  /usr/local/share/ca-certificates/home-arpa-ca.crt
```

### Verify wildcard certificate against CA

```bash
sudo openssl verify \
  -CAfile /opt/stacks/certificates/ca/home-arpa-ca.crt \
  /opt/stacks/certificates/wildcard/home-arpa-wildcard.crt
```

---

# 24. Security Considerations

## Protect the CA private key

Never distribute:

```text
home-arpa-ca.key
```

The CA private key is effectively the master key for certificates issued by this CA.

## Protect the wildcard private key

Never distribute:

```text
home-arpa-wildcard.key
```

It belongs on the server/NPM side.

## Client devices only need the CA certificate

```text
home-arpa-ca.crt
```

## Keep HTTPS behind Tailscale

The current preferred architecture is:

```text
Tailscale
    ↓
Homeserver
    ↓
NPM
    ↓
HTTPS services
```

The services are intended to remain accessible through the private Tailscale network rather than being publicly exposed to the Internet.

## Avoid bypassing NPM

Prefer:

```text
https://photos.home.arpa
```

instead of:

```text
http://100.83.21.64:2283
```

The hostname ensures that traffic goes through NPM and receives the correct certificate.

---

# 25. Future Security Hardening

Currently many Docker applications publish their ports directly on the Homeserver.

For example:

```text
2283
3000
4533
8080
8081
8083
8096
9000
9443
```

A future hardening step should consider preventing clients from bypassing NPM and directly accessing these published ports.

Desired architecture:

```text
Client
   │
   │ HTTPS
   ▼
NPM :443
   │
   ▼
Docker services
```

rather than:

```text
Client
   ├── :2283
   ├── :3000
   ├── :4533
   ├── :8080
   ├── :8081
   ├── :8083
   ├── :8096
   └── :9443
```

**Do this only after confirming all services are working.**

Do not change Docker port mappings or firewall rules during initial HTTPS deployment.

---

# 26. Final Verification Checklist

After reproducing the setup, verify each service in Firefox:

```text
[ ] https://photos.home.arpa
[ ] https://jellyfin.home.arpa
[ ] https://music.home.arpa
[ ] https://books.home.arpa
[ ] https://qbit.home.arpa
[ ] https://metube.home.arpa
[ ] https://drive.home.arpa
[ ] https://adguard.home.arpa
[ ] https://portainer.home.arpa
```

For each service verify:

```text
[ ] DNS resolves
[ ] HTTPS loads
[ ] No Firefox certificate warning
[ ] Certificate is *.home.arpa
[ ] Issuer is Kiran Homeserver Local CA
[ ] NPM proxy works
[ ] Application works normally
```

---

# 27. Final Working Configuration

The completed setup is:

```text
                    Tailscale
                        │
                        ▼
                Homeserver IP
                 100.83.21.64
                        │
                        ▼
             Nginx Proxy Manager
                    :443 HTTPS
                        │
                *.home.arpa cert
                        │
       ┌────────────────┼────────────────┐
       │                │                │
       ▼                ▼                ▼
 photos.home.arpa  jellyfin.home.arpa  music.home.arpa
       │                │                │
     Immich           Jellyfin         Navidrome

       ├── books.home.arpa    → BookOrbit
       ├── qbit.home.arpa     → qBittorrent
       ├── metube.home.arpa   → MeTube
       ├── drive.home.arpa    → Seafile
       ├── adguard.home.arpa  → AdGuard
       └── portainer.home.arpa → Portainer
```

### Certificate hierarchy

```text
Kiran Homeserver Local CA
          │
          │ signs
          ▼
   *.home.arpa certificate
          │
          ├── photos.home.arpa
          ├── jellyfin.home.arpa
          ├── music.home.arpa
          ├── books.home.arpa
          ├── qbit.home.arpa
          ├── metube.home.arpa
          ├── drive.home.arpa
          ├── adguard.home.arpa
          └── portainer.home.arpa
```

**Result: All Homeserver web services are centrally protected by HTTPS using one trusted local CA + one wildcard certificate + Nginx Proxy Manager.**
