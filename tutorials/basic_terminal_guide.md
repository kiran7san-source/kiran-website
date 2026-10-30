
# Linux File Server Administration Guide

## Practical Linux System Administration Tutorial for an Ubuntu Home/File Server

**Platform:** Ubuntu Server 24.04 LTS
**Primary purpose:** File server, media server, Docker host and home server
**Skill level:** Beginner → Intermediate → Practical System Administration

---

# 1. Introduction

This guide is designed to help you **operate and understand a Linux file server from the terminal**.

Rather than simply giving commands to copy and paste, each section explains:

* What the command does
* Why you would use it
* How Linux represents files and directories
* How permissions and ownership work
* How to safely manipulate files
* How to manage disks and mounts
* How to manage Docker and Docker Compose
* How to monitor and maintain the server
* How to troubleshoot common problems
* How to perform routine maintenance
* What commands are safe and what commands require caution

The examples use paths from a typical home-server setup, including:

```text
/mnt/media
/mnt/cloud
/opt/stacks
/home/kiran
```

Your actual paths should always be verified before executing destructive commands.

---

# 2. The Linux Command-Line Philosophy

Linux administration revolves around a few basic concepts.

Most commands follow this general pattern:

```bash
command [options] [arguments]
```

For example:

```bash
ls -lah /mnt/media
```

Here:

* `ls` = command
* `-lah` = options
* `/mnt/media` = argument

Another example:

```bash
sudo chown -R kiran:kiran /mnt/media/Music
```

Here:

* `sudo` = execute with administrative privileges
* `chown` = change ownership
* `-R` = recursively process everything below the directory
* `kiran:kiran` = user and group
* `/mnt/media/Music` = target

---

# 3. Understanding `sudo`

Linux separates ordinary users from the administrator account, traditionally called `root`.

Check your current user:

```bash
whoami
```

Check your groups:

```bash
groups
```

Run a command as administrator:

```bash
sudo command
```

For example:

```bash
sudo ls /root
```

You will normally be asked for your password.

## Why use sudo?

Avoid logging in as root for normal administration.

Instead:

```bash
sudo systemctl restart docker
```

is preferable to logging into a root shell and doing:

```bash
systemctl restart docker
```

You can obtain a root shell when necessary:

```bash
sudo -i
```

The prompt will generally change from something such as:

```text
kiran@homeserver:~$
```

to:

```text
root@homeserver:~#
```

Be extremely careful when the prompt ends in `#`.

Commands executed as root can modify or delete almost anything.

Exit the root shell:

```bash
exit
```

---

# 4. Getting Help

One of the most important Linux skills is learning how to find information yourself.

## `man`

The traditional Linux manual:

```bash
man ls
```

Search for a keyword:

```bash
man rsync
```

Quit:

```text
q
```

## `--help`

Most commands provide a quick summary:

```bash
ls --help
```

```bash
rsync --help
```

```bash
docker --help
```

## Find the location of a command

```bash
which docker
```

or:

```bash
command -v docker
```

Example:

```bash
command -v rsync
```

---

# 5. Navigating the Filesystem

Linux uses a single filesystem hierarchy.

Unlike Windows:

```text
C:\
D:\
E:\
```

Linux has one root:

```text
/
```

Everything exists below `/`.

Important directories include:

```text
/
├── bin
├── boot
├── dev
├── etc
├── home
├── media
├── mnt
├── opt
├── proc
├── root
├── run
├── srv
├── sys
├── tmp
├── usr
└── var
```

For your server, the most important areas are likely:

```text
/home/kiran
/mnt/media
/mnt/cloud
/opt/stacks
/var/lib/docker
/etc
```

---

# 6. `pwd` — Where Am I?

Display the current directory:

```bash
pwd
```

Example:

```text
/home/kiran
```

This is important because many Linux commands operate relative to the current directory.

---

# 7. `ls` — Listing Files

Basic:

```bash
ls
```

Detailed:

```bash
ls -l
```

Show hidden files:

```bash
ls -la
```

Human-readable sizes:

```bash
ls -lh
```

Recommended general-purpose command:

```bash
ls -lah
```

Example:

```bash
ls -lah /mnt/media
```

---

# 8. Understanding `ls -l`

Suppose you see:

```text
drwxr-xr-x  5 kiran kiran 4096 Aug 24 10:30 Music
```

Break it down:

```text
d rwx r-x r-x
  │   │   │
  │   │   └── permissions for others
  │   └────── permissions for group
  └────────── permissions for owner
```

The first character tells you the object type.

```text
d   directory
-   regular file
l   symbolic link
```

The next nine characters represent permissions.

```text
rwx r-x r-x
```

These correspond to:

```text
owner   group   others
rwx     r-x     r-x
```

---

# 9. `cd` — Changing Directories

Move into a directory:

```bash
cd /mnt/media
```

Go into Music:

```bash
cd /mnt/media/Music
```

Go up one level:

```bash
cd ..
```

Go to your home directory:

```bash
cd ~
```

or simply:

```bash
cd
```

Go to the filesystem root:

```bash
cd /
```

Return to the previous directory:

```bash
cd -
```

---

# 10. Absolute vs Relative Paths

An absolute path begins with `/`.

Example:

```bash
/mnt/media/Music
```

It always refers to the same location.

A relative path depends on your current directory.

If you are in:

```text
/mnt/media
```

then:

```bash
Music
```

means:

```text
/mnt/media/Music
```

You should generally prefer **absolute paths for important administrative commands**, especially destructive ones.

For example:

```bash
sudo rm -rf /mnt/media/test
```

is much safer than:

```bash
sudo rm -rf test
```

when you are performing administration.

---

# 11. Creating Directories

Create one directory:

```bash
mkdir test
```

Create a directory including missing parents:

```bash
mkdir -p /mnt/media/test/example
```

The `-p` option means:

> Create parent directories if they don't already exist.

For example:

```bash
mkdir -p /mnt/media/Archive/2026/Photos
```

---

# 12. Creating Files

Create an empty file:

```bash
touch test.txt
```

Create a configuration file:

```bash
touch example.conf
```

Check it:

```bash
ls -l example.conf
```

Be aware that `touch` does not normally "create a text document" in the Windows sense. It creates an empty file or updates its timestamps if the file already exists.

---

# 13. Copying Files with `cp`

Copy one file:

```bash
cp source.txt destination.txt
```

Copy into a directory:

```bash
cp source.txt /mnt/media/
```

Copy a directory:

```bash
cp -r source_directory /mnt/media/
```

