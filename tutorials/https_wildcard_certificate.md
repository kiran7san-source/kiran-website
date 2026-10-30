# Automated Public Key Infrastructure: Let's Encrypt Wildcard SSL via Certbot & Cloudflare DNS

## 1. Overview & Architectural Background
Transport Layer Security (TLS/SSL) provides foundational privacy, authenticity, and data integrity across network communications. While modern reverse proxies make issuing certificates for public subdomains straightforward via HTTP-01 challenges, securing homelab environments accessible only through private overlay networks (like Tailscale) or internal subnets presents unique operational hurdles.

This guide details obtaining and automatically renewing **Let's Encrypt Wildcard SSL Certificates (`*.yourdomain.com`)** using **Certbot** and the **Cloudflare DNS-01 API challenge**.

### ACME Validation Mechanics: HTTP-01 vs. DNS-01
The Automated Certificate Management Environment (ACME) protocol validates domain control before the Certificate Authority (Let's Encrypt) issues cryptographic certificates:

* **HTTP-01 Challenge**:
  * **Mechanism**: The ACME server attempts an inbound HTTP request to `http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN>`.
  * **Constraint**: Requires exposing inbound ports 80 and 443 directly to the public internet. It cannot issue wildcard certificates (`*.domain.com`) because an HTTP token can only prove ownership of an individual, fully qualified host address.
* **DNS-01 Challenge**:
  * **Mechanism**: The ACME client leverages provider APIs (Cloudflare) to provision a dynamic cryptographic `TXT` record at `_acme-challenge.<YOUR_DOMAIN>`. Let's Encrypt queries authoritative global DNS nameservers to verify record propagation.
  * **Zero Inbound Ports**: Bypasses the need for any open inbound firewall or ISP router ports.
  * **Wildcard Capable**: Supports issuance of universal wildcard records covering infinite third-level subdomains (`*.domain.com`, `auth.domain.com`, `nas.domain.com`) under one unified certificate pair.

---

## 2. Production Homelab Use Cases
* **Private Network Encryption**: Secure internal homelab dashboards and reverse proxies with fully trusted CA certificates without opening public firewall ports.
* **Wildcard Multi-Service Routing**: Issue a single unified certificate pair (`cert.pem`, `privkey.pem`) to terminate TLS across all homelab microservices (Portainer, Immich, Jellyfin, Nextcloud).
* **Automated Zero-Touch Renewals**: Rotate short-lived 90-day certificates continuously via cron or systemd timers without manual certificate swaps.
* **Browser Trust Compliance**: Eliminate self-signed certificate security warnings and satisfy strict WebCrypto requirements on mobile and desktop browsers.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12).
* Administrative `sudo` privileges.
* A registered public domain name (e.g., `yourdomain.com`) whose authoritative nameservers point to Cloudflare.
* A scoped Cloudflare API Token with DNS record editing permissions.

### Directory Layout
```text
/opt/homelab/certificates/
├── cloudflare.ini
├── scripts/
│   └── certbot-renew-hook.sh
└── certs/
```

---

## 4. Provisioning Scoped Cloudflare Credentials

To minimize attack surface area, generate an API token restricted exclusively to DNS modifications rather than using your global Cloudflare account key.

### Step 4.1: Generate the Cloudflare API Token
1. Log into the **Cloudflare Dashboard** and navigate to **My Profile** ➔ **API Tokens**.
2. Click **Create Token** ➔ select the **Edit zone DNS** template.
3. Configure the token permissions:
   * **Permissions**: `Zone` | `DNS` | `Edit`
   * **Zone Resources**: `Include` | `Specific zone` | `yourdomain.com`
4. Set a Client IP filter (optional) and click **Continue to summary** ➔ **Create Token**.
5. Copy the generated token string safely.

### Step 4.2: Store Credentials Securely on the Host
Store your token in an access-restricted configuration file readable exclusively by root:

```bash
sudo mkdir -p /etc/letsencrypt /opt/homelab/certificates
sudo tee /etc/letsencrypt/cloudflare.ini <<'EOF'
# Cloudflare API token used by Certbot
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN_HERE
EOF

# Enforce strict root-only read/write privileges
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
```

---

## 5. Step-by-Step Implementation

### Step 5.1: Install Certbot & the Cloudflare DNS Plugin
Install Certbot and the required Cloudflare DNS authenticator using Python package isolation or distribution package managers:

```bash
sudo apt-get update
sudo apt-get install -y certbot python3-certbot-dns-cloudflare
```

Verify binary and plugin installation:

