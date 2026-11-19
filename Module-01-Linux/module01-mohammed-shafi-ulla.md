# Linux Hands-on Assessment – Module 01

**Name:** Mohammed Shafiulla  
**Environment:** Ubuntu 24.04 LTS – Killercoda  
**Assessment:** Linux Administration Hands-on (Tasks 1–10)  
**Status:** Completed

---

## About this assessment

I completed these tasks using an Ubuntu environment in Killercoda. I used the terminal to practise Linux commands and understand how they work in a real system.

This document contains the main commands, results and short notes from each task. I have kept it as a reference so I can use the commands again while practising Linux administration.

> **Note:** Killercoda sessions are temporary. The environment can reset after the session expires, so files, users, packages and services created during a lab may not remain available in a new session.

## Assessment summary

| Task | Topic | Status |
|---|---|---|
| 1 | Know your machine | Completed |
| 2 | Links and directory tree | Completed |
| 3 | Users, groups and permissions | Completed |
| 4 | Team folder and sudo permissions | Completed |
| 5 | Process and service management | Completed |
| 6 | Custom systemd service | Completed |
| 7 | Package management | Completed |
| 8 | Disk management and LVM | Completed |
| 9 | Networking and SSH | Completed |
| 10 | DNS, HTTP server and nftables | Completed |

---

# Task 1 – Know your machine

### Objective

Check the Linux kernel, system uptime, process running as PID 1 and system boot time.

### Commands used

```bash
uname -r
uname
uptime
ps -p 1 -o pid,comm
systemd-analyze
systemd-analyze blame | head -n 3
```

I also saved the information into a text file:

```bash
uname -r > ~/system-info.txt
uptime >> ~/system-info.txt
ps -p 1 -o pid,comm >> ~/system-info.txt
systemd-analyze >> ~/system-info.txt
systemd-analyze blame | head -n 3 >> ~/system-info.txt
cat ~/system-info.txt
```

### Results

| Check | Result |
|---|---|
| Kernel release | `6.8.0-147-generic` |
| System name | `Linux` |
| PID 1 | `systemd` |
| Boot time | `9.141 seconds` |
| Kernel startup | `3.007 seconds` |
| Userspace startup | `6.133 seconds` |

Top three services shown by `systemd-analyze blame`:

```text
2.246s systemd-networkd-wait-online.service
1.816s containerd.service
1.657s podman-clean-transient.service
```

### What I understood

- `uname` displays the system name. `uname -r` displays the kernel release.
- `uptime` shows how long the system has been running and the load average.
- PID 1 is the first userspace process. In this system it is `systemd`.
- `systemd-analyze` gives information about system startup time.
- `systemd-analyze blame` helps identify services that took time during startup.

---

# Task 2 – Links and directory tree

### Objective

Create a directory, make a hard link and a symbolic link, then check what happens when the original filename is removed. Also check recently modified files and disk usage.

### Commands used

```bash
mkdir -p ~/lab2/docs
echo "linux is fun" > ~/lab2/docs/notes.txt
cat ~/lab2/docs/notes.txt

ln ~/lab2/docs/notes.txt ~/lab2/notes.hard
ln -s ~/lab2/docs/notes.txt ~/lab2/notes.link

cd ~/lab2
ls -li docs/notes.txt notes.hard notes.link

rm docs/notes.txt
cat notes.hard
cat notes.link

find /etc/ -type f -mtime -7
df -h /
df -i /
```

### Results and observations

Before removing the original file, the inode information was:

```text
810210 -rw-r--r-- 2 root root 13 ... docs/notes.txt
810210 -rw-r--r-- 2 root root 13 ... notes.hard
810211 lrwxrwxrwx 1 root root 25 ... notes.link -> /root/lab2/docs/notes.txt
```

After removing `docs/notes.txt`:

- `cat notes.hard` still displayed `linux is fun`.
- `cat notes.link` returned `No such file or directory`.