Preserve attributes:

```bash
cp -a source_directory /mnt/media/
```

For server administration, `cp -a` is usually preferable to plain `cp -r` when you want to preserve ownership, permissions, timestamps, symbolic links and other metadata.

---

# 14. Moving and Renaming with `mv`

Rename:

```bash
mv oldname.txt newname.txt
```

Move:

```bash
mv file.txt /mnt/media/
```

Move a directory:

```bash
mv /mnt/media/test /mnt/media/Archive/
```

Linux does not have a separate rename command. Renaming is performed with `mv`.

---

# 15. Deleting Files with `rm`

Delete a file:

```bash
rm file.txt
```

Delete an empty directory:

```bash
rmdir directory
```

Delete a directory and its contents:

```bash
rm -r directory
```

Force deletion:

```bash
rm -f file.txt
```

Recursive + force:

```bash
rm -rf directory
```

## ⚠️ `rm -rf`

This is one of the most dangerous commands on Linux.

For example:

```bash
sudo rm -rf /mnt/media/test
```

will permanently remove the directory and its contents.

Linux generally has no recycle bin for commands such as:

```bash
rm
```

Before using `rm -rf`, verify the target:

```bash
ls -lah /mnt/media/test
```

A useful habit:

```bash
echo /mnt/media/test
```

before executing a destructive command.

---

# 16. File and Directory Sizes

Check the size of a directory:

```bash
du -sh /mnt/media
```

Explanation:

```text
du  disk usage
-s  summary
-h  human readable
```

Check each immediate subdirectory:

```bash
du -h --max-depth=1 /mnt/media
```

Sort by size:

```bash
du -h --max-depth=1 /mnt/media | sort -h
```

This is extremely useful for finding what is consuming disk space.

---

# 17. Finding Large Files

Find files larger than 1 GB:

```bash
find /mnt/media -type f -size +1G
```

Find files larger than 10 GB:

```bash
find /mnt/media -type f -size +10G
```

Display their sizes:

```bash
find /mnt/media -type f -size +1G -exec ls -lh {} \;
```

---

# 18. Finding Files

Find all `.mp3` files:

```bash
find /mnt/media -type f -name "*.mp3"
```

Find directories named `Music`:

```bash
find /mnt/media -type d -name "Music"
```

Case-insensitive search:

```bash
find /mnt/media -iname "*.mp3"
```

Find files modified in the last 7 days:

```bash
find /mnt/media -type f -mtime -7
```

Find files older than 365 days:

```bash
find /mnt/media -type f -mtime +365
```

---

# 19. Wildcards

Linux shells support patterns.

```text
*     anything
?     one character
[abc] one character from a,b,c
```

Example:

```bash
ls *.mp3
```

means:

> List files ending in `.mp3`.

Another example:

```bash
rm *.tmp
```

means:

> Delete every `.tmp` file in the current directory.

Always check before deleting:

```bash
ls *.tmp
```

then:

```bash
rm *.tmp
```

---

# 20. File Permissions — The Core Concept

Linux permissions are based on three classes:

```text
Owner
Group
Others
```

And three basic permissions:

```text
r = read
w = write
x = execute
```

For directories, these have slightly different meanings.

## File

```text
r
```

Read contents.

```text
w
```

Modify contents.

```text
x
```

Execute the file as a program.

## Directory

```text
r
```

List directory contents.

```text
w
```

Create/delete/rename entries.

```text
x
```

Enter/traverse the directory.

This distinction is particularly important for Samba, Docker and shared media directories.

---

# 21. Numeric Permissions

Permissions can also be represented numerically.

```text
r = 4
w = 2
x = 1
```

Therefore:

```text
rwx = 7
rw- = 6
r-x = 5
r-- = 4
```

Three numbers represent:

```text
owner group others
```

For example:

```text
755
```

means:

```text
Owner:  rwx = 7
Group:  r-x = 5
Others: r-x = 5
```

So:

```bash
chmod 755 directory
```

means:

> Owner can read/write/enter; everyone else can read/enter.

---

# 22. `chmod` — Changing Permissions

Example:

```bash
chmod 644 file.txt
```

Results in:

```text
owner   rw-
group   r--
others  r--
```

This is a common permission for ordinary files.

For directories:

```bash
chmod 755 directory
```

is common.

---

# 23. Symbolic `chmod`

Instead of numbers, you can use letters.

Give owner execute permission:

```bash
chmod u+x script.sh
```

Remove write permission from others:

```bash
chmod o-w file.txt
```

Give group write permission:

```bash
chmod g+w directory
```

Remove all permissions for others:

```bash
chmod o-rwx file.txt
```

Give owner and group read/write:

```bash
chmod ug+rw file.txt
```

---

# 24. Recursive Permissions

Change permissions for a directory and everything below it:

```bash
chmod -R 755 /mnt/media/example
```

Be careful.

Blindly doing:

```bash
chmod -R 777 /mnt/media
```

is a bad practice.

It gives everyone read/write/execute access and can create security and operational problems.

---

# 25. Changing Ownership with `chown`

Check current ownership:

```bash
ls -ld /mnt/media/Music
```

Example:

```text
drwxr-xr-x  kiran kiran  Music
```

Here:

```text
owner = kiran
group = kiran
```

Change owner:

```bash
sudo chown kiran Music
```

Change owner and group:

```bash
sudo chown kiran:kiran Music
```

---

# 26. Recursive Ownership

To change ownership of everything below a directory:

```bash
sudo chown -R kiran:kiran /mnt/media/Music
```

This means:

> Change owner and group of the Music directory and all contents below it.

Verify:

```bash
ls -lah /mnt/media/Music
```

---

# 27. A Practical Ownership Example

Suppose:

```bash
ls -ld /mnt/media/Music
```

shows:

```text
drwxr-xr-x root root Music
```

but your normal user needs to manage the directory.

You could use:

```bash
sudo chown kiran:kiran /mnt/media/Music
```

If everything underneath also needs to belong to you:

```bash
sudo chown -R kiran:kiran /mnt/media/Music
```

Then verify:

```bash
ls -ld /mnt/media/Music
```

---

# 28. `chgrp` — Changing Only the Group

Change group:

```bash
sudo chgrp media /mnt/media/Music
```

Recursive:

```bash
sudo chgrp -R media /mnt/media/Music
```

This becomes useful when multiple services/users share files.

---

# 29. ACLs — Advanced Permissions

Traditional permissions are:

```text
owner
group
others
```

ACLs allow more detailed rules.

Check ACL:

