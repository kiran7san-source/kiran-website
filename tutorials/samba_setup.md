# Production Network Attached Storage: Multi-Platform Samba (SMB/CIFS) File Sharing on Linux

## 1. Overview & Architectural Background
Samba is the standard open-source re-implementation of the Server Message Block (SMB) and Common Internet File System (CIFS) networking protocols for UNIX and Linux environments. It delivers seamless, cross-platform file and print sharing across Windows, macOS, Linux, and mobile endpoints.

### Protocol Evolution & Core Mechanics
* **Protocol Dialects (SMBv1 vs. SMBv2/v3)**:
  * **SMBv1 (CIFS)**: Legacy protocol deprecated due to catastrophic vulnerabilities (e.g., WannaCry / EternalBlue) and high network chatty latency. Modern implementations explicitly disable SMBv1.
  * **SMBv2**: Introduced credit-based flow control, compounding requests, and substantial performance increases across high-latency networks.
  * **SMBv3**: Modern production standard featuring end-to-end transparent AES-128/256-GCM transport encryption, multi-channel aggregation (striping throughput across concurrent NICs), directory leasing, and secure dialect negotiation.
* **POSIX-to-NTFS Access Control Translation**: Samba translates Windows Security Identifiers (SIDs) and NT Access Control Lists (ACLs) into standard Linux POSIX user/group permissions (`rwxrwxrwx`) and extended POSIX ACLs.
* **Authentication Subsystems**: Samba maintains an independent, pass-through hash database (`/var/lib/samba/private/passdb.tdb`) decoupling SMB credentials from system UNIX shell passwords while maintaining local UID/GID account parity.

---

## 2. Production Homelab Use Cases
* **High-Throughput Local Storage**: Deliver direct network-share mounts to Windows desktop rigs, macOS workstations, and laptops over LAN.
* **Hypervisor & Container Ingress**: Serve media files to Jellyfin, Navidrome, and Bookorbit containers without copying multi-terabyte datasets across volumes.
* **Centralized Automated Ingest**: Provide accessible drop-folders for network scanners, local backup clients, and download daemons.
* **Granular Multi-User Separation**: Enforce private per-user root shares alongside shared public or multi-tenant media directories.

---

## 3. Prerequisites & Environment Planning
* Linux host (Ubuntu 22.04/24.04 LTS or Debian 11/12).
* Administrative `sudo` privileges.
* Static host IP (e.g., `192.168.1.50`).
* Dedicated storage array or directory mount (e.g., `/mnt/storage/shares`).

### Firewall & Port Requirements
Ensure the following ports are open across your local subnet:
* **TCP Port 445**: Direct SMB transport without NetBIOS (standard modern traffic).
* **TCP Port 139**: NetBIOS session service (legacy compatibility).
* **UDP Ports 137, 138**: NetBIOS name and datagram services.

---

## 4. Step-by-Step Implementation

### Step 4.1: Install Samba Packages
Install the core Samba daemon and management utilities:

```bash
sudo apt-get update
sudo apt-get install -y samba samba-common-bin
```

Verify service daemon availability:

```bash
smbd --version
```

### Step 4.2: Prepare Host Storage Directories & POSIX Groups
Establish dedicated system groups and configure the directory tree:

```bash
# 1. Create a dedicated storage access group
sudo groupadd smbgroup

# 2. Create sample directories for a shared media pool and a secure private share
sudo mkdir -p /mnt/storage/shares/{media,private}

# 3. Apply baseline POSIX group ownership and SGID bit
# Setting the SGID bit (2775) ensures newly created files inherit the parent group
sudo chown -R root:smbgroup /mnt/storage/shares
sudo chmod -R 2775 /mnt/storage/shares
```

### Step 4.3: Configure Users and Samba Passwords
Samba requires users to exist in the underlying Linux system before generating their SMB credentials. Create users with disabled shell access to prevent arbitrary SSH logins:

```bash
# 1. Create a non-login system user
sudo useradd -M -s /usr/sbin/nologin -g smbgroup kiran

# 2. Set the Samba authentication password for this user
sudo smbpasswd -a kiran

# 3. Enable the user within Samba passdb
sudo smbpasswd -e kiran
```