Disk information recorded during the lab:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        19G  5.5G   13G  30% /
```

```text
Filesystem      Inodes  IUsed   IFree IUse%
/dev/vda1      2490368 163187 2327181    7%
```

The recent files found under `/etc` were:

```text
/etc/killercoda/host
/etc/systemd/timesyncd.conf
/etc/systemd/resolved.conf
```

### What I understood

- A hard link points to the same inode as the original file. It can still access the file data after the original filename is removed.
- A symbolic link stores a path to another file. If that target is removed, the symbolic link becomes broken.
- `ls -li` displays inode numbers.
- `df -h` checks disk space in a readable format.
- `df -i` checks inode usage.

**Lab note:** The text created in this task was `linux is fun` in lowercase.

---

# Task 3 – Users, groups and permissions

### Objective

Create users and a group, set directory and file permissions, and give a specific user access through ACL.

### Commands used

```bash
groupadd devteam
useradd -m dev1
useradd -m qa1
usermod -aG devteam dev1

id dev1
id qa1

mkdir -p /opt/project
chown root:devteam /opt/project/
chmod 770 /opt/project/

su - dev1
touch /opt/project/app.conf
chmod 640 /opt/project/app.conf
exit

ls -l /opt/project/app.conf
ls -ld /opt/project
```

Install ACL tools and set access for `qa1`:

```bash
apt install -y acl
setfacl -m u:qa1:r-- /opt/project/app.conf
getfacl /opt/project/app.conf
```

At first, `qa1` received `Permission denied` when trying to access the file. I then added execute permission on the parent directory for `qa1`:

```bash
setfacl -m u:qa1:--x /opt/project
getfacl /opt/project
```

I switched to `qa1` again and tested the file:

```bash
su - qa1
cat /opt/project/app.conf
```

### Results

User and group information:

```text
uid=1001(dev1) gid=1002(dev1) groups=1002(dev1),1001(devteam)
uid=1002(qa1) gid=1003(qa1) groups=1003(qa1)
```

Directory and file permissions:

```text
drwxrwx--- 2 root devteam ... /opt/project
-rw-r----- 1 dev1 dev1 0 ... /opt/project/app.conf
```

File ACL:

```text
# file: opt/project/app.conf
# owner: dev1
# group: dev1
user::rw-
user:qa1:r--
group::r--
mask::r--
other::---
```

Directory ACL entry added for `qa1`:

```text
user:qa1:--x
```

After adding execute permission to the directory, `qa1` could access the file. The file was empty, so `cat` did not display any content. Listing the directory itself was still denied, because `qa1` did not have directory read permission.

### What I understood

- `useradd -m` creates a user with a home directory.
- `usermod -aG` adds a user to an additional group.
- `chown` changes the owner and group of a file or directory.
- `chmod 770` gives the owner and group full access and gives others no access.
- `chmod 640` gives the owner read/write access, the group read access and others no access.
- For directories, execute (`x`) permission is needed to traverse the directory and access a known filename.
- `setfacl` adds or changes ACL permissions. `getfacl` displays them.

---

# Task 4 – Team folder and sudo permissions

### Objective

Create an `ops` group and users, make a shared folder where new files inherit the group, and allow group members to run only the required service restart command through sudo.

### Commands used

```bash
groupadd ops
useradd -m carol
useradd -m dan

usermod -aG ops carol
usermod -aG ops dan

id carol
id dan

mkdir -p /srv/ops-share
chown root:ops /srv/ops-share
chmod 2770 /srv/ops-share
ls -ld /srv/ops-share
```

Create a file as `carol` and check its group:

```bash
su - carol
touch /srv/ops-share/carol-file.txt
exit