```bash
getfacl /mnt/media/Music
```

Grant another user read/write/execute:

```bash
sudo setfacl -m u:username:rwx /mnt/media/Music
```

Grant recursively:

```bash
sudo setfacl -R -m u:username:rwx /mnt/media/Music
```

ACLs are useful when normal ownership/group permissions become insufficient.

Do not use ACLs unnecessarily. Keep the permission model simple whenever possible.

---

# 30. Special Permissions: `setgid` on Directories

For shared directories, the setgid bit can be useful.

Example:

```bash
sudo chmod g+s /mnt/media/shared
```

This causes newly created files/directories to inherit the directory's group in many normal use cases.

Check:

```bash
ls -ld /mnt/media/shared
```

You may see:

```text
drwxrwsr-x
```

The `s` indicates the setgid bit.

This is useful for collaborative directories.

---

# 31. Symbolic Links

Create a symbolic link:

```bash
ln -s /mnt/media/Music /home/kiran/Music
```

Now:

```bash
ls -l /home/kiran/Music
```

will show something similar to:

```text
Music -> /mnt/media/Music
```

Delete the link:

```bash
rm /home/kiran/Music
```

This does **not** delete the original `/mnt/media/Music`.

---

# 32. Reading Files

Display a small file:

```bash
cat file.txt
```

For configuration files:

```bash
cat /etc/fstab
```

For long files:

```bash
less /var/log/syslog
```

Use:

```text
Space    next page
b        previous page
q        quit
```

---

# 33. Viewing the Beginning and End of Files

First 10 lines:

```bash
head file.txt
```

Last 10 lines:

```bash
tail file.txt
```

Follow a changing log:

```bash
tail -f /var/log/syslog
```

This is extremely useful when monitoring services.

---

# 34. Searching Text with `grep`

Search for a word:

```bash
grep "error" logfile.txt
```

Case-insensitive:

```bash
grep -i "error" logfile.txt
```

Search recursively:

```bash
grep -R "UPLOAD_LOCATION" /opt/stacks
```

Show line numbers:

```bash
grep -n "error" logfile.txt
```

Combine with other commands:

```bash
docker logs immich_server 2>&1 | grep -i error
```

---

# 35. Pipes — One of the Most Important Linux Concepts

The pipe:

```text
|
```

takes the output of one command and sends it to another.

Example:

```bash
ls -lah /mnt/media | less
```

Another:

```bash
du -h --max-depth=1 /mnt/media | sort -h
```

Another:

```bash
ps aux | grep docker
```

This ability to combine small commands is one of Linux's greatest strengths.

---

# 36. Output Redirection

Write output to a file:

```bash
ls -lah /mnt/media > inventory.txt
```

Append instead:

```bash
ls -lah /mnt/media >> inventory.txt
```

Redirect errors:

```bash
command 2> errors.txt
```

Redirect both normal output and errors:

```bash
command > output.txt 2>&1
```

Modern shorthand:

```bash
command &> output.txt
```

---

# 37. `rsync` — The Preferred Tool for Large File Operations

For large datasets, `rsync` is often better than `cp`.

Basic:

```bash
rsync -av /source/ /destination/
```

Important:

```text
-a    archive mode
-v    verbose
```

A trailing `/` matters.

```bash
rsync -av /mnt/media/Music/ /backup/Music/
```

means:

> Synchronize the contents of Music.

Whereas:

```bash
rsync -av /mnt/media/Music /backup/
```

can result in:

```text
/backup/Music
```

being created.

---

# 38. Dry-Run Before `rsync`

For important data, first use:

```bash
rsync -avhn /source/ /destination/
```

Here:

```text
-n = dry run
```

Nothing is actually copied.

This is an excellent safety habit.

For example:

```bash
rsync -avhn /mnt/media/Music/ /backup/Music/
```

Review the proposed changes.

Then remove `-n`:

```bash
rsync -avh /mnt/media/Music/ /backup/Music/
```

---

# 39. Mirroring with `rsync --delete`

This command is powerful:

```bash
rsync -avh --delete /source/ /destination/
```

It makes the destination closely mirror the source.

If a file exists only on the destination, it can be deleted.

Therefore ALWAYS dry-run first:

```bash
rsync -avhn --delete /source/ /destination/
```

Only after verifying the output:

```bash
rsync -avh --delete /source/ /destination/
```

Never use `--delete` casually.

---

# 40. Monitoring Disk Space

Check filesystem usage:

```bash
df -h
```

Example output:

```text
Filesystem      Size  Used Avail Use%
/dev/sda        3.6T  2.1T  1.5T  59%
```

The important columns are:

```text
Size
Used
Avail
Use%
```

For your server, regularly check:

```bash
df -h
```

especially:

```text
/mnt/media
/mnt/cloud
/
```

---

# 41. Inode Usage

Sometimes a filesystem has free disk space but cannot create files because it has run out of inodes.

Check:

```bash
df -i
```

Look at:

```text
IUse%
```

This is particularly useful on filesystems containing millions of small files.

---

# 42. Disk and Mount Information

List disks:

```bash
lsblk
```

Filesystem information:

```bash
lsblk -f
```

This is one of the most useful commands when troubleshooting disks.

More detailed:

```bash
sudo blkid
```

---

# 43. Mounts

Show currently mounted filesystems:

```bash
findmnt
```

Check a particular path:

```bash
findmnt /mnt/media
```

Check:

```bash
mount | grep /mnt
```

---

# 44. `/etc/fstab`

Persistent mounts are normally configured in:

```text
/etc/fstab
```

View it:

```bash
cat /etc/fstab
```

Before changing it, create a backup:

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

After making changes, test:

```bash
sudo mount -a
```

If there is an error, fix it before rebooting.

This is an extremely important habit.

---

# 45. Never Assume a Disk Is Mounted

Before copying important data to:

```text
/mnt/media
```

verify:

```bash
findmnt /mnt/media
```

Why?

If the disk unexpectedly becomes unmounted, `/mnt/media` may simply become an ordinary directory on the root filesystem.

You could then accidentally copy many gigabytes onto `/`.

Therefore, before large file operations:

```bash
df -h /mnt/media
findmnt /mnt/media
```

---

# 46. Processes

List running processes:

```bash
ps aux
```

Search:

```bash
ps aux | grep docker
```

Interactive process monitor:

```bash
top
```

If installed:

```bash
htop
```

CPU and memory usage can often be identified quickly with:

```bash
top
```

---

# 47. System Memory

Check RAM:

```bash
free -h
```

Important columns:

