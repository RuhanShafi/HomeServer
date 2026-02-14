# System Setup

## Table of Contents
* [Home](https://github.com/RuhanShafi/HomeServer/blob/main/README.md)
* Initial Setup - You're Already Here :) 
* [Apps](https://github.com/RuhanShafi/HomeServer/blob/main/apps/README.md) - A brief rundown on all the different app and services that I run on my Home Lab
* [Media](https://github.com/RuhanShafi/HomeServer/blob/main/media/README.md) - Deeper Dive into my Jellyfin & *arr stack Setup

### Installing Proxmox

### Post Install Steps

Install Proxmox using Standard Settings

#### Switching Enterprise Repositories to No-Subscription Repositories and Updating System

1. Navigate to _Node > Repositories_ Disable the enterprise repositories.
2. Now click Add and enable the no subscription repository. Finally, go _Updates > Refresh_.
3. Upgrade your system by clicking _Upgrade_ above the repository setting page.

#### Configuring systemd to ignore Shutting Laptop Lid (Optional if not using Laptop)

1. `nano /etc/systemd/logind.conf`
2. Find the following lines and change
```bash
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
LidSwitchIgnoreInhibited=HandleLidSwitchExternalPower
```
3. Save and quit
4. Restart systemd with the following command: 
```bash
systemctl restart systemd-logind
```

Now you can safely shut the Laptop Lid without Proxmox suspending. Be sure to reload the Proxmox web page to confirm if the systemd configs were applied.

#### Configuring Storage Solution using `MergerFS`

> [!NOTE]
> This is the primary point where my setup diverges from TechHut's as I don't have the financial funds to drop like 10k on a Home lab... yet

>[!WARNING]
> This configuration assumes the following; a) a fresh installation without advanced storage settings during the installation, b) My specific storage solution

1. Print out exact disk IDs
```bash
ls -la /dev/disk/by-id/ | grep -E "sda|sdb"
```
2. Shrink pve-data to free up space for tank:
Check current pve-data usage first:
```bash
lvs
df -h /var/lib/vz
```
Then shrink it (adjust size based on what you need for VMs/containers):
```bash
# Stop any running VMs/containers first
# Shrink by 800GB to leave room for flash + tank
lvreduce -L -800G pve/data
```

3. Create new logical volumes:
```bash
# 100GB for flash pool
lvcreate -L 100G -n flash pve
mkfs.ext4 /dev/pve/flash

# Remaining space for tank-internal (~730GB)
lvcreate -l 100%FREE -n tank-internal pve
mkfs.ext4 /dev/pve/tank-internal
```

4. Format external drive:

```bash
mkfs.ext4 -L tank-external /dev/sdb1
```

5. Create mount points:

```bash
mkdir -p /mnt/flash
mkdir -p /mnt/disks/internal
mkdir -p /mnt/disks/external
mkdir -p /mnt/tank
```

6. Add to `/etc/fstab`:
```bash
# Flash storage
/dev/pve/flash /mnt/flash ext4 defaults 0 2


# Individual disks for mergerfs
/dev/pve/tank-internal /mnt/disks/internal ext4 defaults 0 2
/dev/sdb1 /mnt/disks/external ext4 defaults,nofail 0 2

# MergerFS pool
/mnt/disks/* /mnt/tank fuse.mergerfs defaults,allow_other,use_ino,cache.files=partial,dropcacheonclose=true,category.create=mfs 0 0
```

7. Install `mergerfs`:
```
apt update
apt install mergerfs
```
8. Mount everything:
```bash 
mount -a
```
9. Verify the setup:
```bash
df -h
lsblk
```
