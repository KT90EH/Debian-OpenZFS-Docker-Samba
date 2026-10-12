# Debian-OpenZFS-Docker-Samba
Public

**My Project Visna Notes**

AI-assisted using Grok and Claude.

**Server Name:** `Visna`

**Goal:** Replace my TrueNAS Scale CE server with a custom-made Debian home server.

**Hardware:** AMD Ryzen 5 5600X CPU, ASRock Rack X470D4U motherboard, TP-Link 2.5GB PCIe Network Card, HDD's are CMR, etc...

---

## Table of Contents

- [Intro](#Intro)
- [Install Debian 13 Trixie (ext4)](#install-debian-13-trixie-ext4)
- [Debian 13 Manual Partitioning — SATA SSD / ext4](#debian-13-manual-partitioning--sata-ssd--ext4)
- [Post Install Server Setup](#post-install-server-setup)
- [Make perry a sudo user](#make-perry-a-sudo-user)
- [Networking](#networking)
- [Install firewalld](#install-firewalld)
- [Migrate to deb822 Format in Debian 13 Trixie and Install Backports](#migrate-to-deb822-format-in-debian-13-trixie-and-install-backports)
- [Install Cockpit](#install-cockpit)
- [Install Cockpit Files](#install-cockpit-files)
- [Install Misc. Tools](#install-misc-tools)
- [Limit Journal Size](#limit-journal-size)
- [SSH Hardening](#ssh-hardening)
- [ZFS on Debian — Setup Notes](#zfs-on-debian--setup-notes)
- [Samba on ZFS for Docker — Setup Runbook](#samba-on-zfs-for-docker--setup-runbook)
- [Install Docker Community Edition](#install-docker-community-edition)
- [Install Tailscale](#install-tailscale)
- [DNS Setup: Omega, Visna and AP-5](#dns-setup-omega-visna-and-ap-5)
- [Install Dockhand Free Edition](#install-dockhand-free-edition)
- [Install Netdata Community Edition](#install-netdata-community-edition)
- [AppArmor Basics for a Debian Server](#AppArmor-Basics-for-a-Debian-Server)
- [Final Notes](#Final-Notes)

---

## Intro

- Debian 13 Trixie on a single boot disk with two ZFS pools
- SSH, SSH Keys, Firewalld, Cockpit, Cockpit Files, ClamAV, Tailscale, Samba, Docker, Dockhand, Netdata, etc.
- Local home network
- Bare metal server

---

## Install Debian 13 Trixie (ext4)

Use the KVM console on the X470D4U motherboard's IPMI. ⚠️

- Install OS on a single SATA SSD disk
- ZFS pools come later: `pool1` (two NVMe SSDs) and `pool2` (two HDDs)
- Do **not** use LVM — use the whole disk

### Recommended Architecture

| Component   | Technology              |
|-------------|-------------------------|
| Root        | ext4 (no LVM)           |
| Data Pools  | ZFS (separate)          |
| Monitoring  | Netdata                 |
| Apps        | Docker / Containers     |

---

## Debian 13 Manual Partitioning — SATA SSD / ext4

**Target:** single SATA SSD boot drive, UEFI boot, ext4 root. **No LVM** — the entire disk is partitioned directly (no volume group / logical volumes).

### Partition Layout

| # | Purpose                | Size            | Filesystem | Mount Point  |
|---|------------------------|-----------------|------------|--------------|
| 1 | EFI System Partition   | 1 GB            | FAT32      | `/boot/efi`  |
| 2 | Swap                   | 4.0 GB          | swap       | —            |
| 3 | Root                   | remaining space | ext4       | `/`          |

### Steps in the Debian Installer

1. At **Partition disks**, select **Manual** (not "Guided - use entire disk" or any LVM-guided option — this avoids LVM entirely and gives direct control over partitions on the whole disk).
2. Select the target disk (e.g. `sda`) → confirm **Create new empty partition table on this device?** → **Yes** (creates GPT).
3. Select the **FREE SPACE** line → **Create a new partition**.

#### Partition 1 — EFI System Partition

- Size: `1G`
- Type: Primary
- Location: Beginning
- Use as: **EFI System Partition**
- Done setting up the partition

#### Partition 2 — swap

- Size: `4G`
- Type: Primary
- Use as: **swap area**
- Done setting up the partition

#### Partition 3 — root

- Size: accept remaining space (default)
- Type: Primary
- Use as: **Ext4 journaling file system**
- Mount point: `/`
- Done setting up the partition

4. Select **Finish partitioning and write changes to disk**.
5. Confirm the summary matches:
   - `#1 EFI System Partition — 1.0 GB`
   - `#2 swap — 4.0 GB`
   - `#3 / ext4 — remaining space`
6. Confirm **Yes** to write changes to disk.

### Post-Install Verification Commands

```bash
lsblk -f
sudo fdisk -l /dev/sda
df -hT
```

---

## Post Install Server Setup

Reference guides:

- https://computingforgeeks.com/things-to-do-after-installing-debian-13-trixie/
- https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu
- https://computingforgeeks.com/harden-ubuntu-2604-server/
- https://computingforgeeks.com/ubuntu-2604-initial-server-setup/

### Update package lists

```bash
apt update
```

### Core utilities

```bash
apt install -y \
  curl wget git vim nano htop ncdu tree unzip zip rsync \
  tmux net-tools dnsutils sudo gnupg2 ca-certificates hdparm
```

### Monitoring / diagnostics / logging tools

```bash
apt install -y iotop nload iftop lm-sensors smartmontools logrotate
```

For lm-sensors, run `sudo sensors-detect` after install. ⚠️

### Set hostname and timezone

```bash
hostnamectl set-hostname yourhostname
timedatectl set-timezone America/Los_Angeles
```

### Time sync

Debian 13 uses systemd-timesyncd by default — verify it's running:

```bash
timedatectl status
```

### Disable unused services (Deferred)

```bash
systemctl list-unit-files --state=enabled
```
> ⚠️ **Hold off** Until you learn more about this!

---

## Make perry a sudo user

Run as root:

```bash
apt install sudo
adduser perry
usermod -aG sudo perry
```

---

## Networking

- Let DHCP assign a local IP address. ⚠️
- Reserve the IP address in your home router.

---

## Install firewalld

Login with SSH (use PowerShell or terminal):

```bash
ssh perry@10.0.0.xx

sudo apt update
sudo apt install firewalld
sudo systemctl enable --now firewalld
sudo systemctl start firewalld
sudo systemctl status firewalld
sudo firewall-cmd --reload
```

---

## Migrate to deb822 Format in Debian 13 Trixie and Install Backports

Reference guides:

- [Migrate to deb822 Format – OSTechNix](https://ostechnix.com/migrate-to-deb822-format-debian-13-trixie/)
- [Debian Backports Instructions](https://backports.debian.org/Instructions/)

### Step 1 – Clean the classic sources.list

```bash
sudo nano /etc/apt/sources.list
```

Comment out (or remove) any `deb cdrom` lines:

```text
# deb cdrom:[Debian GNU/Linux ...]
```

Save and exit.

### Step 2 – Migrate official sources to deb822 format

```bash
sudo apt modernize-sources
```

Verify the new file:

```bash
cat /etc/apt/sources.list.d/debian.sources
```

**Expected output:**

```text
# Modernized from /etc/apt/sources.list
Types: deb deb-src
URIs: http://deb.debian.org/debian/
Suites: trixie
Components: main non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

# Modernized from /etc/apt/sources.list
Types: deb deb-src
URIs: http://security.debian.org/debian-security/
Suites: trixie-security
Components: main non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

# Modernized from /etc/apt/sources.list
Types: deb deb-src
URIs: http://deb.debian.org/debian/
Suites: trixie-updates
Components: main non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

### Step 3 – Add the Trixie backports repository

Create a new deb822 sources file:

```bash
sudo nano /etc/apt/sources.list.d/debian-backports.sources
```

Paste the following content:

```text
Types: deb deb-src
URIs: http://deb.debian.org/debian
Suites: trixie-backports
Components: main contrib non-free-firmware
Enabled: yes
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

Save and exit.

### Step 4 – Update the package index

```bash
sudo apt-get update
```

### Step 5 – Modernize third-party sources (Docker & Tailscale)

After installing Docker and/or Tailscale, their classic `.list` files also need conversion:

```bash
sudo apt modernize-sources
```

#### Docker

```bash
cat /etc/apt/sources.list.d/docker.sources
```

**Expected output:**

```text
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: trixie
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
```

#### Tailscale

```bash
cat /etc/apt/sources.list.d/tailscale.sources
```

**Expected output:**

```text
Types: deb
URIs: https://pkgs.tailscale.com/stable/debian/
Suites: trixie
Components: main
Signed-By: /usr/share/keyrings/tailscale-archive-keyring.gpg
```

### Step 6 – Netdata (already modernized)

Netdata creates a deb822 file automatically. You can verify it with:

```bash
cat /etc/apt/sources.list.d/netdata.sources
```

**Expected output:**

```text
X-Repolib-Name: Netdata stable repository
Types: deb
URIs: http://repository.netdata.cloud/repos/stable/debian/
Suites: trixie/
Signed-By: /usr/share/keyrings/netdata-archive-keyring.gpg
By-Hash: Yes
Enabled: Yes

X-Repolib-Name: Netdata repository configuration repository
Types: deb
URIs: http://repository.netdata.cloud/repos/repoconfig/debian/
Suites: trixie/
Signed-By: /usr/share/keyrings/netdata-archive-keyring.gpg
By-Hash: Yes
Enabled: Yes
```

### Step 7 – Final update

```bash
sudo apt-get update
```

**Done.** Your system now uses the modern deb822 sources format and has access to the Debian Trixie backports repository.

---

## Install Cockpit

Use the newest version of Cockpit from backports:

```bash
sudo apt install -t trixie-backports cockpit

sudo firewall-cmd --add-service=cockpit --permanent
sudo firewall-cmd --reload

sudo systemctl enable --now cockpit.socket
sudo systemctl status cockpit
```

Notes:

- Use the default public zone for all your ports.
- Docker will create its own zone. Let Docker be Docker. ⚠️
- Tailscale does not need a zone. Let Tailscale be Tailscale. ⚠️

---

## Install Cockpit Files

- https://github.com/cockpit-project/cockpit-files

```bash
sudo apt install gettext nodejs npm make
git clone https://github.com/cockpit-project/cockpit-files.git
cd cockpit-files
sudo make install
```

---

## Install Misc. Tools

### smartmontools

1. Install smartmontools (Netdata needs this to read S.M.A.R.T. data):

   ```bash
   sudo apt install smartmontools
   ```

2. Install nvme-cli (needed for Netdata SMART monitoring of NVMe drives):

   ```bash
   sudo apt install nvme-cli
   ```

3. Enable S.M.A.R.T. on your drives:

   ```bash
   sudo smartctl -s on /dev/sda
   sudo smartctl -s on /dev/nvme0   # if you have NVMe drives
   ```

4. Verify it's working:

   ```bash
   sudo smartctl -a /dev/sda
   ```

### ipmitool

1. Install:

   ```bash
   sudo apt update
   sudo apt install ipmitool
   sudo modprobe ipmi_si
   sudo modprobe ipmi_devintf
   echo -e "ipmi_si\nipmi_devintf" | sudo tee -a /etc/modules
   ```

2. Check/confirm the IP of the BMC network:

   ```bash
   sudo ipmitool lan print 1
   ```

---

## Limit Journal Size

Reference: https://learn.netdata.cloud/docs/logs-management/log-storage-and-retention

```bash
sudo nano /etc/systemd/journald.conf
```

Uncomment and update under the `[Journal]` section:

```ini
[Journal]
Storage=persistent
Compress=yes
SystemMaxUse=2G
SystemKeepFree=1G
SystemMaxFileSize=128M
SystemMaxFiles=100
MaxRetentionSec=1month
MaxFileSec=1week
```

```bash
sudo systemctl restart systemd-journald
```

Check current usage:

```bash
sudo journalctl --disk-usage
```

Force cleanup right now (if it's already large):

```bash
sudo journalctl --vacuum-size=500M
# or
sudo journalctl --vacuum-time=1month
```

Verify it applied:

```bash
systemctl status systemd-journald
journalctl --disk-usage
```

---

## SSH Hardening

### Sources

- [Linuxize: SSH Hardening Best Practices](https://linuxize.com/post/ssh-hardening-best-practices/)
- [computingforgeeks: Harden Ubuntu 24.04 Server](https://computingforgeeks.com/harden-ubuntu-2604-server/)
- [computingforgeeks: Things to do after installing Debian 13 Trixie](https://computingforgeeks.com/things-to-do-after-installing-debian-13-trixie/)
- [YouTube walkthrough](https://www.youtube.com/watch?v=MWqfc_fegVg)
- [YouTube walkthrough](https://www.youtube.com/watch?v=GxRu35fy-oY)

### On the client (your desktop/laptop)

**1. Generate a dedicated key**

```bash
ssh-keygen -t ed25519 -C "perry@kde-desktop" -f ~/.ssh/visna_key
```

Set a passphrase when prompted.

**2. Add an SSH config entry**

```bash
sudo nano ~/.ssh/config
```

```
# Visna — SSH over LAN
Host visna-lan
    HostName 10.0.0.xx
    User perry
    IdentityFile ~/.ssh/visna_key
    IdentitiesOnly yes
```

**3. Copy the public key to the server**

```bash
ssh-copy-id -i ~/.ssh/visna_key.pub perry@10.0.0.xx
```

This appends the key to `~/.ssh/authorized_keys` on the server.

**4. Connect using the key**

```bash
ssh visna-lan
```

Log in successfully at least once and confirm you're not prompted for a password — this is the step people skip and then lock themselves out.

### On the server (Visna)

**5. Add a hardening drop-in config**

```bash
sudo tee /etc/ssh/sshd_config.d/90-hardening.conf > /dev/null <<EOF
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers perry
EOF
```

`AllowUsers perry` is included so a future added account doesn't get SSH access by default. Drop this line if you don't want that restriction.

**6. Test the config syntax, then reload (not restart)**

```bash
sudo sshd -t && sudo systemctl reload ssh
```

`reload` keeps your current session alive while applying the new config.

**7. Verify the settings took effect**

```bash
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|maxauthtries|allowusers'
```

**8. Before closing your session** — open a *second* terminal and test a fresh login (`ssh visna-lan`) while the first session is still active. If it fails, you still have the original session open to fix or remove `90-hardening.conf`.

---

## ZFS on Debian — Setup Notes

**References**

- [OpenZFS Docs](https://openzfs.github.io/openzfs-docs/)
- [Configuring ZFS on Debian (itstorage.net)](https://itstorage.net/index.php/ldce/lwstgservm/674-configur-zfs-debian)

### 1. Installing ZFS on Debian

> Source: [OpenZFS — Getting Started / Debian](https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/index.html)

Pin ZFS to the Trixie backports repo:

```bash
sudo nano /etc/apt/preferences.d/90_zfs
```

Add:

```
Package: src:zfs-linux
Pin: release n=trixie-backports
Pin-Priority: 990
```

Install headers first so DKMS has something to build against. The metapackage `linux-headers-amd64` keeps headers tracking future kernels, and the second line covers the case where you're running a kernel older than the newest one in the repo:

```bash
apt update
apt install linux-headers-amd64 linux-headers-$(uname -r)
apt install zfs-dkms zfsutils-linux
```

Load the module and confirm:

```bash
modprobe zfs
zpool version
lsmod | grep zfs
```

`zpool version` should show both the userland and `zfs-kmod` versions.

#### After kernel upgrades

Before rebooting into a new kernel, verify the module built for it:

```bash
dkms status
```

You should see `zfs/<version>, <newest kernel>: installed`.

If the newest kernel is missing, install its headers and DKMS will build automatically:

```bash
apt install linux-headers-<kernel-version>
```

If you've already rebooted and ZFS won't load (`modprobe: FATAL: Module zfs not found`), the module wasn't built for the running kernel. Install the matching headers as above, or boot the previous kernel from GRUB's Advanced options and pool access will work again in the meantime.

To see why a build failed:

```bash
tail -50 /var/lib/dkms/zfs/*/build/make.log
```

### 2. Create ZFS Pools

List installed drives by stable ID (use these paths, not `/dev/sdX`):

```bash
ls -l /dev/disk/by-id/
```

#### SSD pool (`fastpool`)

```bash
sudo zpool create -o ashift=12 -o autotrim=on -o autoexpand=on \
  -O compression=lz4 \
  -O atime=off \
  -O xattr=sa \
  -O acltype=posixacl \
  -O dnodesize=auto \
  -O recordsize=128k \
  fastpool mirror \
  /dev/disk/by-id/nvme-WD_Red_SN700_500GB_2249#REDACTED \
  /dev/disk/by-id/nvme-WD_Red_SN700_500GB_2326#REDACTED
```

#### HDD pool (`bulkpool`)

```bash
sudo zpool create -o ashift=12 -o autoexpand=on \
  -O compression=lz4 \
  -O atime=off \
  -O xattr=sa \
  -O acltype=posixacl \
  -O dnodesize=auto \
  -O recordsize=128k \
  bulkpool mirror \
  /dev/disk/by-id/ata-WDC_WD80EFPX-68C4ZN0_WD-#REDACTED \
  /dev/disk/by-id/ata-WDC_WD80EFPX-68C4ZN0_WD-#REDACTED
```

> **Note:** `autotrim=on` is set on the SSD pool only — it's not relevant for spinning HDDs.

### 3. Power Management (Deferred)

> ⚠️ **Hold off** — disabling aggressive spin-down for the media HDDs may not be needed for WD Red Plus drives. Revisit if spin-down/spin-up cycling becomes a problem.

### 4. Create ZFS Datasets

**Inheritance reminder:** properties set on the pool/parent are inherited by child datasets, so you only need to override what differs:

- `sync=standard` is already the default — no need to set explicitly.
- `dedup=off` is already the default — no need to set explicitly.

```bash
### SSD Pool - Appdata
sudo zfs create fastpool/appdata
sudo zfs create fastpool/appdata/configs
sudo zfs create -o recordsize=16k fastpool/appdata/databases
sudo zfs create -o recordsize=64k fastpool/appdata/cache

### SSD Pool - Documents
sudo zfs create fastpool/documents
sudo zfs create fastpool/documents/perry

### HDD Pool - MediaVault
sudo zfs create -o recordsize=1M bulkpool/mediavault
```

| Dataset                        | Recordsize       | Rationale                      |
|--------------------------------|------------------|--------------------------------|
| `fastpool/appdata`             | 128k (inherited) | General app config             |
| `fastpool/appdata/configs`     | 128k (inherited) | Small config files             |
| `fastpool/appdata/databases`   | 16k              | Matches typical DB page size   |
| `fastpool/appdata/cache`       | 64k              | Mixed-size cache objects       |
| `fastpool/documents`           | 128k (inherited) | General documents              |
| `bulkpool/mediavault`          | 1M               | Large sequential media files   |

### 5. Verify Setup

Check dataset properties:

```bash
zfs get recordsize,compression,atime,xattr,acltype,dnodesize -r fastpool/appdata
zfs get recordsize,compression,atime,xattr,acltype,dnodesize -r fastpool/documents
zfs get recordsize,compression,atime,xattr,acltype,dnodesize -r bulkpool
```

Check pool/dataset overview:

```bash
zpool status
zfs list -o name,recordsize,compression,atime,xattr,acltype,dnodesize,mountpoint
zpool get autotrim,autoexpand
```

---

## Samba on ZFS for Docker — Setup Runbook

**Host:** `Visna` (Debian 13 + OpenZFS + Docker + Samba)

**Pools:**

- `fastpool` — SSD, hosts app config / docs
- `bulkpool` — HDD, hosts media

### 1. Install Samba

```bash
sudo apt install samba samba-common-bin acl attr
```

### 2. Create Users and Groups

```bash
# Regular login user
sudo adduser perry
sudo smbpasswd -a perry
sudo smbpasswd -e perry

# File-share-only user (no shell login)
sudo adduser --no-create-home --shell /usr/sbin/nologin <username>
sudo smbpasswd -a <username>
sudo smbpasswd -e <username>

# Remove a no-login user (Samba first, then Linux)
sudo smbpasswd -x <username>
sudo deluser <username>

# Find no-login accounts first if needed:
awk -F: '$7 ~ /(nologin|false)/ {print $1, $7}' /etc/passwd
```

Groups (one per share):

```bash
sudo groupadd -g 2001 grp-samba-appdata
sudo groupadd -g 2002 grp-samba-perry
sudo groupadd -g 2003 grp-samba-mediavault

sudo usermod -aG grp-samba-appdata perry
sudo usermod -aG grp-samba-perry perry
sudo usermod -aG grp-samba-mediavault perry
```

### 3. Make Sure ZFS Isn't Also Managing Shares

```bash
zfs get sharesmb
sudo zfs set sharesmb=off fastpool/appdata
sudo zfs set sharesmb=off fastpool/documents
sudo zfs set sharesmb=off bulkpool/mediavault
```

### 4. Global Samba Settings

```bash
sudo nano /etc/samba/smb.conf
```

```ini
[global]
workgroup = Vault-Tec
server string = %h server (Samba, Debian)
netbios name = Visna

server role = standalone server
security = user
map to guest = never
obey pam restrictions = yes

# Character encoding
unix charset = UTF-8

# Logging
log file = /var/log/samba/log.%m
max log size = 1000
logging = file
log level = 1

# Disable printing completely
printcap name = /dev/null
load printers = no
disable spoolss = yes
printing = bsd

# Protocol hygiene
server min protocol = SMB2_10

# Network binding (only if you have multiple interfaces)
#interfaces = lo eth0
#bind interfaces only = yes

# macOS / Apple compatibility
vfs objects = fruit streams_xattr
fruit:metadata = stream
fruit:model = MacSamba
fruit:posix_rename = yes
fruit:veto_appledouble = no
fruit:wipe_intentionally_left_blank_rfork = yes
fruit:delete_empty_adfiles = yes
fruit:nfs_aces = no
unix extensions = no

# Extended attributes
ea support = yes

# Performance
use sendfile = yes
min receivefile size = 16384

# Windows + macOS compatibility
store dos attributes = yes

# Hide common junk files (Windows + macOS)
veto files = /._*/.DS_Store/.Trashes/Thumbs.db/desktop.ini/ehthumbs.db/
delete veto files = yes
```

> **Note:** `dos filemode` is set **per-share below, not here** — and it's explicitly `no` on every share (see §7). Forced create/directory modes are the source of truth for permissions; `dos filemode = yes` would let clients override them by toggling the DOS read-only attribute, which defeats the point of forcing modes. Keeping it `no` explicitly (rather than just omitting it) documents that this was a deliberate choice, not an oversight.

### 5. Share: `[Docker]` — `/fastpool/appdata`

Owner `perry:grp-samba-appdata` (1000:2001) — matches container `PUID/PGID`.

```bash
sudo chown -R perry:grp-samba-appdata /fastpool/appdata
sudo chmod -R g+rwX,o-rwx /fastpool/appdata
sudo chmod 2770 /fastpool/appdata
sudo find /fastpool/appdata -type d -exec chmod g+s {} +
```

> ⚠️ Re-run all three steps above whenever a **new child dataset** is created under `/fastpool/appdata` — new ZFS datasets start at root-owned `755` with no setgid, regardless of the parent.

```bash
sudo nano /etc/samba/smb.conf
```

```ini
[Docker]
comment = SSD zfs pool samba share
path = /fastpool/appdata
browseable = yes
read only = no
# Access control
valid users = @grp-samba-appdata
force user = perry
force group = grp-samba-appdata
# Permissions
create mask = 0660
force create mode = 0660
directory mask = 2770
force directory mode = 2770
inherit permissions = no
dos filemode = no
# Optional extras. Should be per-share, not global
# better write performance (safe on NVMe)
# HDD should be strict sync = yes
# SSD should be strict sync = no
strict sync = no
sync always = no
# fruit settings stay in global
```

### 6. Share: `[Perry]` — `/fastpool/documents/perry`

Owner `root:grp-samba-perry` — personal docs, group-owned rather than user-owned.

```bash
sudo chown -R :grp-samba-perry /fastpool/documents/perry
sudo chmod -R g+rwX,o-rwx /fastpool/documents/perry
sudo chmod 2770 /fastpool/documents/perry
sudo find /fastpool/documents/perry -type d -exec chmod g+s {} +
```

```bash
sudo nano /etc/samba/smb.conf
```

```ini
[Perry]
comment = SSD zfs pool samba share
path = /fastpool/documents/perry
browseable = yes
read only = no
# Access control
valid users = @grp-samba-perry
force group = grp-samba-perry
# Permissions
create mask = 0660
force create mode = 0660
directory mask = 2770
force directory mode = 2770
inherit permissions = no
dos filemode = no
# Optional extras. Should be per-share, not global
# better write performance (safe on NVMe)
# HDD should be strict sync = yes
# SSD should be strict sync = no
strict sync = no
sync always = no
# fruit settings stay in global
```

### 7. Share: `[MediaVault]` — `/bulkpool/mediavault`

Owner `perry:grp-samba-mediavault` (1000:2003).

When the Docker container is `PUID/PGID` (1000:2001), use `group_add: -"2003"`.

```bash
sudo chown -R perry:grp-samba-mediavault /bulkpool/mediavault
sudo chmod -R g+rwX,o-rwx /bulkpool/mediavault
sudo chmod 2770 /bulkpool/mediavault
sudo find /bulkpool/mediavault -type d -exec chmod g+s {} +
```

```bash
sudo nano /etc/samba/smb.conf
```

```ini
[MediaVault]
comment = HDD zfs pool Samba share
path = /bulkpool/mediavault
browseable = yes
read only = no
# Access control
valid users = @grp-samba-mediavault
force user = perry
force group = grp-samba-mediavault
# Permissions
create mask = 0660
force create mode = 0660
directory mask = 2770
force directory mode = 2770
inherit permissions = no
dos filemode = no
# Optional extras. Should be per-share, not global
# better write performance (safe on NVMe)
# HDD should be strict sync = yes
# SSD should be strict sync = no
strict sync = yes
sync always = no
# fruit settings stay in global
```

> Note `strict sync = yes` here vs `no` on the SSD-backed shares — a deliberate write-integrity/throughput tradeoff for the HDD pool.

### 8. Validate, Enable, and Start

```bash
sudo testparm -s
sudo systemctl enable --now smbd nmbd
sudo systemctl status smbd nmbd
```

On Debian 13, double-check unit names match what's installed:

```bash
systemctl list-unit-files | grep -i smb
systemctl list-unit-files | grep -i nmb
sudo systemctl is-active smbd nmbd
```

### 9. After Any Config Change

```bash
sudo testparm -s
sudo systemctl restart smbd nmbd
sudo systemctl status smbd nmbd
```

### 10. Firewall

```bash
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

### Quick Reference: Groups → Shares

| Group                  | GID  | Share          | Path                         | Force user | Force group          |
|------------------------|------|----------------|------------------------------|------------|----------------------|
| `grp-samba-appdata`    | 2001 | `[Docker]`     | `/fastpool/appdata`          | perry      | perry                |
| `grp-samba-perry`      | 2002 | `[Perry]`      | `/fastpool/documents/perry`  | —          | grp-samba-perry      |
| `grp-samba-mediavault` | 2003 | `[MediaVault]` | `/bulkpool/mediavault`       | perry      | perry                |

### Known Gotchas

- **New child datasets reset permissions.** Re-chown/chmod/setgid after creating any dataset under `appdata`, `documents/perry`, or `mediavault`.
- **`dos filemode` stays `no` on all shares** — forced create/directory modes are the single source of truth for permissions; don't let a client's read-only toggle override them.

---

## Install Docker Community Edition

Use the official Docker install instructions:

- https://docs.docker.com/engine/install/debian/
- https://docs.docker.com/engine/install/linux-postinstall/

### Add log rotation + DNS

Reference: https://forums.lawrencesystems.com/t/toms-docker-setup-simple-clean-and-easy-to-back-up/25904

Comcast/Xfinity's DNS resolver:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["75.75.75.75", "75.75.76.76"]
}
```

Or use **Quad9** (Recommended). It blocks known malicious domains. Quad9 maintains a blocklist of domains known to be associated with malware, phishing, and botnet command-and-control servers. When a container tries to resolve one of those domains, Quad9 refuses to answer (returns NXDOMAIN) instead of resolving it.

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["9.9.9.9", "149.112.112.112"]
}
```

Or use Cloudflare with `1.1.1.2` / `1.0.0.2` (blocks malware + phishing):

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["1.1.1.2", "1.0.0.2"]
}
```

#### Steps

Check if the file already exists and what's in it:

```bash
sudo cat /etc/docker/daemon.json 2>/dev/null
```

If it doesn't exist yet, or is empty, just create it fresh:

```bash
sudo nano /etc/docker/daemon.json
```

Paste the merged JSON above, save.

Validate it's actually valid JSON before restarting Docker (a syntax error here will make Docker fail to start):

```bash
python3 -m json.tool /etc/docker/daemon.json
```

If it prints the JSON back out cleanly, you're good. If it errors, fix the reported line (usually a missing/extra comma).

Restart Docker to apply:

```bash
sudo systemctl restart docker
```

Verify both settings took effect:

```bash
docker info | grep -A3 "Logging Driver"
docker info | grep -i dns
```

Or more directly, run a test container and check:

```bash
docker run --rm alpine cat /etc/resolv.conf
docker inspect <container_name> | grep -A5 LogConfig
```

> **Note on log settings:** this only applies to containers created after the restart — existing containers keep their original DNS config until recreated (`docker compose down && docker compose up -d`, or `docker restart` isn't enough — needs a recreate).

---

## Install Tailscale

> **Note:** Tailscale does not work well with desktop or browser ProtonVPN turned on. Turn it off, or use a split tunnel to the server IP.

**KISS:** Don't use an exit node or subnets — they're not needed for a home server.

### Step 1 — Add the device

Use the Tailscale website to add a device:

https://login.tailscale.com/admin/machines/new-linux

Example install:

```bash
curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up --auth-key=tskey-auth-####
```

### Step 2 — Advertise SSH on the host

Docs: https://tailscale.com/docs/features/tailscale-ssh#advertise-ssh-on-the-host

```bash
sudo tailscale set --ssh
```

Or combine it with Step 1:

```bash
curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up --auth-key=tskey-auth-#### --ssh
```

---

## DNS Setup: Omega, Visna and AP-5

All three servers use the same setup. This guide is the single reference for it, and for rebuilding or adding a server.

**What every server ends up with:**

- The server looks up names through Quad9, and those lookups are **encrypted** (DNS-over-TLS).
- Tailscale names like `visna.<tailnet-name>.ts.net` still work.
- Docker containers use Quad9 directly, **not encrypted**. That is on purpose, because encrypting containers can mean extra upkeep every time you add a Compose stack. The appendix at the end explains how to change that if you ever want to.

### The only differences between the servers

| Server | Network interface |
|--------|-------------------|
| Omega  | `enp2s0`          |
| Visna  | `enp1s0`          |
| AP-5   | `eno1`            |

Everywhere this guide says `INTERFACE`, use that server's name from the table. If a server ever shows a different name, run `ip -br addr` to find it.

Everything else is identical. Do the steps in order. Each step ends with a quick check.

### Step 1. Make sure dhcpcd starts at boot

dhcpcd is the program that gets the server its network address. It runs as a service.

```bash
sudo apt install dhcpcd
sudo systemctl enable --now dhcpcd.service
```

**Check:**

```bash
systemctl status dhcpcd
ip -br addr show INTERFACE
```

The service should say `active (running)`, and the interface should show an address like `10.0.0.x/24`.

### Step 2. Tell dhcpcd to stay out of DNS

```bash
sudo nano /etc/dhcpcd.conf
```

Make sure these three lines are in it, each on its own line, not indented and not inside any `interface` block:

```
nohook resolv.conf
nooption domain_name_servers, domain_search, domain_name
allowinterfaces en*
```

What they do:

- The first two stop dhcpcd from passing Comcast's DNS servers to the rest of the system. You need **both**.
- The third makes dhcpcd manage only wired `en...` interfaces and ignore the virtual interfaces Docker creates. The `en*` pattern matches `enp2s0`, `enp1s0` and `eno1`, so the same line works on all three servers.

**Do not copy an exact interface name from another server.** If you prefer an exact name, it must be that server's own: `allowinterfaces enp2s0` on Omega, `enp1s0` on Visna, `eno1` on AP-5. A name that doesn't match leaves the server with no network address the next time dhcpcd restarts or the server reboots.

Save (Ctrl+O, Enter) and exit (Ctrl+X), then restart:

```bash
sudo systemctl restart dhcpcd
```

**Check that the server still has its network:**

```bash
ip -br addr show INTERFACE
ip route show default
ping -c 2 9.9.9.9
```

You should see an address, a default route through your router (`10.0.0.1`), and replies from the ping.

### Step 3. Install systemd-resolved

This is the program that handles DNS from now on.

```bash
sudo apt install systemd-resolved
```

### Step 4. Point it at Quad9 with encryption

Put your settings in your own file in the `resolved.conf.d` folder. Package updates never touch that folder, so they can't overwrite your settings. The stock `/etc/systemd/resolved.conf` stays as it came.

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo nano /etc/systemd/resolved.conf.d/quad9.conf
```

Put exactly this in the file:

```ini
[Resolve]
DNS=
DNS=9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net 2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net
DNSOverTLS=yes
DNSSEC=yes
```

What the lines do:

- The empty `DNS=` line clears any server list set earlier, for example in `resolved.conf`. Server lists from different files add together, so without it you could end up with duplicates.
- The `#dns.quad9.net` after each address lets the server confirm it is really talking to Quad9.
- `DNSOverTLS=yes` means encryption is required. If it ever fails, the lookup fails rather than quietly going out unencrypted.

**If you already put these lines in `/etc/systemd/resolved.conf`** (on any of your three servers, you probably did), put that file back to its original state by putting a `#` in front of the lines you added. That leaves one place to look.

**Check the combined settings:**

```bash
systemd-analyze cat-config systemd/resolved.conf
```

This prints every file that contributes, in order, with your `quad9.conf` last. The settings from your file are the ones that apply.

### Step 5. Turn it on and let it manage `/etc/resolv.conf`

```bash
sudo systemctl enable --now systemd-resolved
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

### Step 6. Clear leftover DNS settings and restart

```bash
sudo resolvectl revert INTERFACE
sudo systemctl restart systemd-resolved
```

**Check:**

```bash
resolvectl status
```

You want to see:

- `resolv.conf mode: stub`
- The four Quad9 addresses under **Global**, with `+DNSOverTLS`
- Under the interface's own Link entry: **no** DNS servers and **no** `comcast.net` domain

(`DNSSEC=yes/unsupported` on the Global line is harmless, ignore it.)

### Step 7. Connect Tailscale

```bash
sudo tailscale set --accept-dns=true
sudo systemctl restart tailscaled
```

**Check:**

```bash
cat /etc/resolv.conf
```

The bottom of the file should read:

```
nameserver 127.0.0.53
options edns0 trust-ad
search <tailnet-name>.ts.net
```

### Step 8. Test the server's DNS

```bash
nslookup cloudflare.com
nslookup visna.<tailnet-name>.ts.net
```

Both should answer, and both should show `Server: 127.0.0.53`.

Optional extra check on Tailscale:

```bash
resolvectl status tailscale0
```

It should list `ts.net` and your tailnet name, with `Default Route: no`.

### Step 9. Set Docker to use Quad9

```bash
sudo nano /etc/docker/daemon.json
```

It should contain exactly:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["9.9.9.9", "149.112.112.112"]
}
```

Restart Docker:

```bash
sudo systemctl restart docker
```

**Check:**

```bash
docker run --rm alpine cat /etc/resolv.conf
docker run --rm alpine nslookup cloudflare.com
```

You should see `nameserver 9.9.9.9` and `nameserver 149.112.112.112`, and the lookup should work.

Containers that were already running keep their old settings until they are recreated:

```bash
docker compose up -d --force-recreate
```

### Step 10. Prove the encryption is working

```bash
sudo resolvectl flush-caches; nslookup example.org > /dev/null; sudo ss -tnp | grep ':853'
```

You should see a line containing `ESTAB`, `systemd-resolve`, and a Quad9 address ending in `:853` (it may be `9.9.9.9` or one of the IPv6 addresses starting `2620:fe::`). That is an encrypted connection to Quad9.

If it prints nothing, run it again. A lookup answered from memory, or a connection that has just closed itself, can make it come up empty.

### Checking that all three servers match

Run this block on each server and compare:

```bash
resolvectl status | head -8
tail -3 /etc/resolv.conf
grep -E 'nohook|nooption|allowinterfaces' /etc/dhcpcd.conf
cat /etc/docker/daemon.json
sudo resolvectl flush-caches; nslookup example.org > /dev/null; sudo ss -tnp | grep ':853'
systemctl status dhcpcd | head -4
systemd-analyze cat-config systemd/resolved.conf | grep -v '^#' | grep -v '^$'
```

On every server you should see:

- `resolv.conf mode: stub`, Quad9 under Global, and `+DNSOverTLS`
- `search <tailnet-name>.ts.net` at the bottom of `resolv.conf`
- The three dhcpcd lines, with `allowinterfaces` matching that server's own interface (or `en*`)
- `"dns": ["9.9.9.9", "149.112.112.112"]` in `daemon.json`
- An `ESTAB` connection to a Quad9 address on port 853
- dhcpcd `active (running)`
- The settings from `quad9.conf` (the Quad9 `DNS=` line, `DNSOverTLS=yes`, `DNSSEC=yes`) in the combined resolved settings

The `allowinterfaces` line is the one most worth checking. It is the only line that has to differ between servers (unless you use `en*`).

### Things to remember

- **Containers are not encrypted.** They talk to Quad9 directly in plain text. This is a deliberate choice.
- **Containers can't look up Tailscale names** (the `.ts.net` ones). If a container needs to reach another of your machines, use that machine's `100.x.x.x` address.
- **After any `firewall-cmd --reload`,** run `sudo systemctl restart docker`. A firewall reload can wipe Docker's network rules.
- **Package updates and your config files.** The resolved settings live in `/etc/systemd/resolved.conf.d/quad9.conf`, which packages never touch. dhcpcd has no such folder, so your edits are in `/etc/dhcpcd.conf` itself. Debian does not silently replace a file you have edited: if an update brings a new version, it asks, and the default answer keeps yours. Never choose "install the package maintainer's version" for `/etc/dhcpcd.conf`. As a safety net, keep a copy with `sudo cp /etc/dhcpcd.conf /root/dhcpcd.conf.mine`. After an upgrade, a leftover `/etc/dhcpcd.conf.dpkg-dist` file means the package offered a new version and yours was kept, which is fine.
- **Restarting dhcpcd:** always use `sudo systemctl restart dhcpcd`. Do not run `dhcpcd -x` or start dhcpcd by hand, because that conflicts with the service.

### If something goes wrong

**Names on your Tailscale network stop resolving, but normal websites work.**  
Something has overwritten `/etc/resolv.conf`. Check that `nohook resolv.conf` and the `nooption` line are both in `/etc/dhcpcd.conf`, then run:

```bash
sudo systemctl restart dhcpcd
sudo systemctl restart tailscaled
```

**`resolvectl status` still shows Comcast servers under the interface.**  
Run `sudo resolvectl revert INTERFACE` and `sudo systemctl restart systemd-resolved`. If they come back, the `nooption` line is missing from `dhcpcd.conf`.

**`resolvectl status` says `resolv.conf mode: foreign`.**  
The link in Step 5 is missing. Run the `ln -sf` command from Step 5 again, then `sudo systemctl restart tailscaled`.

**Odd `169.254.x.x` addresses on `veth...` interfaces.**  
The `allowinterfaces` line is missing from `dhcpcd.conf`. Add it and restart dhcpcd. The old addresses disappear when the containers are recreated.

**The server has no network address after a restart or reboot.**  
Check `allowinterfaces` in `/etc/dhcpcd.conf`. If the name doesn't match the server's real interface, dhcpcd ignores it. Change the line to `allowinterfaces en*`, then `sudo systemctl restart dhcpcd`.

**No DNS at all.**  
Because encryption is required, a problem reaching Quad9 on port 853 breaks lookups completely. Temporarily set `DNSOverTLS=no` in `/etc/systemd/resolved.conf.d/quad9.conf`, run `sudo systemctl restart systemd-resolved`, and see whether lookups return. If they do, the problem is the encrypted connection (a firewall or router blocking port 853). Set it back to `yes` once you have found the cause.

### Appendix: encrypted DNS for Docker containers (documented, not in use)

None of the three servers uses this. It is written down in case you want it someday.

**What it does:** containers send their lookups to the server's own `systemd-resolved`, which encrypts them to Quad9. It also means containers should be able to resolve `*.ts.net` names, because resolved handles the Tailscale routing.

**What it costs:** the lookups have to be pointed at an address that resolved listens on. That means one extra listener, and possibly one per Docker network (see Option B). Neither option has been tested on these servers, so test before you rely on it.

#### Option A: one listener for every container (try this first)

Containers on any network can usually reach the server's own `docker0` address (`172.17.0.1`). If that works on your servers, you need one listener and no per-network upkeep.

**1. Create the folder and file:**

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo nano /etc/systemd/resolved.conf.d/docker.conf
```

Put this in it:

```ini
[Resolve]
DNSStubListenerExtra=172.17.0.1
```

**2. Restart resolved and check it is listening:**

```bash
sudo systemctl restart systemd-resolved
sudo ss -tulnp | grep '172.17.0.1:53'
```

You should see `systemd-resolve` on `172.17.0.1:53`, for both UDP and TCP.

**3. Point Docker at it.** Change the `dns` line in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["172.17.0.1"]
}
```

```bash
sudo systemctl restart docker
```

**4. Test from a container on a custom Compose network, not just the default one.** The default `bridge` network will always work and tells you nothing:

```bash
docker network ls
docker run --rm --network <one-of-your-compose-networks> alpine nslookup cloudflare.com
```

If that answers, Option A works and you are done with setup. If it times out, go to Option B.

#### Option B: one listener per Docker network

Use this only if Option A times out from custom networks.

**1. List every network's gateway address:**

```bash
for net in $(docker network ls --format '{{.Name}}' | grep -v -E '^(host|none)$'); do
  echo -n "$net: "
  docker network inspect "$net" --format '{{(index .IPAM.Config 0).Gateway}}'
done
```

**2. Add one `DNSStubListenerExtra` line per gateway** in `/etc/systemd/resolved.conf.d/docker.conf`:

```ini
[Resolve]
DNSStubListenerExtra=172.17.0.1
DNSStubListenerExtra=172.18.0.1
DNSStubListenerExtra=172.19.0.1
```

Then `sudo systemctl restart systemd-resolved`.

**3. In each Compose file, point every service at its own network's gateway:**

```yaml
services:
  your-service:
    dns:
      - 172.19.0.1
```

**The upkeep:** every new Compose stack needs a new line in `docker.conf` and a `dns:` entry in its compose file. A stack that is missing them quietly falls back to Docker's defaults, with no error. Gateway addresses can also change when a network is recreated. If you want them to stay put, give each network a fixed subnet in its compose file.

#### Checking that containers really are encrypted

```bash
docker run --rm --network <network> alpine cat /etc/resolv.conf
sudo resolvectl flush-caches; docker run --rm --network <network> alpine nslookup example.org; sudo ss -tnp | grep ':853'
```

The container's `resolv.conf` should show the gateway address (or `127.0.0.11`, Docker's built-in resolver, on Compose networks), not `9.9.9.9`. After the lookup, an `ESTAB` connection from `systemd-resolve` to Quad9 on port 853 means the container's query went out encrypted.

#### Two things to watch

- **After a reboot,** check that the listener came back: `sudo ss -tulnp | grep '172.17.0.1:53'`. If resolved starts before Docker creates its network, the extra listener may be missing. If it is, `sudo systemctl restart systemd-resolved` fixes it. I haven't confirmed whether this happens on your servers.
- **If container lookups time out,** the firewall may be blocking DNS from the Docker networks to the server. Check `sudo firewall-cmd --get-active-zones` to see which zone holds the Docker interfaces, and that it allows DNS.

#### Going back to plain Quad9 for containers

```bash
sudo rm /etc/systemd/resolved.conf.d/docker.conf
sudo systemctl restart systemd-resolved
```

Then set `daemon.json` back to `"dns": ["9.9.9.9", "149.112.112.112"]` and run `sudo systemctl restart docker`. Recreate any containers that should pick up the change.

---

## Install Dockhand Free Edition

References:

- https://dockhand.pro/
- https://dockhand.pro/manual/#quick-start
- https://www.youtube.com/watch?v=F7aXfYGf5q4

Create the directory on the host (boot disk):

```bash
mkdir -p /opt/dockhand
cd /opt/dockhand
sudo nano /opt/dockhand/compose.yaml
```

Copy, paste and save:

```yaml
services:
  dockhand:
    image: fnsys/dockhand:latest
    container_name: dockhand
    ports:
      - "3000:3000"
    environment:
      - PUID=1000  # Your user ID (run: id -u)
      - PGID=1000  # Your group ID (run: id -g)
    group_add:
      # note: Find the Docker group ID on your host (Linux) with   stat -c '%g' /var/run/docker.sock
      - "999"  # Docker group GID for socket access
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./dockhand-db:/app/data
```

Start the server:

```bash
docker compose up -d
```

If you ever need to recreate dockhand (not `docker compose down && docker compose up -d`, or `docker restart`):

> No data is lost when you recreate dockhand.

```bash
cd /opt/dockhand
docker compose up -d --force-recreate
```

Confirm the new daemon.json settings (DNS + log rotation) actually took effect on this fresh container:

```bash
docker compose exec dockhand cat /etc/resolv.conf
docker inspect dockhand | grep -A5 LogConfig
```

---

## Install Netdata Community Edition

### 1.0 Use netdata kickstart.sh

Reference guides:

- [Install Netdata with kickstart.sh](https://learn.netdata.cloud/docs/netdata-agent/installation/linux/)
- [A Complete Netdata Course — 8 Episodes, Zero to Production](https://www.netdata.cloud/resources/netdata-tutorial/)

### 2.0 Enable Bearer Token Protection to Secure Port 19999

Reference: [bearer-token-protection](https://learn.netdata.cloud/docs/netdata-agent/configuration/securing-agents/bearer-token-protection)

Edit your `netdata.conf` using the edit-config script:

```bash
cd /etc/netdata
sudo netdatacli dumpconfig > ~/netdata.conf
cd ~
sudo mv netdata.conf /etc/netdata/netdata.conf
cd /etc/netdata
sudo ./edit-config netdata.conf
```

Add or modify the `[web]` section:

```ini
[web]
bearer token protection = yes
```

Restart Netdata to apply:

```bash
sudo systemctl restart netdata
```

### 3.0 Limit netdata log size

```bash
sudo nano /etc/systemd/journald@netdata.conf
```

```ini
[Journal]
Storage=persistent
SystemMaxUse=200M
SystemMaxFileSize=32M
MaxRetentionSec=1month
MaxFileSec=1week
```

```bash
sudo systemctl restart systemd-journald@netdata.service
sudo journalctl --namespace=netdata --disk-usage
```

---

# AppArmor Basics for a Debian Server

A short guide to what to install and what to check now and then.

## What AppArmor does

AppArmor confines programs to a defined set of allowed actions (files they can read or write, network access, capabilities). On Debian 10 and later it is enabled by default, so on most servers you leave it on and mostly ignore it until it blocks something.

## Install (one time)

```
sudo apt update
sudo apt install apparmor apparmor-utils
```

- `apparmor`: the service and the default profiles. Usually already installed.
- `apparmor-utils`: gives you `aa-status`, `aa-enforce`, `aa-complain`, `aa-logprof` and `aa-unconfined`.
- `apparmor-profiles` (optional): extra profiles, most of which ship in complain mode. Skip unless you need one.
- `apparmor-profiles-extra` (optional): a larger set of mostly experimental profiles. Skip unless you need a specific one.

No restart is needed after installing. The package loads profiles and enables the service itself.

## Confirm it is active

```
systemctl is-enabled apparmor                  # enabled
cat /sys/module/apparmor/parameters/enabled    # Y
sudo aa-status                                 # profiles loaded
```

`active (exited)` in `systemctl status apparmor` is normal. The service loads the profiles and then exits.

## Occasional checks

Do these monthly, after big package updates, or whenever a service misbehaves.

```
sudo aa-status                                  # profiles loaded, enforce vs complain
sudo aa-unconfined                              # network services with no profile
sudo journalctl -b | grep -i apparmor | grep DENIED
```

- **aa-status**: shows how many profiles are in enforce mode (blocking) and complain mode (logging only).
- **aa-unconfined**: lists network-listening services with no AppArmor profile.
- **The journal search**: no output means nothing has been blocked.
- **ps auxZ**: shows the confinement label of every running process, if you want a closer look.

If you install `auditd`, denials often go to `/var/log/audit/audit.log` instead of the kernel journal, so the journal search can come up empty. Use `sudo ausearch -m AVC -ts recent` in that case.

## When a service breaks with "permission denied"

Try this when Unix permissions look fine, especially after moving a data directory.

1. Look for denials with the journal command above.
2. Put the profile in complain mode to confirm AppArmor is the cause:

   ```
   sudo aa-complain /etc/apparmor.d/<profile>
   ```
3. Fix it with `sudo aa-logprof`, or add a rule to `/etc/apparmor.d/local/<profile>`, for example:

   ```
   /srv/mydata/** rwk,
   ```
4. Reload the profile and switch back to enforce mode:

   ```
   sudo apparmor_parser -r /etc/apparmor.d/<profile>
   sudo aa-enforce /etc/apparmor.d/<profile>
   ```

## Command cheat sheet

| Task | Command |
| --- | --- |
| Overall status | `sudo aa-status` |
| Unprofiled network services | `sudo aa-unconfined` |
| Log violations only | `sudo aa-complain <profile>` |
| Block violations | `sudo aa-enforce <profile>` |
| Update profile from logged denials | `sudo aa-logprof` |
| Reload one profile after editing | `sudo apparmor_parser -r /etc/apparmor.d/<profile>` |
| Reload all profiles | `sudo systemctl reload apparmor` |

## Where things live

- `/etc/apparmor.d/`: active profiles.
- `/etc/apparmor.d/local/`: your own additions. Put changes here so package updates don't overwrite them.
- `/etc/apparmor.d/disable/`: disabled profiles.

## Rules of thumb

- Don't disable AppArmor globally. Use complain mode or add a rule for the one profile causing trouble.
- Prefer `aa-complain` over `aa-disable` when troubleshooting, since it keeps logging what would have been blocked.
- After editing a profile, reload it with `apparmor_parser -r`. You don't need to restart the whole service.
- Leave Docker's `docker-default` profile alone.
- Profiles already in complain mode, such as the Samba RPC helpers (`samba-dcerpcd`, `rpcd_lsad`), are fine to leave as they are on a LAN-only file server.


---

# Final Notes

This README is a personal operational runbook for building and maintaining a Debian 13 home server using OpenZFS, Docker, Samba, Tailscale, and system monitoring. It is intentionally practical: commands are written as they were used, with the expectation that the operator has root access and is working on a physical or virtual Debian system.

The most important practices throughout are:
- use `sudo` for privileged operations
- verify commands before and after major changes
- keep backups before ZFS and Samba modifications
- validate DNS and firewall changes immediately
- use the correct interface name or the `en*` pattern when editing `dhcpcd.conf`

---



This file represents the core notes and commands required to set up and maintain the server. It can be expanded over time as the system evolves.