```text
total
used
free
available
swap
```

Do not interpret Linux `free` memory exactly like Windows. Linux intentionally uses unused RAM for caching.

The `available` figure is usually more useful when assessing memory pressure.

---

# 48. System Uptime

```bash
uptime
```

Example:

```text
11:30:00 up 5 days, 3:22, 1 user, load average: ...
```

Useful for determining whether the server recently rebooted.

---

# 49. Reboot and Shutdown

Reboot:

```bash
sudo reboot
```

Shutdown:

```bash
sudo shutdown -h now
```

or:

```bash
sudo poweroff
```

Before rebooting a Docker server, it is often useful to check:

```bash
docker ps
```

and:

```bash
systemctl --failed
```

---

# 50. Systemd Services

Ubuntu uses `systemd`.

List services:

```bash
systemctl list-units --type=service
```

Check a service:

```bash
systemctl status docker
```

Start:

```bash
sudo systemctl start docker
```

Stop:

```bash
sudo systemctl stop docker
```

Restart:

```bash
sudo systemctl restart docker
```

Enable at boot:

```bash
sudo systemctl enable docker
```

Disable:

```bash
sudo systemctl disable docker
```

---

# 51. Failed Services

One of the first troubleshooting commands after a problem:

```bash
systemctl --failed
```

If something is listed, investigate it.

Example:

```bash
systemctl status docker
```

---

# 52. System Logs with `journalctl`

View recent system logs:

```bash
sudo journalctl
```

Recent logs:

```bash
sudo journalctl -n 100
```

Follow logs live:

```bash
sudo journalctl -f
```

Logs from the current boot:

```bash
sudo journalctl -b
```

Kernel messages:

```bash
sudo journalctl -k
```

Docker service logs:

```bash
sudo journalctl -u docker
```

Only recent Docker logs:

```bash
sudo journalctl -u docker -n 100
```

---

# 53. Troubleshooting a Server That Suddenly Rebooted

Check previous boots:

```bash
journalctl --list-boots
```

Look at the previous boot:

```bash
sudo journalctl -b -1
```

Kernel messages from previous boot:

```bash
sudo journalctl -k -b -1
```

Search for errors:

```bash
sudo journalctl -b -1 | grep -iE "error|fail|panic|oom"
```

This is particularly useful after an unexpected shutdown or reboot.

---

# 54. Package Management with APT

Update package information:

```bash
sudo apt update
```

Upgrade installed packages:

```bash
sudo apt upgrade
```

A common maintenance sequence:

```bash
sudo apt update
sudo apt upgrade
```

Remove unnecessary packages:

```bash
sudo apt autoremove
```

Clean package cache:

```bash
sudo apt clean
```

---

# 55. Check for Reboot Requirement

After updates:

```bash
test -f /var/run/reboot-required && echo "Reboot required"
```

If a reboot is required:

```bash
sudo reboot
```

---

# 56. Installing Software

Search:

```bash
apt search rsync
```

Install:

```bash
sudo apt install rsync
```

Check whether installed:

```bash
apt policy rsync
```

---

# 57. Docker Fundamentals

Docker allows applications to run inside containers.

Useful concepts:

```text
Image
Container
Volume
Network
Compose
```

An image is the template.

A container is a running instance.

A volume or bind mount provides persistent data.

Docker Compose defines how multiple containers work together.

---

# 58. Check Docker

```bash
docker version
```

Check daemon:

```bash
sudo systemctl status docker
```

List running containers:

```bash
docker ps
```

List all containers:

```bash
docker ps -a
```

---

# 59. Docker Images

List images:

```bash
docker images
```

Pull an image:

```bash
docker pull image:tag
```

Remove an unused image:

```bash
docker image prune
```

Remove unused images more aggressively:

```bash
docker image prune -a
```

Be cautious with aggressive cleanup.

---

# 60. Docker Containers

Start:

```bash
docker start container_name
```

Stop:

```bash
docker stop container_name
```

Restart:

```bash
docker restart container_name
```

Remove:

```bash
docker rm container_name
```

Force removal:

```bash
docker rm -f container_name
```

Do not remove containers merely because they are stopped unless you understand how they are recreated.

With Compose, the container is normally disposable because its configuration is defined in the Compose project.

---

# 61. Docker Logs

View logs:

```bash
docker logs container_name
```

Last 100 lines:

```bash
docker logs --tail 100 container_name
```

Follow logs:

```bash
docker logs -f container_name
```

With timestamps:

```bash
docker logs -t container_name
```

For troubleshooting:

```bash
docker logs --tail 200 container_name
```

is often a good starting point.

---

# 62. Inspecting Containers

Very useful:

```bash
docker inspect container_name
```

This can show:

* Mounts
* Networks
* Environment
* Restart policy
* Container configuration
* IP addresses

For mounts specifically:

```bash
docker inspect container_name --format '{{json .Mounts}}'
```

---

# 63. Entering a Container

If the container has Bash:

```bash
docker exec -it container_name bash
```

If it uses `sh`:

```bash
docker exec -it container_name sh
```

Exit:

```bash
exit
```

This is useful for examining the application's internal filesystem.

Remember that changes made inside a container may disappear when the container is recreated unless they are stored in a persistent volume or bind mount.

---

# 64. Docker Resource Usage

Check containers:

```bash
docker stats
```

This gives live CPU, memory, network and block I/O information.

One-shot:

```bash
docker stats --no-stream
```

Useful when investigating:

> Why is the server using so much CPU/RAM?

---

# 65. Docker Disk Usage

```bash
docker system df
```

This shows:

* Images
* Containers
* Local volumes
* Build cache

It is useful for determining whether Docker is consuming excessive disk space.

---

# 66. Docker Compose

If a service is defined in a Compose project:

```bash
cd /opt/stacks/example
```

Check the project:

```bash
docker compose ps
```

Start:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

Restart:

```bash
docker compose restart
```

View logs:

```bash
docker compose logs
```

Follow logs:

```bash
docker compose logs -f
```

Specific service:

```bash
docker compose logs -f service_name
```

---

# 67. Updating a Docker Compose Application

A typical workflow is:

```bash
cd /opt/stacks/example
```

Inspect current state:

```bash
docker compose ps
```

Pull new images:

```bash
docker compose pull
```

Recreate:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Then inspect logs:

```bash
docker compose logs --tail 100
```

Do not blindly update every Docker application at once.

For important applications, read their release notes and understand whether database migrations or configuration changes are required.

---

# 68. Understanding Bind Mounts