ls -l /srv/ops-share/
```

Sudo rule created using:

```bash
visudo -f /etc/sudoers.d/ops
```

Rule added:

```sudoers
%ops ALL=(root) /usr/bin/systemctl restart cron
```

Validate the rule:

```bash
visudo -cf /etc/sudoers.d/ops
sudo -l -U dan
```

### Results

Group membership:

```text
uid=1001(carol) gid=1002(carol) groups=1002(carol),1001(ops)
uid=1002(dan) gid=1003(dan) groups=1003(dan),1001(ops)
```

Shared directory permissions:

```text
drwxrws--- 2 root ops ... /srv/ops-share
```

The file created by `carol` showed the `ops` group:

```text
-rw-rw-r-- 1 carol ops 0 ... carol-file.txt
```

Sudo validation:

```text
/etc/sudoers.d/ops: parsed OK
```

`sudo -l -U dan` showed the allowed command:

```text
(root) /usr/bin/systemctl restart cron
```

### What I understood

- The leading `2` in `chmod 2770` enables the setgid bit on the directory.
- With setgid on a shared directory, newly created files inherit the directory's group.
- `%ops` in the sudoers rule applies the permission to the group, not to one individual user.
- `visudo` checks sudoers syntax before saving the rule.
- The rule allows the specified command only; it does not give the group full root access.

---

# Task 5 – Process and service management

### Objective

Install and manage Nginx, check its processes and listening port, view service logs, and practise stopping a background process.

### Commands used

```bash
apt install -y nginx

systemctl is-active nginx
systemctl is-enabled nginx

pgrep -a nginx
ps -C nginx -o pid,ppid,user,comm
```

Stop Nginx and check port 80:

```bash
systemctl stop nginx
systemctl is-active nginx
ss -ltnp '( sport = :80 )'
```

Start it again and check:

```bash
systemctl start nginx
systemctl is-active nginx
ss -ltnp '( sport = :80 )'
```

View logs:

```bash
journalctl -u nginx --since "10 minutes ago"
```

Test a background process:

```bash
sleep 1000 &
pgrep -x sleep
kill -TERM 49811
pgrep -x sleep
```

### Results

Nginx was:

```text
active
enabled
```

Process details recorded during the lab:

```text
PID   PPID USER      COMMAND
2476     1 root      nginx
2478  2476 www-data  nginx
```

After stopping Nginx, the port 80 check returned no listening process. After starting it again, Nginx was listening on port 80.

The background `sleep` process was started with PID `49811`. After `kill -TERM 49811`, `pgrep -x sleep` returned no process.

### What I understood

- `systemctl` is used to manage system services.
- `is-active` checks whether a service is running.
- `is-enabled` checks whether it is configured to start at boot.
- Nginx normally has a master process and worker process(es).
- `pgrep` searches for processes by name.
- `ss -ltnp` displays listening TCP ports and the related processes.
- `journalctl` displays systemd service logs.
- `kill -TERM` asks a process to stop gracefully.

---

# Task 6 – Custom systemd service

### Objective

Create a small web page and run a Python HTTP server through a custom systemd service. Configure systemd to restart the service if its process stops unexpectedly.

### Create the web page

```bash
mkdir -p /srv/web
echo "Hello from Systemd" > /srv/web/index.html
cat /srv/web/index.html

chown -R root:root /srv/web
chmod 755 /srv/web
chmod 644 /srv/web/index.html

ls -ld /srv/web
ls -l /srv/web/index.html
```

### Service file

Created `/etc/systemd/system/web-demo.service`:

```ini
[Unit]
Description=Web Demo HTTP Server
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/srv/web
ExecStart=/usr/bin/python3 -m http.server 8080
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### Start and test the service

```bash
systemctl daemon-reload
systemctl enable --now web-demo
systemctl status web-demo
curl http://localhost:8080
```

Check the process:

```bash
systemctl show -p MainPID web-demo
ps -p 6493 -o pid,ppid,user,comm
```

Test automatic restart by killing the main process:

```bash
kill -9 6493
systemctl show -p MainPID web-demo
systemctl status web-demo
```

### Results

The page returned:

```text
Hello from Systemd
```

The service was active and running under `www-data`. The original main PID was `6493`. After the process was killed, systemd restarted it with a new PID:

```text
MainPID=16464
```

The service status showed:

```text
Active: active (running)
Restart counter is at 1
```

### What I understood

- A systemd unit file defines how a service should run.
- `User=www-data` runs the service as the `www-data` user instead of root.
- `WorkingDirectory` sets the working directory for the service.
- `ExecStart` defines the command to start.
- `Restart=always` tells systemd to restart the service when it exits.
- `RestartSec=3` waits three seconds before restarting.
- `daemon-reload` makes systemd read the updated unit file.
- `enable --now` enables the service at boot and starts it immediately.