```bash
certbot plugins --text
```
*(Ensure `dns-cloudflare` is listed in the detected plugins).*

### Step 5.2: Issue the Wildcard Certificate
Request a certificate covering both the root apex domain (`yourdomain.com`) and all immediate subdomains (`*.yourdomain.com`). Set a propagation delay (30–60 seconds) to ensure Cloudflare DNS record propagation settles before Let's Encrypt performs validation queries:

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d yourdomain.com \
  -d *.yourdomain.com \
  --agree-tos \
  --non-interactive \
  --email admin@yourdomain.com
```

### Step 5.3: Certificate Path Verification
Upon successful validation, Let's Encrypt commits your keys to `/etc/letsencrypt/live/yourdomain.com/`:
* `fullchain.pem`: Combined server certificate and intermediate CA certificate bundle.
* `privkey.pem`: Your private cryptographic key (keep strictly confidential).
* `cert.pem`: Base server leaf certificate.
* `chain.pem`: Let's Encrypt intermediate certificate authority chain.

Verify file generation:

```bash
sudo ls -l /etc/letsencrypt/live/[yourdomain.com/](https://yourdomain.com/)
```

---

## 6. Automated Renewal & Service Reload Pipeline

Let's Encrypt certificates remain valid for 90 days. Certbot installs a systemd timer by default, but you should configure post-renewal hooks to automatically deploy new certificates to target services (Nginx, Docker containers, reverse proxies) without manual intervention.

### Step 6.1: Create a Post-Renewal Deploy Hook
Create `/etc/letsencrypt/renewal-hooks/deploy/reload-services.sh`:

```bash
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-services.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

echo "[$(date)] Certificate renewed. Triggering service reloads..."

# 1. Reload Nginx Proxy Manager (if running via Docker)
if docker ps --format '{{.Names}}' | grep -q "^nginx-proxy-manager$"; then
    echo "Reloading Nginx Proxy Manager container..."
    docker exec nginx-proxy-manager nginx -s reload
fi

# 2. Reload Host Nginx (if installed natively)
if systemctl is-active --quiet nginx; then
    echo "Reloading system Nginx..."
    systemctl reload nginx
fi

echo "[$(date)] Service reloads executed successfully."
EOF

sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-services.sh
```

### Step 6.2: Test Renewal Automation (Dry Run)
Simulate the end-to-end renewal pipeline using Certbot's dry-run engine:

```bash
sudo certbot renew --dry-run
```

Ensure the output terminates with: `Congratulations, all simulated renewals succeeded`.

---

## 7. Mounting Certificates into Docker Compose Services

To consume the issued wildcard certificates within containerized reverse proxies (e.g., Nginx, Traefik, or Caddy), mount the live directory into your containers as read-only volumes:

```yaml
services:
  reverse-proxy:
    image: nginx:alpine
    container_name: reverse-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/letsencrypt/live/[yourdomain.com/fullchain.pem:/etc/nginx/certs/fullchain.pem:ro](https://yourdomain.com/fullchain.pem:/etc/nginx/certs/fullchain.pem:ro)
      - /etc/letsencrypt/live/[yourdomain.com/privkey.pem:/etc/nginx/certs/privkey.pem:ro](https://yourdomain.com/privkey.pem:/etc/nginx/certs/privkey.pem:ro)
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

---

## 8. Operational Troubleshooting Matrix

| Issue / Error Output | Root Cause | Remediation Procedure |
| :--- | :--- | :--- |
| **`Cloudflare API error: 6003 Invalid request headers`** | Syntax error, extra spaces, or using an account Global Key under a token variable. | Ensure `/etc/letsencrypt/cloudflare.ini` uses `dns_cloudflare_api_token = ...` without trailing spaces or quotes. |
| **`Waiting 60 seconds for DNS changes -- DNS problem: NXDOMAIN looking up TXT`** | DNS propagation delay exceeded or nameservers are out of sync. | Increase the propagation timeout flag to `--dns-cloudflare-propagation-seconds 120`. |
| **`Permission Denied` when containers access certificates** | Target container UID cannot traverse `/etc/letsencrypt/archive/` due to root-only `0700` POSIX directory masks. | Set `/etc/letsencrypt/archive` directory traversal permissions to `0755` (`sudo chmod 0755 /etc/letsencrypt/{live,archive}`) while leaving `privkey.pem` restricted to read-only for the container's GID. |
| **`Too many certificates already issued for exact set of domains`** | Exceeded Let's Encrypt rate limit thresholds (5 duplicate certificates per 7-day rolling window). | Pass the `--test-cert` (staging) flag during trial deployments and configuration testing. |