A Compose configuration might contain:

```yaml
volumes:
  - /mnt/cloud/immich:/data
```

This means:

```text
Host:
/mnt/cloud/immich

Container:
/data
```

The application sees:

```text
/data
```

but the actual persistent files are stored on the host at:

```text
/mnt/cloud/immich
```

This distinction is essential when backing up Docker applications.

---

# 69. Finding Docker Mounts

For a running container:

```bash
docker inspect container_name
```

Look for:

```text
Mounts
```

You can also use:

```bash
docker inspect container_name --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

This produces something similar to:

```text
/mnt/cloud/immich -> /data
```

---

# 70. Docker Compose Files Are Important

For every important application, know:

1. Where the Compose file is.
2. Where `.env` is stored.
3. Where persistent data is stored.
4. Which ports are used.
5. Which Docker networks are used.
6. Which databases are used.

For your server, Compose projects are organized under:

```text
/opt/stacks
```

You can inspect them with:

```bash
find /opt/stacks -maxdepth 2 -type f \( -name "compose.yml" -o -name "docker-compose.yml" \)
```

---

# 71. Searching Your Compose Configuration

Find references to a particular path:

```bash
grep -R "/mnt/cloud" /opt/stacks
```

Find references to an environment variable:

```bash
grep -R "UPLOAD_LOCATION" /opt/stacks
```

Find port mappings:

```bash
grep -R "ports:" /opt/stacks
```

---

# 72. Checking Which Process Uses a Port

Find what is listening:

```bash
sudo ss -tulpn
```

Search a particular port:

```bash
sudo ss -tulpn | grep ':8080'
```

This is very useful when Docker says:

```text
port is already allocated
```

---

# 73. Network Information

Show IP addresses:

```bash
ip addr
```

Show routes:

```bash
ip route
```

Test connectivity:

```bash
ping 8.8.8.8
```

Test DNS:

```bash
ping google.com
```

Show listening ports:

```bash
sudo ss -tulpn
```

---

# 74. DNS Troubleshooting

Check DNS resolution:

```bash
resolvectl status
```

Resolve a hostname:

```bash
resolvectl query example.com
```

For local services, verify that the hostname resolves to the expected address.

---

# 75. File Ownership Problems with Docker

A common Docker problem looks like:

```text
Permission denied
```

First inspect the host directory:

```bash
ls -ld /path/to/data
```

Then:

```bash
ls -lah /path/to/data
```

Inspect the container:

```bash
docker inspect container_name
```

Determine what user the container runs as.

Do not immediately solve every Docker permission problem using:

```bash
chmod -R 777
```

Instead determine:

```text
Which user?
Which group?
Which directory?
Which process?
Which mount?
```

Then assign the minimum required permissions.

---

# 76. Samba/File Sharing Permissions

For Samba shares, remember there are potentially two permission layers:

```text
Samba permissions
        +