---

# Task 7 – Package management

### Objective

Update package information, check package versions, install a package, inspect its files, identify which package owns a command and remove the test package.

### Commands used

```bash
apt update
apt-cache policy tree
apt install -y tree

tree --version
dpkg -L tree
dpkg -L tree | grep '/usr/bin'

dpkg -S /usr/bin/ls
dpkg -l | grep '^ii' | wc -l

apt remove -y tree
dpkg -l tree
apt purge -y tree
```

### Results

Package candidate shown by `apt-cache policy tree`:

```text
Installed: (none)
Candidate: 2.1.1-2ubuntu3.24.04.2
```

Installed version:

```text
tree v2.1.1
```

The binary was installed at:

```text
/usr/bin/tree
```

Package ownership check:

```text
coreutils: /usr/bin/ls
```

Installed package count shown by the command:

```text
815
```

After `apt remove -y tree`, `dpkg -l tree` reported that no package was found.

I then ran `apt purge -y tree`. It reported that the package was not installed, because it had already been removed.

### What I understood

- `apt update` refreshes the package lists. It does not upgrade installed packages by itself.
- `apt-cache policy` shows available and installed package versions.
- `apt install` installs a package.
- `dpkg -L` lists files installed by a package.
- `dpkg -S` identifies which installed package owns a file.
- `apt remove` removes the package.
- `apt purge` removes a package and its package-managed configuration files. In this lab, the package had already been removed before I ran purge.

---

# Task 8 – Disk management and LVM

### Objective

Check disks and filesystems, identify large directories, create a loop-backed disk, configure LVM, format and mount a logical volume, and extend it while mounted.

### Inspect disks and usage

```bash
lsblk -f
df -hT
du -h --max-depth=1 /var | sort -rh | head -n 5
lvm version
```

Top `/var` entries recorded:

```text
888M    /var
607M    /var/cache
257M    /var/lib
23M     /var/log
1.1M    /var/backups
```

### Create a loop device

```bash
dd if=/dev/zero of=/root/loopdisk.img bs=1M count=1024
losetup -fP --show /root/loopdisk.img
losetup -a
lsblk
```

The loop device created was:

```text
/dev/loop0
```

The image file size was 1 GiB.

### Create the LVM structure

```bash
pvcreate /dev/loop0
vgcreate vgdemo /dev/loop0
vgs

lvcreate -L 300M -n lvdata vgdemo
lvs
```

The volume group was `vgdemo` and the logical volume was `lvdata`.

### Format and mount

```bash
mkfs.ext4 /dev/vgdemo/lvdata
blkid /dev/vgdemo/lvdata

mkdir -p /mnt/data
mount /dev/vgdemo/lvdata /mnt/data

df -hT /mnt/data
cp /etc/hosts /mnt/data/
ls -l /mnt/data/
cat /mnt/data/hosts
```

The filesystem type was `ext4`. The mounted logical volume was available at `/mnt/data`.

### Extend the logical volume

```bash
lvs
lvextend -L +200M /dev/vgdemo/lvdata
resize2fs /dev/vgdemo/lvdata

df -hT /mnt/data
lvs
```

### Results

The logical volume was extended from 300 MiB to 500 MiB:

```text
Size of logical volume vgdemo/lvdata changed from 300.00 MiB to 500.00 MiB.
Logical volume vgdemo/lvdata successfully resized.
```

After `resize2fs`, the mounted filesystem showed:

```text
Filesystem               Type  Size  Used Avail Use% Mounted on
/dev/mapper/vgdemo-lvdata ext4  459M   28K  435M   1% /mnt/data
```

The `lvs` output showed:

```text
lvdata vgdemo ... 500.00m
```

### What I understood

- `lsblk` displays block devices and their relationships.
- `df -hT` shows mounted filesystem space and type.
- A loop device allows a file to be used like a block device.
- LVM uses a Physical Volume (PV), Volume Group (VG) and Logical Volume (LV).
- `mkfs.ext4` creates an ext4 filesystem.
- `mount` makes the filesystem available at a directory.
- `lvextend` increases the logical volume size.
- `resize2fs` expands the ext4 filesystem to use the additional space.
- The lab used a loop device, so the main system disk was not resized.

