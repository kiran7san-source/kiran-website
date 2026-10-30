# Production Container Management: Portainer Community Edition Deployment & Hardening

## 1. Overview & Architectural Background
Portainer Community Edition (CE) is a lightweight, centralized container management platform that provides a visual dashboard for Docker standalone hosts, Docker Swarm clusters, and Kubernetes environments. It bridges the gap between raw CLI administration and full enterprise orchestration platforms by providing intuitive container lifecycle monitoring, stack definitions, image registries, and network topology visualization.

### Architectural Breakdown & Core Mechanics
* **Docker Socket Binding (`docker.sock`)**: Portainer interfaces directly with the host's Docker daemon via the local UNIX domain socket (`/var/run/docker.sock`). Through this IPC channel, Portainer consumes the Docker Engine REST API without requiring network port exposure on the local loopback interface.
* **Declarative Stack Orchestration**: Portainer natively implements the Compose specification via its "Stacks" feature, allowing administrators to define, deploy, upgrade, and version multi-container architectures directly from a centralized web console or synchronized Git repositories.
* **Stateless Binary vs. Persistent Volume**: The Portainer binary is packaged as an optimized, single-executable Go container with minimal runtime dependencies. All user identities, hashed passwords, environment configurations, and stack metadata persist separately inside a dedicated data directory (`/data`), ensuring that the Portainer container itself can be pulled, recreated, and updated in seconds without data loss.
* **Agent Architecture (Edge & Multi-Node)**: While managing the local host directly via socket binding, Portainer also connects to remote physical nodes across separate networks using the lightweight `portainer-agent` over mutual TLS (mTLS), enabling centralized single-pane-of-glass infrastructure governance.

---

## 2. Production Homelab Use Cases
* **Visual Container Lifecycle Operations**: Quickly restart, stop, inspect, and kill containers without opening a terminal session.
* **Interactive Container Consoles & Logs**: Inspect real-time container `stdout`/`stderr` streams and drop into interactive pseudo-TTY bash/sh sessions directly from any web browser.
* **Resource Consumption Monitoring**: View real-time CPU, RAM, and network I/O usage graphs for each running container to identify memory leaks or rogue processes.
* **Orphaned Asset Pruning**: Identify and safely purge dangling images, stopped development containers, and unreferenced Docker volumes with a single click.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12) with Docker and Docker Compose installed.
* Dedicated persistent volume path on the host filesystem.
* Accessible reverse proxy (Nginx Proxy Manager, Caddy, or Traefik) terminating with SSL.
* Internal hostname or local FQDN (e.g., `portainer.home.lab` or `portainer.internal.yourdomain.com`).

### Directory Layout
```text
/opt/homelab/portainer/
├── compose.yaml
└── data/
```

---

## 4. Step-by-Step Implementation

### Step 4.1: Prepare Host Directories
Create the directory structure for Portainer persistent data:

```bash
sudo mkdir -p /opt/homelab/portainer/data
sudo chown -R $USER:$USER /opt/homelab/portainer
cd /opt/homelab/portainer
```

### Step 4.2: Deploy Docker Compose Manifest (`compose.yaml`)
Create `/opt/homelab/portainer/compose.yaml`:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data:/data
    ports:
      - "127.0.0.1:9000:9000"  # HTTP WebUI & API (bound to localhost for reverse proxy)
      - "127.0.0.1:9443:9443"  # Direct HTTPS fallback
    networks:
      - ingress-net

networks:
  ingress-net:
    external: true
```

> **Security Note on Read-Only Socket**: Mounting `/var/run/docker.sock:ro` provides read-only inspection access. However, because Docker socket operations inherently control host containers, administrative privileges within the Portainer WebUI equate to root-equivalent host access. Always secure this interface behind private networks and robust passwords.

Launch the container:

```bash
sudo docker compose up -d
```

---

## 5. Security & Reverse Proxy Ingress

### Step 5.1: Initial Setup Window Protection
Portainer features a security timeout: if an administrative account is not created within **5 minutes** of container spin-up, the setup wizard automatically locks to prevent unauthorized interception.

If locked out, simply restart the container to reset the 5-minute timer:
```bash
sudo docker compose restart portainer
```

### Step 5.2: Reverse Proxy Configuration (Nginx Proxy Manager)
1. In NPM, navigate to **Hosts** ➔ **Proxy Hosts** ➔ **Add Proxy Host**:
   * **Domain Names**: `portainer.home.lab`
   * **Scheme**: `http`
   * **Forward Hostname / IP**: `portainer` (or `127.0.0.1`)
   * **Forward Port**: `9000`
   * Check **Block Common Exploits** and **Websockets Support** (essential for web terminal console sessions).
2. Under the **SSL** tab:
   * Select your SSL Wildcard or local certificate.
   * Check **Force SSL** and **HTTP/2 Support**.
3. Save the proxy host.

---

## 6. Initial Configuration & Onboarding

1. Open your browser and navigate to `https://portainer.home.lab` (or `http://<SERVER_IP>:9000`).
2. Set a strong password (minimum 12 characters) for the primary `admin` user.
3. On the environment selection screen, choose **Get Started with the environment you're currently running in** (`local`).
4. You will be redirected to the main dashboard displaying container statuses, network drivers, and volume distributions.

---

## 7. Operational Maintenance & Container Updates

Because Portainer cannot gracefully stop and re-create its own container from within its own WebUI, update Portainer from the host shell:

```bash
cd /opt/homelab/portainer

# 1. Pull the latest Portainer CE image
sudo docker compose pull

# 2. Re-create the container with identical volume bindings
sudo docker compose up -d --remove-orphans

# 3. Clean up the obsolete image
sudo docker image prune -f
```

---

## 8. Operational Troubleshooting Matrix

| Symptom / Event | Root Cause | Remediation Procedure |
| :--- | :--- | :--- |
| **"Your Portainer instance timed out for security purposes"** | The initial setup wizard was open for longer than 5 minutes without creating an admin account. | Restart the container using `docker compose restart portainer` and reload the webpage within 5 minutes. |
| **Container Console / Exec Fails with "Disconnected"** | Reverse proxy is dropping WebSocket connection upgrade headers during terminal sessions. | Ensure **Websockets Support** is enabled in NPM, or add `proxy_set_header Upgrade $http_upgrade;` and `proxy_set_header Connection "upgrade";` to your Nginx configuration. |
| **`Got permission denied while trying to connect to the Docker daemon socket`** | Mismatched permissions or missing access to `/var/run/docker.sock`. | Verify that the host Docker daemon is running via `sudo systemctl status docker` and that `/var/run/docker.sock` has `rw-rw----` permissions assigned to group `docker`. |
| **Stack deployment error: "Compose file version is obsolete"** | Legacy `version: '3'` headers in user manifests. | Remove the top-level `version:` key from your Compose YAML file; modern Docker Compose V2 plugins interpret syntax without explicit version headers. |
