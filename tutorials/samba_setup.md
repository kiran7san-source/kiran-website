This is a good time to set up Samba properly. Since you'll be using it from **Windows**, **macOS**, and Docker containers, we'll create a clean layout that avoids permission issues later.

# Recommended Shared Folders

I recommend these shares:

| Linux Path   | Purpose                            | Share Name |
| ------------ | ---------------------------------- | ---------- |
| `/mnt/media` | Movies, TV Shows, Music            | `Media`    |
| `/mnt/cloud` | Documents, backups, personal files | `Cloud`    |
| `/srv/share` | Fast shared folder on the NVMe SSD | `Share`    |

Using `/srv/share` is preferable to creating another folder directly under `/mnt`, because `/srv` is the standard location for data served over the network.

---

# Step 1: Install Samba

```bash
sudo apt update
sudo apt install samba -y
```

Verify:

```bash
smbd --version
```

---

# Step 2: Create the NVMe shared folder

```bash
sudo mkdir -p /srv/share
```

Give yourself ownership:

```bash
sudo chown -R kiran:kiran /srv/share
```

Verify:

```bash
ls -ld /srv/share
```

---

# Step 3: Verify permissions

Check all three folders:

```bash
ls -ld /mnt/media
ls -ld /mnt/cloud
ls -ld /srv/share
```

If `/srv/share` isn't owned by `kiran`, fix it:

```bash
sudo chown -R kiran:kiran /srv/share
```

For your NTFS mounts, ownership is controlled by the `uid=` and `gid=` options in `/etc/fstab`, so you don't need to run `chown` on them.

---

# Step 4: Backup Samba configuration

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

---

# Step 5: Edit Samba configuration

```bash
sudo nano /etc/samba/smb.conf
```

Scroll to the **bottom** of the file and append:

```ini
[Media]
    path = /mnt/media
    browseable = yes
    writable = yes
    read only = no

    guest ok = no
    valid users = kiran

    create mask = 0664
    directory mask = 0775

[Cloud]
    path = /mnt/cloud
    browseable = yes
    writable = yes
    read only = no

    guest ok = no
    valid users = kiran

    create mask = 0664
    directory mask = 0775

[Share]
    path = /srv/share
    browseable = yes
    writable = yes
    read only = no

    guest ok = no
    valid users = kiran

    create mask = 0664
    directory mask = 0775
```

Save and exit.

---

# Step 6: Add a Samba password

This password is separate from your Linux login password (it can be the same if you choose).

```bash
sudo smbpasswd -a kiran
```

You'll be prompted to enter and confirm the password.

Enable the account:

```bash
sudo smbpasswd -e kiran
```

---

# Step 7: Check the configuration

```bash
testparm
```

If it reports:

```
Loaded services file OK.
```

the configuration is valid.

---

# Step 8: Restart Samba

```bash
sudo systemctl restart smbd
sudo systemctl enable smbd
```

Verify:

```bash
sudo systemctl status smbd
```

It should show `active (running)`.

---

# Step 9: Allow Samba through the firewall (if enabled)

Check whether UFW is active:

```bash
sudo ufw status
```

If it is active:

```bash
sudo ufw allow Samba
```

---

# Step 10: Find your server's IP address

```bash
hostname -I
```

Suppose it returns:

```
192.168.1.50
```

---

# Step 11: Connect from Windows

Open File Explorer and enter:

```
\\192.168.1.50
```

You should see:

* `Media`
* `Cloud`
* `Share`

Log in with:

* **Username:** `kiran`
* **Password:** (the one you set with `smbpasswd`)

---

# Step 12: Connect from macOS

In Finder:

**Go → Connect to Server…** (or press `⌘K`)

Enter:

```
smb://192.168.1.50
```

Authenticate with the same Samba username and password.

---

# Step 13: Test

Create a test file from each client:

```
Media/test.txt
Cloud/test.txt
Share/test.txt
```

Then verify on the server:

```bash
ls /mnt/media
ls /mnt/cloud
ls /srv/share
```

You should see the files.

---

# Optional: Improve compatibility with macOS

If you'll be using Finder frequently, Samba can advertise support for macOS metadata and Time Machine. Since you only mentioned using the Mac to copy files, this isn't required. We can enable Apple's SMB extensions later if you decide you need features like better Finder integration or Time Machine backups.