Linux filesystem permissions
```

A Samba share may allow access while Linux permissions still deny it.

When troubleshooting:

```bash
ls -ld /path/to/share
```

Then:

```bash
namei -l /path/to/share
```

`namei` is particularly useful because it shows permissions for each directory component.

Example:

```bash
namei -l /mnt/media/Music
```

---

# 77. `namei` — A Very Useful Troubleshooting Command

Suppose:

```text
/mnt/media/Music
```

gives permission denied.

Run:

```bash
namei -l /mnt/media/Music
```

It allows you to see permissions for:

```text
/
mnt
media
Music
```

A user needs appropriate execute/traverse permission on parent directories to reach the target.

---

# 78. File Ownership Summary

Useful commands:

```bash
ls -l file
```

```bash
stat file
```

```bash
namei -l /path/to/file
```

```bash
getfacl file
```

Changing:

```bash
sudo chown user:group file
```

Permissions:

```bash
chmod 644 file
```

Directory:

```bash
chmod 755 directory
```

---

# 79. `stat` — Detailed File Information

Run:

```bash
stat file.txt
```

You can see:

* Size
* Permissions
* Owner
* Group
* Access time
* Modification time
* Change time
* Inode

This is useful when debugging unusual filesystem behaviour.

---

# 80. File Timestamps

Linux tracks several timestamps.

Commonly visible:

```text
Access
Modify
Change
```

View:

```bash
stat file.txt
```

Modify time:

```bash
touch file.txt
```

Set a particular timestamp:

```bash
touch -d "2026-01-01 12:00" file.txt
```

Be careful with timestamps when working with backup and synchronization tools.

---

# 81. Archives with `tar`

Create archive:

```bash
tar -czf backup.tar.gz /path/to/data
```

Options:

```text
c = create
z = gzip
f = file
```

List archive:

```bash
tar -tzf backup.tar.gz
```

Extract:

```bash
tar -xzf backup.tar.gz
```

Extract to a directory:

```bash
tar -xzf backup.tar.gz -C /destination
```

For very large datasets, `tar` is not automatically better than `rsync`. They serve different purposes.

---

# 82. Compression

Compress a file:

```bash
gzip file.txt
```

Decompress:

```bash
gunzip file.txt.gz
```

For directories, use `tar` together with compression.

---

# 83. Backup Philosophy

A backup should not simply mean:

> I copied the files somewhere.

A proper backup should consider:

```text
Data
Configuration
Databases
Docker Compose files
Credentials/secrets
Certificates
System configuration
Filesystem structure
Recovery procedure
```

For your server, particularly important areas include:

```text
/opt/stacks
/etc
/home/kiran
/mnt/cloud
/mnt/media
```

However, not every directory should be blindly backed up.

For example:

```text
/var/lib/docker
```

usually should not be treated as the primary source of Docker application configuration when your Compose files and persistent bind mounts are available.

---

# 84. Restic

If using Restic:

Check repository:

```bash
restic -r /path/to/repository snapshots
```

List snapshots:

```bash
restic -r /path/to/repository snapshots
```

Check repository integrity:

```bash
restic -r /path/to/repository check
```

List files in a snapshot:

```bash
restic -r /path/to/repository ls latest
```

Restore:

```bash
restic -r /path/to/repository restore latest --target /restore
```

Always understand where your repository is located before performing backup or restore operations.

---

# 85. Backup Verification

A backup is not trustworthy merely because the backup command completed successfully.

Periodically verify:

```bash
restic check
```

and perform test restores.

For `rsync` backups, compare source and destination:

```bash
rsync -avhn /source/ /destination/
```

If the output is empty or contains only expected differences, the two trees are synchronized.

---

# 86. External Backup Drives

Before using an external HDD:

```bash
lsblk -f
```

Identify the filesystem and device.

Then:

```bash
findmnt
```

Confirm where it is mounted.

For example:

```text
/mnt/backup
```

Then verify:

```bash
df -h /mnt/backup
```

Only after confirming the correct disk should you run a large `rsync`.

---

# 87. A Safe Backup Workflow

Before copying:

```bash
lsblk -f
```

Then:

```bash
findmnt /mnt/backup
```

Then:

```bash
df -h /mnt/backup
```

Check destination:

```bash
ls -lah /mnt/backup
```

Perform dry run:

```bash
rsync -avhn /source/ /mnt/backup/
```

Review.

Then execute:

```bash
rsync -avh /source/ /mnt/backup/
```

This workflow prevents many accidental backup mistakes.

---

# 88. Checking Disk Health

Install tools if necessary:

```bash
sudo apt install smartmontools
```

List disks:

```bash
lsblk
```

Check SMART information:

```bash
sudo smartctl -a /dev/sda
```

For an NVMe drive, you may use:

```bash
sudo smartctl -a /dev/nvme0
```

SMART support varies by device and USB enclosure.

For a USB external disk, the enclosure may prevent SMART passthrough.

---

# 89. Checking Kernel Messages for Disk Errors

Search:

```bash
sudo journalctl -k | grep -iE "error|fail|ata|nvme|i/o"
```

Or:

```bash
dmesg | grep -iE "error|fail|ata|nvme|i/o"
```

Look especially for:

```text
I/O error
UNC
reset
timeout
filesystem error
EXT4-fs error
```

Repeated disk errors should never be ignored.

---

# 90. Filesystem Errors

For an ext4 filesystem, you may encounter:

```text
EXT4-fs error
```

Do not immediately run filesystem repair on a mounted filesystem.

Filesystem checks generally need the filesystem unmounted.

For example:

```bash
sudo umount /mnt/media
```

Then, depending on the situation:

```bash
sudo fsck.ext4 /dev/sdX1
```

**Never blindly substitute a device name.**

Verify using:

```bash
lsblk -f
```

Running `fsck` against the wrong partition can cause serious damage.

---

# 91. Monitoring Disk Usage

A useful routine:

```bash
df -h
```

Then:

```bash
sudo du -xhd1 /mnt/media | sort -h
```

And:

```bash
sudo du -xhd1 /mnt/cloud | sort -h
```

The `-x` option prevents `du` from crossing onto other filesystems.

This is useful when you want to know what is consuming space on a particular filesystem.

---

# 92. Finding Deleted Files Still Consuming Disk Space

Sometimes:

```bash
df -h
```

shows a lot of usage, but:

```bash
du -sh /
```

doesn't seem to explain it.

A process may still have an open deleted file.

Check:

```bash
sudo lsof +L1
```

This can reveal deleted files that are still held open by processes.

Restarting the relevant process may release the space.

---

# 93. Open Files

See which processes are accessing a directory:

```bash
sudo lsof +D /mnt/media
```

For large directories this can be expensive.

A more targeted approach:

```bash
sudo lsof /mnt/media/somefile
```

---

# 94. Cron and Scheduled Tasks

View your user's cron jobs:

```bash
crontab -l
```

Edit:

```bash
crontab -e
```

System-wide scheduled tasks may be found in:

```text
/etc/cron.d
/etc/cron.daily
/etc/cron.weekly
/etc/cron.monthly
```

Systemd timers are increasingly common.

List them:

```bash
systemctl list-timers
```

---

# 95. Checking Scheduled Backup Jobs

For a backup script, first find the scheduler:

```bash
systemctl list-timers
```

and:

```bash
crontab -l
```

Then inspect the script itself:

```bash
sudo less /path/to/backup-script.sh
```

Test manually:

```bash
sudo /path/to/backup-script.sh
```

Never assume an automated backup is working just because the timer exists.

Check its logs.

---

# 96. Shell Scripts

A basic script:

```bash
#!/bin/bash

echo "Starting backup"

rsync -avh /source/ /backup/

echo "Backup complete"
```

Make executable:

```bash
chmod +x backup.sh
```

Run:

```bash
./backup.sh
```

Or:

```bash
bash backup.sh
```

---

# 97. Safer Shell Scripts

A good starting point:

```bash
#!/bin/bash

set -euo pipefail

echo "Starting backup"

rsync -avh /source/ /backup/

echo "Backup complete"
```

`set -e` stops on errors.

`set -u` detects unset variables.

`pipefail` makes pipelines fail if an earlier command fails.

For important automated backup scripts, this is a valuable pattern.

---

# 98. Environment Variables

View an environment variable:

```bash
echo "$HOME"
```

Examples:

```bash
echo "$USER"
```

```bash
echo "$PATH"
```

List all:

```bash
env
```

Docker Compose frequently uses `.env` files.

Treat these carefully because they may contain passwords and secrets.

---

# 99. Editing Configuration Files

A beginner-friendly editor:

```bash
nano /etc/fstab
```

Save:

```text
Ctrl+O
```

Press Enter.

Exit:

```text
Ctrl+X
```

For Docker Compose:

```bash
nano /opt/stacks/example/compose.yml
```

Before editing an important configuration file:

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

---

# 100. Checking Configuration Before Restarting

Never make a configuration change and immediately restart a critical service if you can validate first.

For Docker Compose:

```bash
docker compose config
```

This checks and renders the Compose configuration.

Then:

```bash
docker compose up -d
```

For Nginx:

```bash
nginx -t
```

if Nginx is installed directly.

---

# 101. Docker Networks

List networks:

```bash
docker network ls
```

Inspect:

```bash
docker network inspect network_name
```

This can help diagnose:

* Containers unable to communicate
* Incorrect network configuration
* Reverse proxy issues
* Database connectivity problems

---

# 102. Docker Volumes

List:

```bash
docker volume ls
```

Inspect:

```bash
docker volume inspect volume_name
```

Remove an unused volume:

```bash
docker volume prune
```

Be cautious.

A volume may contain important application data even if it appears unused.

---

# 103. Docker Cleanup

First inspect:

```bash
docker system df
```

Then potentially:

```bash
docker image prune
```

Remove stopped containers:

```bash
docker container prune
```

Remove unused networks:

```bash
docker network prune
```

Do not routinely run:

```bash
docker system prune -a --volumes
```

on a production/home server without understanding exactly what will be removed.

---

# 104. Common Docker Troubleshooting Sequence

When an application stops working:

### Step 1 — Is the container running?

```bash
docker ps -a
```

### Step 2 — Check its status

```bash
docker inspect container_name
```

### Step 3 — Read logs

```bash
docker logs --tail 200 container_name
```

### Step 4 — Check resources

```bash
docker stats --no-stream
```

### Step 5 — Check disk space

```bash
df -h
```

### Step 6 — Check Docker

```bash
systemctl status docker
```

### Step 7 — Check the Compose project

```bash
cd /opt/stacks/project
docker compose ps
docker compose logs --tail 200
```

---

# 105. Common Linux Troubleshooting Sequence

When something is wrong, avoid randomly changing things.

Use this order:

```text
1. What exactly is failing?
2. Is the server running?
3. Is the filesystem mounted?
4. Is there enough disk space?
5. Is the service running?
6. Is the port listening?
7. What do the logs say?
8. Are permissions correct?
9. Is DNS/networking working?
10. What changed recently?
```

Useful commands:

```bash
uptime
```

```bash
df -h
```

```bash
lsblk -f
```

```bash
systemctl --failed
```

```bash
docker ps -a
```

```bash
sudo ss -tulpn
```

```bash
sudo journalctl -p err -b
```

---

# 106. The `journalctl -p err` Shortcut

Show errors from the current boot:

```bash
sudo journalctl -p err -b
```

Show warnings and errors:

```bash
sudo journalctl -p warning -b
```

Previous boot:

```bash
sudo journalctl -p err -b -1
```

This is an excellent first command when investigating a problem.

---

# 107. Checking Server Health

A useful quick health check:

```bash
echo "=== Uptime ==="
uptime

