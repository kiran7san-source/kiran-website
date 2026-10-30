# Private Homelab Ingress: Tailscale, AdGuard Home & Nginx Proxy Manager

## 1. Overview & Architectural Background
In a modern homelab, exposing administrative dashboards (Proxmox, Portainer, TrueNAS) and private applications to the public internet introduces unnecessary attack surfaces. This guide architects a **zero-trust, split-horizon private ingress stack** combining three core technologies:

* **Tailscale (Mesh Overlay VPN)**: Provides encrypted, point-to-point WireGuard mesh connectivity across devices without port forwarding or CGNAT traversal issues.
* **AdGuard Home (Split-Horizon Recursive DNS)**: Acts as a network-wide DNS sinkhole and authoritative resolver for a custom internal domain (e.g., `*.home.lab` or `*.internal.yourdomain.com`).
* **Nginx Proxy Manager (Reverse Proxy & Ingress Gateway)**: Handles SSL termination and routes subdomains to corresponding internal service ports via clean HTTP/HTTPS endpoints.

### Traffic Flow Architecture
```text
[ Remote Client (Tailscale) ]
            │
            ▼ 1. DNS Query: "jellyfin.home.lab"
[ AdGuard Home (100.x.y.z:53) ] ──▶ Returns NPM IP (100.x.y.z)
            │
            ▼ 2. HTTPS Request: https://jellyfin.home.lab (Port 443)
[ Nginx Proxy Manager (100.x.y.z:443) ]
            │
            ├─▶ http://192.168.1.50:8096  (Jellyfin)
            ├─▶ http://192.168.1.50:9000  (Portainer)
            └─▶ http://192.168.1.50:8080  (qBittorrent)
```

---

## 2. Core Homelab Use Cases
* **Public Port Elimination**: Secure all home server dashboards without opening port 80 or 443 on your ISP router.
* **Friendly URL Addressing**: Replace messy port numbers (e.g., `192.168.1.50:9443`) with clean, memorable subdomains (e.g., `portainer.home.lab`).
* **Centralized TLS Termination**: Use a single wildcard SSL certificate across all private internal services.
* **Ad & Tracker Blocking**: Block telemetry, advertising, and malicious domains on all Tailscale-connected mobile and desktop nodes.

---

## 3. Prerequisites & Environment Planning
* Ubuntu 22.04/24.04 LTS host with Docker & Docker Compose installed.
* Tailscale daemon installed and joined to your Tailnet.
* A base domain name (or a local reserved zone like `.internal` or `.lab`).
* Static Tailscale IP assigned to this host (e.g., `100.100.100.50`).

### Directory Layout
```text
/opt/homelab/
├── adguard/
│   ├── work/
│   └── conf/
└── npm/
    ├── data/
    └── letsencrypt/
```

---

## 4. Step-by-Step Deployment

### Step 4.1: Prepare Host Directories
Create root-owned persistent volume locations for both services:

```bash
sudo mkdir -p /opt/homelab/adguard/{work,conf}
sudo mkdir -p /opt/homelab/npm/{data,letsencrypt}
cd /opt/homelab
```

### Step 4.2: Resolve Port 53 Systemd-Resolved Conflict
Ubuntu systems run `systemd-resolved` on `127.0.0.53:53`, preventing AdGuard Home from binding to port 53. Free port 53 by disabling the stub listener:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d/
sudo tee /etc/systemd/resolved.conf.d/adguard.conf <<'EOF'
[Resolve]
DNS=1.1.1.1 8.8.8.8
DNSStubListener=no
EOF

# Link resolv.conf and restart systemd-resolved
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

### Step 4.3: Deploy Docker Compose Manifest
Create `/opt/homelab/docker-compose.yml`:

