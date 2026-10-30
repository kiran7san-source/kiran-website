Tailscale is one of the best choices for a home server. It gives you secure remote access without exposing SSH, Jellyfin, or qBittorrent directly to the internet or configuring port forwarding.

---

# What you'll get

After setup, you'll be able to access your server from anywhere using its Tailscale IP or MagicDNS hostname.

For example:

* SSH:

  ```text
  ssh kiran@100.x.x.x
  ```

* Jellyfin:

  ```text
  http://100.x.x.x:8096
  ```

* qBittorrent:

  ```text
  http://100.x.x.x:8080
  ```

* Samba (optional, usually slower over the internet):

  ```text
  \\100.x.x.x\Media
  ```

---

# Step 1: Install Tailscale

Run:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Verify the installation:

```bash
tailscale version
```

---

# Step 2: Start the service

```bash
sudo systemctl enable tailscaled
sudo systemctl start tailscaled
```

Check:

```bash
sudo systemctl status tailscaled
```

You should see:

```text
active (running)
```

---

# Step 3: Authenticate

Run:

```bash
sudo tailscale up
```

It will display a URL similar to:

```text
https://login.tailscale.com/...
```

Open it in your browser and log in with the account you want to use (Google, Microsoft, GitHub, Apple, etc.).

Once approved, the server joins your private Tailscale network (tailnet).

---

# Step 4: Verify

Run:

```bash
tailscale status
```

Example:

```text
100.87.25.10   homeserver   linux
```

Find your Tailscale IP:

```bash
tailscale ip -4
```

Example:

```text
100.87.25.10
```

---

# Step 5: Install Tailscale on your other devices

Install it on:

* MacBook
* Windows laptop
* Phone (optional)

Log in with the **same Tailscale account**.

After a few seconds, all devices should appear in:

```bash
tailscale status
```

---

# Step 6: Test SSH

From your Mac:

```bash
ssh kiran@100.x.x.x
```

Replace `100.x.x.x` with the server's Tailscale IP.

You should be able to log in exactly as if you were on your home network.

---

# Step 7: Access Jellyfin remotely

Open:

```text
http://100.x.x.x:8096
```

No router port forwarding is required.

---

# Step 8: Access qBittorrent

Open:

```text
http://100.x.x.x:8080
```

Again, no router changes are needed.

---

# Step 9: Enable MagicDNS (recommended)

In the Tailscale admin console:

1. Open the **DNS** settings.
2. Enable **MagicDNS**.

Then instead of using the IP, you can connect using a hostname such as:

```text
http://homeserver:8096
```

or, depending on your tailnet:

```text
http://homeserver.your-tailnet.ts.net:8096
```

This avoids remembering IP addresses.

---

# Step 10: Keep SSH restricted

Since you'll use Tailscale, it's a good idea to **avoid exposing SSH to the public internet**.

Check that your router **does not** have port **22** forwarded. Tailscale will handle remote SSH access securely.

---

# Step 11: Optional — Enable Tailscale SSH

Tailscale has its own SSH feature that lets you log in without opening SSH on your LAN. Since you've already configured OpenSSH and it works well, I recommend **keeping OpenSSH** for now. We can revisit Tailscale SSH later if you want centralized access control.

---

# Step 12: Verify startup

Reboot the server:

```bash
sudo reboot
```

After it comes back up, confirm Tailscale is running:

```bash
tailscale status
```

It should reconnect automatically.

---

## Recommended security settings

Once Tailscale is working, I recommend:

* Keep **Jellyfin (8096)** and **qBittorrent (8080)** accessible only via your local network and Tailscale. Don't forward these ports on your router.
* Use a strong password for qBittorrent.
* Keep Ubuntu updated regularly:

  ```bash
  sudo apt update && sudo apt upgrade
  ```
* Back up your Docker Compose files and persistent data (e.g. `/opt/stacks` and your container config directories).

That gives you secure remote access without exposing your services directly to the internet.