echo
echo "=== Memory ==="
free -h

echo
echo "=== Disk ==="
df -h

echo
echo "=== Failed Services ==="
systemctl --failed

echo
echo "=== Docker ==="
docker ps

echo
echo "=== Docker Disk Usage ==="
docker system df
```

This gives a quick overview without changing anything.

---

# 108. Checking Your Main Storage

For your media storage:

```bash
df -h /mnt/media
```

```bash
findmnt /mnt/media
```

```bash
ls -ld /mnt/media
```

```bash
sudo du -xhd1 /mnt/media | sort -h
```

For cloud/application storage:

```bash
df -h /mnt/cloud
```

```bash
findmnt /mnt/cloud
```

```bash
sudo du -xhd1 /mnt/cloud | sort -h
```

---

# 109. Checking Your Docker Stack Directory

```bash
ls -lah /opt/stacks
```

Find Compose files:

```bash
find /opt/stacks -type f \( -name "compose.yml" -o -name "docker-compose.yml" \)
```

Find environment files:

```bash
find /opt/stacks -type f -name ".env"
```

Find large files:

```bash
sudo find /opt/stacks -type f -size +100M -exec ls -lh {} \;
```

---

# 110. Useful Command Combinations

## Find the biggest directories

```bash
sudo du -xhd1 /mnt/media | sort -h
```

## Find biggest files

```bash
sudo find /mnt/media -type f -printf '%s %p\n' | sort -n | tail -20
```

## Find errors in Docker logs

```bash
docker logs container_name 2>&1 | grep -iE "error|failed|fatal"
```

## Find services that failed

```bash
systemctl --failed
```

## Find what is listening on a port

```bash
sudo ss -tulpn | grep ':8080'
```

## Find a configuration value

```bash
grep -R "keyword" /opt/stacks
```

---

# 111. Safe Administration Rules

## Rule 1 — Verify before deleting

Instead of:

```bash
rm -rf /path
```

first:

```bash
ls -lah /path
```

---

## Rule 2 — Verify mounts

Before copying large amounts of data:

```bash
findmnt /mnt/media
```

---

## Rule 3 — Dry-run `rsync`

```bash
rsync -avhn ...
```

---

## Rule 4 — Never casually use `chmod 777`

Avoid:

```bash
chmod -R 777 /path
```

Understand the permission problem first.

---

## Rule 5 — Back up configuration before modifying it

Example:

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

---

## Rule 6 — Don't reboot to fix everything

First inspect:

```bash
systemctl --failed
```

```bash
journalctl -p err -b
```

---

## Rule 7 — Don't delete Docker data casually

Avoid blindly deleting:

```text
/var/lib/docker
```

or Docker volumes.

Determine where persistent data actually resides first.

---

# 112. A Practical Daily Administration Routine

You do not need to run dozens of commands every day.

A simple check is:

```bash
uptime
df -h
free -h
systemctl --failed
docker ps
```

If everything looks normal, stop there.

---

# 113. Weekly Maintenance

A reasonable weekly routine:

```bash
sudo apt update
```

Then:

```bash
sudo apt upgrade
```

Check disk:

```bash
df -h
```

Check Docker:

```bash
docker system df
```

Check failed services:

```bash
systemctl --failed
```

Check recent errors:

```bash
sudo journalctl -p err -b
```

Check backups:

```bash
restic snapshots
```

or your appropriate backup command.

---

# 114. Monthly Maintenance

Periodically:

### Review disk usage

```bash
sudo du -xhd1 /mnt/media | sort -h
```

### Review cloud/application storage

```bash
sudo du -xhd1 /mnt/cloud | sort -h
```

### Check Docker images

```bash
docker images
```

### Check Docker storage

```bash
docker system df
```

### Check SMART health

```bash
sudo smartctl -a /dev/sdX
```

### Verify backups

Perform an actual test restore.

---

# 115. Emergency Troubleshooting Checklist

If the server appears inaccessible:

```bash
ping SERVER_IP
```

If reachable, SSH:

```bash
ssh kiran@SERVER_IP
```

Then:

```bash
uptime
```

Check disks:

```bash
df -h
```

Check mounts:

```bash
lsblk -f
```

Check services:

```bash
systemctl --failed
```

Check Docker:

```bash
systemctl status docker
```

Check containers:

```bash
docker ps -a
```

Check errors:

```bash
sudo journalctl -p err -b
```

Check network:

```bash
ip addr
ip route
```

Check ports:

```bash
sudo ss -tulpn
```

---

# 116. The Most Important Commands to Learn

If you are learning Linux, prioritize these commands.

## Filesystem

```bash
pwd
ls
cd
mkdir
touch
cp
mv
rm
find
```

## File information

```bash
stat
du
df
lsblk
findmnt
```

## Permissions

```bash
chmod
chown
chgrp
getfacl
setfacl
namei
```

## Text

```bash
cat
less
head
tail
grep
```

## Shell

```bash
|
>
>>
&&
```

## Processes

```bash
ps
top
htop
kill
```

## System

```bash
systemctl
journalctl
uname
uptime
free
```

## Networking

```bash
ip
ss
ping
resolvectl
```

## Docker

```bash
docker ps
docker logs
docker inspect
docker exec
docker stats
docker system df
docker compose
```

## Backup

```bash
rsync
restic
```

---

# 117. Command Cheat Sheet

| Task                | Command                         |
| ------------------- | ------------------------------- |
| Current directory   | `pwd`                           |
| List files          | `ls -lah`                       |
| Change directory    | `cd /path`                      |
| Create directory    | `mkdir -p /path`                |
| Copy                | `cp -a source destination`      |
| Move/rename         | `mv source destination`         |
| Delete file         | `rm file`                       |
| Delete directory    | `rm -r directory`               |
| File size           | `du -sh file`                   |
| Directory size      | `du -sh directory`              |
| Disk usage          | `df -h`                         |
| Find file           | `find /path -name "file"`       |
| File permissions    | `ls -l`                         |
| Change permissions  | `chmod 755 file`                |
| Change owner        | `chown user:group file`         |
| Recursive ownership | `chown -R user:group directory` |
| Detailed file info  | `stat file`                     |
| Mount information   | `findmnt`                       |
| Disk information    | `lsblk -f`                      |
| Running processes   | `ps aux`                        |
| Memory              | `free -h`                       |
| Service status      | `systemctl status service`      |
| Failed services     | `systemctl --failed`            |
| System logs         | `journalctl`                    |
| Recent errors       | `journalctl -p err -b`          |
| Docker containers   | `docker ps -a`                  |
| Docker logs         | `docker logs container`         |
| Docker mounts       | `docker inspect container`      |
| Docker resources    | `docker stats`                  |
| Compose status      | `docker compose ps`             |
| Compose logs        | `docker compose logs`           |
| Compose start       | `docker compose up -d`          |
| Compose stop        | `docker compose down`           |
| Sync files          | `rsync -avh`                    |

---

# 118. A Recommended Mental Model for Your Server

When administering the server, think of it as five layers:

```text
┌──────────────────────────────────────┐
│ Applications                         │
│ Immich / Jellyfin / Navidrome / etc.│
├──────────────────────────────────────┤
│ Docker                               │
│ Containers / Images / Networks       │
├──────────────────────────────────────┤
│ Linux filesystem                     │
│ Files / Ownership / Permissions      │
├──────────────────────────────────────┤
│ Storage                              │
│ HDD / NVMe / ext4 / Mounts           │
├──────────────────────────────────────┤
│ Hardware                             │
│ CPU / RAM / Disk / UPS / Network     │
└──────────────────────────────────────┘
```

When something breaks, determine **which layer is actually failing**.

For example:

> Immich isn't working.

Do not immediately reinstall Immich.

Work downward:

```text
Immich application
       ↓
