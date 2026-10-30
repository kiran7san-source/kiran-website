Ubuntu Server is an excellent choice for a home server, especially if you want a stable Docker host for Immich, Jellyfin, qBittorrent, and Samba. Since you have already performed a fresh installation, I recommend installing Docker using Docker's official repository rather than Ubuntu's `docker.io` package. This ensures you always get the latest stable Docker Engine and Compose plugin.

## Step 1: Update the system

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

Reconnect via SSH after the reboot.

---

## Step 2: Install prerequisites

```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
```

---

## Step 3: Add Docker's official GPG key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

---

## Step 4: Add the Docker repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update package lists:

```bash
sudo apt update
```

---

## Step 5: Install Docker Engine

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## Step 6: Verify Docker installation

```bash
docker --version
```

Example output:

```
Docker version 28.x.x
```

Verify Docker Compose:

```bash
docker compose version
```

---

## Step 7: Enable Docker at boot

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Check status:

```bash
sudo systemctl status docker
```

You should see:

```
Active: active (running)
```

Press **Q** to exit.

---

## Step 8: Allow your user to run Docker without sudo

Replace `$USER` automatically:

```bash
sudo usermod -aG docker $USER
```

Then either:

```bash
newgrp docker
```

or log out and back in.

Verify:

```bash
docker ps
```

This should work without `sudo`.

---

## Step 9: Test Docker

Run the official hello-world container:

```bash
docker run hello-world
```

You should see:

```
Hello from Docker!
```

---

## Step 10: Verify Docker Compose

Create a test directory:

```bash
mkdir ~/docker-test
cd ~/docker-test
```

Create `compose.yml`:

```yaml
services:
  hello:
    image: hello-world
```

Run:

```bash
docker compose up
```

If it prints the Hello World message, Docker Compose is working correctly.

---

## Step 11: Create a directory structure

Since you're building a media server, I'd suggest organizing it like this:

```text
/opt
    ├── stacks
    │      ├── immich
    │      ├── jellyfin
    │      ├── qbittorrent
    │      ├── watchtower
    │      └── nginx-proxy-manager
    │
    └── docker-data
```

Create it:

```bash
sudo mkdir -p /opt/stacks
sudo mkdir -p /opt/docker-data
sudo chown -R $USER:$USER /opt/stacks /opt/docker-data
```

---

## Step 12: Check Docker storage location

```bash
docker info | grep "Docker Root Dir"
```

Expected:

```
Docker Root Dir: /var/lib/docker
```

Leave this on your SSD for the best performance.

---

### Next steps

Once Docker is installed, I recommend setting up your server in this order:

1. Mount your storage drives:

   * `/mnt/media`
   * `/mnt/photos`
   * `/mnt/torrents`
   * `/mnt/cloud`
2. Install Samba and configure your shares.
3. Deploy Jellyfin.
4. Deploy qBittorrent.
5. Deploy Immich.
6. Configure automatic updates (optional, e.g. Watchtower).
7. Set up backups for your Docker Compose files and Immich database.

This order minimizes permission issues and ensures all containers can access the correct storage locations.