---

## 5. Hardened Configuration (`/etc/samba/smb.conf`)

Back up the distro default configuration and write a tuned, hardened definition:

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

Deploy the updated `/etc/samba/smb.conf`:

```ini
[global]
   workgroup = WORKGROUP
   server string = Homelab File Server
   server role = standalone server
   security = user
   map to guest = Bad User
   dns proxy = no

   # Protocol Tuning & Security Hardening
   server min protocol = SMB2_10
   server max protocol = SMB3
   client min protocol = SMB2_10
   client max protocol = SMB3
   smb encrypt = auto

   # Performance & Multi-Channel Optimizations
   server multi channel support = yes
   aio read size = 1
   aio write size = 1
   use sendfile = yes
   deadtime = 15

   # Apple macOS SMB Compatibility Enhancements
   vfs objects = catia fruit streams_xattr
   fruit:metadata = stream
   fruit:model = Macmini
   fruit:posix_rename = yes
   fruit:veto_appledouble = yes
   fruit:wipe_intentionally_left_blank_rfork = yes
   fruit:delete_empty_adfiles = yes

   # Disable Unnecessary NetBIOS & Printing Overhead
   load printers = no
   printing = bsd
   printcap name = /dev/null
   disable spoolss = yes

# --- Share Definitions ---

[Media]
   comment = Centralized Media Library
   path = /mnt/storage/shares/media
   browseable = yes
   read only = no
   guest ok = yes
   create mask = 0664
   directory mask = 0775
   force group = smbgroup

[Private]
   comment = Secure Personal Storage
   path = /mnt/storage/shares/private
   browseable = no
   read only = no
   guest ok = no
   valid users = kiran
   create mask = 0660
   directory mask = 0770
   force user = kiran
   force group = smbgroup
```

---

## 6. Syntax Validation & Service Activation

Verify configuration file syntax using `testparm`:

```bash
# Validate config syntax without warnings
testparm -s
```

Restart and enable the Samba services:

```bash
sudo systemctl restart smbd nmbd
sudo systemctl enable smbd nmbd
```

### Configure UFW Firewall (If Active)
Allow Samba traffic across your local subnet (adjust IP range to match your LAN):

```bash
sudo ufw allow proto tcp from 192.168.1.0/24 to any port 445
sudo ufw allow proto tcp from 192.168.1.0/24 to any port 139
sudo ufw allow proto udp from 192.168.1.0/24 to any port 137:138
```

---

## 7. Client Connection Walkthrough

* **Windows**:
  * Open File Explorer and enter `\\192.168.1.50\Media` in the address bar.
  * For private shares, map a persistent network drive and check **Connect using different credentials**, entering your assigned username and Samba password.
* **macOS**:
  * Open Finder, press `Cmd + K`, and enter `smb://192.168.1.50/Media`.
* **Linux (via `/etc/fstab`)**:
  Mount dynamically on boot via CIFS utils:
  ```bash
  sudo apt-get install -y cifs-utils
  # Mount command syntax:
  sudo mount -t cifs -o username=kiran,uid=1000,gid=1000 //192.168.1.50/Private /mnt/localshare
  ```

---

## 8. Operational Troubleshooting Matrix

| Symptom / Error | Root Cause | Remediation Procedure |
| :--- | :--- | :--- |
| **`Permission Denied` when writing via SMB** | Underlying Linux filesystem POSIX permissions conflict with Samba user access. | Verify the target path ownership matches the group: `sudo chown -R :smbgroup /mnt/storage/shares && sudo chmod -R 775 /mnt/storage/shares`. |
| **Windows: "Multiple connections to a server using more than one user name..."** | Windows credential manager caches previous guest or user sessions for the same server IP. | Open Windows Command Prompt and purge cached SMB connections via `net use * /delete /y`. |
| **macOS `.DS_Store` & AppleDouble files cluttering shares** | macOS writing metadata forks. | Verify `vfs objects = catia fruit streams_xattr` is declared under `[global]` in `/etc/samba/smb.conf`. |
| **Samba service fails to start** | Parse error in `/etc/samba/smb.conf`. | Execute `testparm` to identify invalid directive strings or unclosed brackets. |