Docker container
       ↓
Docker network
       ↓
Host filesystem
       ↓
/mnt/cloud mounted?
       ↓
Disk healthy?
```

This approach is much more reliable than randomly changing configuration.

---

# 119. A Good Linux Administrator's Workflow

When faced with an unfamiliar problem:

### 1. Observe

```bash
uptime
df -h
free -h
docker ps -a
systemctl --failed
```

### 2. Reproduce

Determine exactly what is failing.

### 3. Inspect

```bash
journalctl
docker logs
ls -lah
findmnt
ss -tulpn
```

### 4. Form a hypothesis

For example:

> The application cannot access its storage directory.

### 5. Test the hypothesis

```bash
findmnt /mnt/cloud
ls -ld /mnt/cloud
namei -l /mnt/cloud
```

### 6. Make the smallest necessary change

Avoid changing multiple unrelated things.

### 7. Verify

Check the application again.

### 8. Record the solution

For a home server, maintaining a personal administration notebook is extremely useful.

---

# 120. Final Practical Routine

For your particular home server, the following compact routine is worth memorizing.

## Quick health check

```bash
uptime
free -h
df -h
systemctl --failed
docker ps
```

## Storage investigation

```bash
lsblk -f
findmnt
df -h
sudo du -xhd1 /mnt/media | sort -h
sudo du -xhd1 /mnt/cloud | sort -h
```

## Docker investigation

```bash
docker ps -a
docker stats --no-stream
docker system df
docker logs --tail 100 CONTAINER_NAME
```

## Service investigation

```bash
systemctl status SERVICE
sudo journalctl -u SERVICE -n 100
```

## Network investigation

```bash
ip addr
ip route
sudo ss -tulpn
```

## Permission investigation

```bash
ls -ld /path
namei -l /path
getfacl /path
```

## Backup investigation

```bash
df -h /backup
findmnt /backup
rsync -avhn /source/ /backup/
```

---

# 121. Golden Rules

Keep these principles in mind:

1. **Never delete what you haven't verified.**
2. **Always verify mounts before large file operations.**
3. **Use `rsync --dry-run` before important synchronization.**
4. **Understand ownership before changing permissions.**
5. **Avoid `chmod 777` as a generic solution.**
6. **Do not delete Docker volumes unless you know what they contain.**
7. **Read logs before changing configuration.**
8. **Back up configuration before modifying it.**
9. **Make one change at a time when troubleshooting.**
10. **Test backups by restoring data.**
11. **Monitor disk space before it becomes critical.**
12. **Treat `/etc`, `/opt/stacks`, application data and databases as important recovery information.**
13. **Use absolute paths for destructive operations.**
14. **When uncertain, inspect first and change later.**
15. **A command succeeding does not necessarily mean the operation achieved what you intended. Always verify the result.**

---

# 122. The Most Important Lesson

Linux administration is less about memorizing hundreds of commands and more about understanding the relationship between:

```text
command
   ↓
filesystem
   ↓
permissions
   ↓
process
   ↓
service
   ↓
storage
   ↓
network
```

For example, if a Docker application reports:

```text
Permission denied
```

the solution isn't necessarily:

```bash
chmod -R 777 ...
```

Instead ask:

```text
Which process generated the error?
Which user is that process running as?
Which host directory is mounted?
Who owns that directory?
What permissions does it have?
What permissions do its parent directories have?
Is the filesystem mounted?
Is the filesystem read-only?
```

Those questions—and the ability to answer them using commands such as:

```bash
docker inspect
ls -l
namei -l
stat
getfacl
findmnt
df -h
journalctl
```

are what turn command-line usage into actual Linux system administration.