---

# Task 9 – Networking and SSH

### Objective

Check network interfaces, default gateway, DNS configuration and listening TCP ports. Test the local web server, resolve a public domain and create an SSH key pair.

### Network information

```bash
ip -br addr
ip route show default
resolvectl status
```

Recorded network details:

| Item | Result |
|---|---|
| Interface | `enp1s0` |
| IPv4 address | `172.30.1.2/24` |
| Default gateway | `172.30.1.1` |
| DNS servers | `8.8.8.8`, `1.1.1.1` |

### Check listening TCP ports

```bash
ss -ltnp
```

The output showed services including SSH on port `22`, Nginx on port `80`, the system DNS resolver on port `53` and Killercoda-related services.

### Test Nginx

```bash
curl -sSI http://localhost
```

Result:

```text
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Content-Type: text/html
Content-Length: 63
```

### DNS lookup

```bash
getent ahosts ubuntu.com
dig ubuntu.com
```

`getent` returned IPv4 and IPv6 addresses. `dig` returned three IPv4 A records:

```text
185.125.190.29
185.125.190.20
185.125.190.21
```

The DNS server used by `dig` was `8.8.8.8`.

### Generate an SSH key

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/lab_key -N ""
ssh-keygen -lf ~/.ssh/lab_key.pub
ls -ld ~/.ssh
```

The key fingerprint was:

```text
SHA256:M3/X4R1zNX7rHVO7T/FhARVqAiq33rUfSrIetvJqevQ root@ubuntu (ED25519)
```

The `.ssh` directory permissions were:

```text
drwx------ 2 root root ... /root/.ssh
```

### What I understood

- `ip -br addr` gives a short view of network interfaces and IP addresses.
- `ip route` shows routing information, including the default gateway.
- `resolvectl status` displays DNS settings.
- `ss -ltnp` shows listening TCP sockets and process details.
- `getent` uses the system name-resolution configuration.
- `dig` sends a DNS query directly to a DNS server.
- `ssh-keygen` creates a public/private SSH key pair.
- The private key must be kept secret. The public key can be installed on a server for key-based login.

---

# Task 10 – DNS, HTTP server and nftables

### Objective

Test local hostname resolution using `/etc/hosts`, run a Python HTTP server, and compare the firewall actions `drop` and `reject`.

## Part 1 – Test `/etc/hosts`

Add a temporary entry:

```bash
echo '10.9.9.9 example.com' >> /etc/hosts
getent hosts example.com
dig example.com
getent ahostsv4 example.com
grep example.com /etc/hosts
```

The IPv4 system lookup returned:

```text
10.9.9.9 STREAM example.com
10.9.9.9 DGRAM
10.9.9.9 RAW
```

`dig example.com` continued to return public DNS addresses because `dig` queries DNS directly and does not use `/etc/hosts`.

Remove the temporary entry:

```bash
sed -i '\|^10\.9\.9\.9[[:space:]]\+example\.com$|d' /etc/hosts
getent ahostsv4 example.com
grep example.com /etc/hosts
```

After removal, `getent ahostsv4` returned public DNS addresses and `grep` returned no matching entry.

### What I understood

The system resolver can use `/etc/hosts` before querying DNS, depending on the system configuration. `dig` queries DNS directly, so it does not read the local hosts file.

## Part 2 – Run a Python HTTP server

Create a test page and start the server on port `9090`:

```bash
mkdir -p /tmp/webtest
echo "Hello from Python HTTP Server" > /tmp/webtest/index.html
cd /tmp/webtest

python3 -m http.server 9090 >/tmp/http9090.log 2>&1 &
echo $! > /tmp/http9090.pid

cat /tmp/http9090.pid
curl -I http://127.0.0.1:9090
```

The server PID was `8280`. The response was:

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.12.3
Content-type: text/html
Content-Length: 30
```

### What I understood