```yaml
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "3000:3000/tcp" # Initial configuration wizard
      - "8085:80/tcp"   # Permanent admin WebUI
    volumes:
      - /opt/homelab/adguard/work:/opt/adguardhome/work
      - /opt/homelab/adguard/conf:/opt/adguardhome/conf
    networks:
      ingress-net:
        ipv4_address: 172.20.0.2

  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"   # HTTP Ingress
      - "443:443" # HTTPS Ingress
      - "81:81"   # NPM Management WebUI
    volumes:
      - /opt/homelab/npm/data:/data
      - /opt/homelab/npm/letsencrypt:/etc/letsencrypt
    networks:
      ingress-net:
        ipv4_address: 172.20.0.3

networks:
  ingress-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

Bring up the stack:

```bash
sudo docker compose up -d
```

---

## 5. Configuration & Integration

### Step 5.1: Initial AdGuard Home Setup
1. Open your browser and navigate to `http://<SERVER_IP>:3000`.
2. Step through the installation wizard:
   * **Web Admin Interface**: Set listen port to `80` (mapped to host `8085`).
   * **DNS Server**: Set listen interface to `All interfaces` on port `53`.
   * Create an administrative username and password.
3. Once completed, access the dashboard at `http://<SERVER_IP>:8085`.

### Step 5.2: Configure DNS Wildcard Rewrite in AdGuard Home
To route all subdomains automatically to Nginx Proxy Manager:
1. In AdGuard Home, go to **Filters** ➔ **DNS rewrites**.
2. Click **Add DNS rewrite**.
3. **Domain name**: `*.home.lab` (or your chosen internal domain).
4. **Answer**: Enter your server's **Tailscale IPv4 address** (e.g., `100.x.y.z`).
5. Save the rule.

### Step 5.3: Set AdGuard Home as Global Tailnet Nameserver
To ensure all mobile, laptop, and remote nodes resolve internal domains:
1. Open the Tailscale Admin Console (`[https://login.tailscale.com/admin/dns](https://login.tailscale.com/admin/dns)`).
2. Go to the **DNS** tab.
3. Under **Nameservers**, click **Add nameserver** ➔ **Custom**.
4. Enter your host server's Tailscale IPv4 address (`100.x.y.z`).
5. Enable **Override local DNS** (or enable **Split DNS** for `home.lab` if you only want homelab queries routed through AdGuard).

### Step 5.4: Configure Ingress Hosts in Nginx Proxy Manager
1. Access the NPM dashboard at `http://<SERVER_IP>:81`.
   * *Default credentials*: `admin@example.com` / `changeme` (update immediately on first login).
2. Navigate to **Hosts** ➔ **Proxy Hosts** ➔ **Add Proxy Host**:
   * **Domain Names**: `jellyfin.home.lab`
   * **Scheme**: `http`
   * **Forward Hostname / IP**: `192.168.1.50` (or the Docker gateway `172.20.0.1`)
   * **Forward Port**: `8096`
   * Check **Block Common Exploits** and **Websockets Support**.
3. Under the **SSL** tab:
   * Select or request an SSL certificate (Let's Encrypt DNS-01 challenge or local self-signed certificate).
   * Check **Force SSL** and **HTTP/2 Support**.
4. Save the host.

---

## 6. Verification & Health Checks

Test DNS resolution and HTTP proxy routing from any Tailscale-connected client:

```bash
# 1. Test Split-DNS resolution against AdGuard Home
dig @100.x.y.z jellyfin.home.lab

# Expected Answer:
# jellyfin.home.lab.  10  IN  A  100.x.y.z

# 2. Test HTTP reverse proxy ingress
curl -I -k https://jellyfin.home.lab

# Expected Status:
# HTTP/2 200 (or 302/401 depending on target application)
```

---

## 7. Troubleshooting & Security Hardening

| Symptom / Requirement | Diagnostic & Remediation |
| :--- | :--- |
| **Port 53 `bind: address already in use`** | Verify `systemd-resolved` was disabled with `sudo ss -tulpn \| grep :53`. Check for rogue `dnsmasq` instances. |
| **NPM 502 Bad Gateway** | Verify NPM can route to the target backend IP. If pointing to another Docker container on the same host, ensure both share a custom Docker bridge network or target the host bridge (`172.17.0.1`). |
| **DNS Leaks on Mobile** | Ensure **Override local DNS** is active in the Tailscale admin panel and the Tailscale app has "Always-on VPN" enabled on mobile devices. |
| **Admin UI Security** | Ensure ports `81` and `8085` are bound only to the Tailscale interface (`100.x.y.z:81:81`) or protected behind firewall rules so they are never exposed to untrusted subnets. |