The Python command starts a basic HTTP server from the current directory. The `&` runs it in the background, and `$!` gives the PID of the last background process.

## Part 3 – Test nftables DROP

Check nftables:

```bash
nft --version
```

Version recorded:

```text
nftables v1.0.9 (Old Doc Yak #3)
```

Create a separate table and input chain for this lab:

```bash
nft add table inet lab10
nft 'add chain inet lab10 input { type filter hook input priority 0; policy accept; }'
nft add rule inet lab10 input tcp dport 9090 drop
nft -a list chain inet lab10 input
```

The rule was:

```text
tcp dport 9090 drop # handle 2
```

Test the connection:

```bash
curl --max-time 5 -v http://127.0.0.1:9090
```

Result:

```text
Connection timed out after 5002 milliseconds
curl: (28) Connection timed out after 5002 milliseconds
```

### What I understood

`DROP` silently discards matching packets. The client does not receive a rejection response, so it waits until the connection timeout.

## Part 4 – Replace DROP with REJECT

Delete the DROP rule and add a REJECT rule:

```bash
nft delete rule inet lab10 input handle 2
nft add rule inet lab10 input tcp dport 9090 reject with tcp reset
nft -a list chain inet lab10 input
```

The new rule was:

```text
tcp dport 9090 reject with tcp reset # handle 4
```

Test again:

```bash
curl --max-time 5 -v http://127.0.0.1:9090
```

Result:

```text
connect to 127.0.0.1 port 9090 ... failed: Connection refused
curl: (7) Failed to connect to 127.0.0.1 port 9090
```

### What I understood

- `REJECT` actively rejects matching traffic.
- `with tcp reset` sends a TCP reset, so the client gets an immediate connection failure.
- `DROP` and `REJECT` are different firewall actions: one silently discards packets and the other sends a rejection.

## Part 5 – Cleanup

Remove the lab firewall table and stop the Python server:

```bash
nft delete table inet lab10
kill -TERM "$(cat /tmp/http9090.pid)"

nft list tables
ss -ltnp '( sport = :9090 )'
```

The `lab10` table was no longer listed. Port `9090` had no listening process.

The remaining tables shown were:

```text
table ip nat
table ip filter
table ip6 nat
table ip6 filter
```

I left those existing tables untouched.

---

# Useful commands – Quick revision

| Command | Use |
|---|---|
| `uname -r` | Show kernel release |
| `uptime` | Show system uptime and load |
| `ps` | Display process information |
| `systemd-analyze` | Check boot time |
| `ls -li` | Show file inode numbers |
| `df -hT` | Check filesystem space and type |
| `df -i` | Check inode usage |
| `id username` | Check user and group membership |
| `chmod` | Change permissions |
| `chown` | Change file owner and group |
| `setfacl` / `getfacl` | Manage and view ACLs |
| `visudo` | Safely edit and validate sudoers rules |
| `systemctl` | Manage systemd services |
| `pgrep` | Find processes by name |
| `ss -ltnp` | Show listening TCP ports and processes |
| `journalctl` | View systemd logs |
| `apt` | Manage Ubuntu packages |
| `dpkg -L` | List files installed by a package |
| `lsblk -f` | View block devices and filesystems |
| `pvs`, `vgs`, `lvs` | View LVM details |
| `pvcreate`, `vgcreate`, `lvcreate` | Create LVM components |
| `lvextend` | Extend a logical volume |
| `resize2fs` | Resize an ext filesystem |
| `ip -br addr` | Show network interfaces and addresses |
| `ip route` | View routing table |
| `resolvectl status` | View DNS configuration |
| `getent` | Resolve names using system configuration |
| `dig` | Query DNS |
| `ssh-keygen` | Generate SSH keys |
| `nft` | Manage nftables firewall rules |

---

## Final note

This assessment gave me hands-on practice with Linux system information, files and links, users and permissions, sudo access, processes, services, packages, storage, networking, SSH, DNS and firewall rules.

I want to keep practising these commands so I can become more comfortable with Linux administration and troubleshooting.

**Assessment status: All 10 tasks completed.**
