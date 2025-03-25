---
layout: post
title: Debian btrfs snapper
---

In this section, I'll be installing Debian Linux with BtrFS and snapper for auto snapshots. This is the first part in intermediate optional installs. If you just want plain Debian, feel free to just install Debian and skip to Part 2.

I'll be following the guide from [this tutorial](https://medium.com/@inatagan/installing-debian-with-btrfs-snapper-backups-and-grub-btrfs-27212644175f) with some modifications.

### Motivation
Proxmox has this neat feature where you can create a snapshot of a VM or an LXC and rollback to it in case you mess something up. So why not borrow this feature for our entire base operating system. I'll be making use of a Copy-On-Write filesystem BtrFS. You also have the option of using ZFS if you like to, but I'll stick with BtrFS as it is good enough for a single disk.

### Installation
First things first, connect the mini PC with a display and a keyboard at the very least. Optionally with an ethernet as well as WiFi connection has been rather finicky for me during Debian installs.

[Download the Debian ISO](https://www.debian.org/) and create a bootable USB either using [Balena Etcher](https://etcher.balena.io/) or create a [Ventoy](https://www.ventoy.net/en/index.html) disk if you want to create one USB drive for multiple ISOs.

You might want to disable secure boot in the BIOS settings. After that, boot from the USB and you should now boot into the Debian installer boot screen. Here, go to `Advanced Options -> Expert Install`. You can also select the `Graphical Expert Install` if you have a mouse plugged in, but honestly, it doesn't offer any additional value so just stick with regular non graphic expert install for a snappier experience.

![Advanced Options](../assets/img/1.1/Boot1.png)
![Expert Install](../assets/img/1.1/Boot2.png)

Just keep pressing enter here or go through each menu item and change if needed. Create a local admin user and disable root user for extra security. Once you reach Partition disks menu, then we can begin with the main installation process.

In the menu, select `Manual Partition` item.

![Manual Partition](../assets/img/1.1/partition1.png)

In the next screen, select the drive where debian is to be installed and create a new partition of type gpt (for UEFI systems). If using an old BIOS system, use msdos. 

You should now see free space under the disk name. Select the free space and create a new partition of size `500M`. Change the use as to EFI System Partition and make sure Bootable flag is on.

![Boot Partition](../assets/img/1.1/partition2.png)

In the next screen, again select the free space under the new partition we just created and create a new partition, this time just press enter to default to the entire remaining space. Change the use as to btrfs journaling file system. Leave the rest as defaults and select done.

> In the linked article, they setup an encrypted volume here. However, unless you want to connect a display and keyboard to the mini PC and enter the password every time it reboots, I suggest leaving it unencrypted. Alternatively, you can try to auto-decrypt using the onboard TPM2 chip.
{: .prompt-info}

![Root Partition](../assets/img/1.1/partition3.png)

Now just select finish and select no to swap space (I'll be using ZRAM for swap later). Write the changes to disk in the next step, but don't install the system yet, we'll need to configure btrfs before installing.

### Configuring BTRFS
Now we'll need to press `Ctrl + Alt + F2` to enter the BusyBox terminal and press Enter to activate it.

Type `df -h` in the terminal to see your mounted drives at /target and /target/boot/efi. Do note down the filesystem names for each of them. If using SATA drives (HDD or SSD), it should be /dev/sda1 and /dev/sda2, or something similar. If using NVME drive, it should be /dev/nvme0p1 and /dev/nvme0p2, or something similar. Unmount both of them with the following commands:
```bash
umount /target/boot/efi
umount /target
```

Then mount the root partition drive with the following command (make sure to replace the root_partition variable with the correct filesystem name, for example `sda2`):
```bash
mount /dev/${root_partition} /mnt
```

Now move into the mounted directory and have a look at its contents.
```bash
$ cd /mnt
$ ls
@rootfs
```

Rename @rootfs to @ with the following command to make it compatible with Timeshift in case I use it sometime in the future.
```bash
mv @rootfs/ @
```

Now create btrfs subvolumes with the following commands. Creating subvolumes prevents unnecessary data taking up space in our snapshots. We really don't want to include logs, cache or tmp files in our snapshots. We should also have a separate backup strategy for our home directory, but I'm not going to go into that as I don't need it at the moment.
```bash
btrfs su cr @snapshots
btrfs su cr @home
btrfs su cr @log
btrfs su cr @cache
btrfs su cr @tmp
```

Now we mount the root subvolume (@) and create various directories for all the other subvolumes. Then we can mount all the subvolumes and the boot partition.
> Note: Some of the mount options below are ssd specific. If using mechanical HDD, feel free to remove options such as `discard=async` and `ssd`
{: .prompt-info}
```bash
mount -o noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@ /dev/${root_partition} /target

mkdir -p /target/boot/efi /target/.snapshots /target/home /target/var/log /target/var/cache /target/var/tmp

mount /dev/${boot_partition} /target/boot/efi

mount -o noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@snapshots /dev/${root_partition} /target/.snapshots
mount -o noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@home /dev/${root_partition} /target/home
mount -o noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@log /dev/${root_partition} /target/var/log
mount -o noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@cache /dev/${root_partition} /target/var/cache
mount -o noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@tmp /dev/${root_partition} /target/var/tmp
```

To make these changes persist on reboot, we need to edit the `/etc/fstab` file and write the mount instructions we typed above inside it.
```bash
$ nano /target/etc/fstab
```
You will see a line that is mounting the @rootfs to the / mount point. We cut that line with `Ctrl + K` and then paste it 6 times with `Ctrl + U`. Then edit each line as follows (leave the UUID as it was):
```text
UUID=1234-5678-90ab-cdef /                        btrfs  noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@                   0    0
UUID=1234-5678-90ab-cdef /.snapshots              btrfs  noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@snapshots          0    0
UUID=1234-5678-90ab-cdef /home                    btrfs  noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@home               0    0
UUID=1234-5678-90ab-cdef /var/log                 btrfs  noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@log                0    0
UUID=1234-5678-90ab-cdef /var/cache               btrfs  noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@cache              0    0
UUID=1234-5678-90ab-cdef /var/tmp                 btrfs  noatime,space_cache=v2,compress=zstd:1,ssd,discard=async,subvol=@tmp                0    0
```

Now finally, we need to unmount the `/mnt` directory and go back to the installation with `Ctrl + Alt + F1`.
```bash
cd /
umount /mnt
exit
```

### Continue Installation
Press Enter on the "Install the base system" to proceed with the installation. Press Enter on all prompts to continue with defaults, but feel free to change anything you like.

In the next steps, again just press Enter a bunch of times. I made the following changes:
* Use non-free software: Yes
* Enable source repositories in APT: No
* Services to use: security updates, release updates, backported software
* Install security updates automatically (Makes life easier)
* Software selection: SSH server and standard system utilities only (Uncheck Debian desktop environment and Gnome)

When prompted to reboot, remove the installation media and press Enter to reboot into a fresh minimal Debian install. Login with the local user we setup during the installation since I disabled the root login.

### Create SWAP
Since we skipped adding a Swap partition during installation, let's create a ZRAM Swap. This creates a compressed block of Swap in RAM itself saving the trouble of constant read/write to SSD and shortening its lifespan, and will be much faster than traditional Swap. This does use a bit of CPU power to compress/decompress data in the ZRAM block, but I don't really have to worry about that running a Ryzen processor instead of something like a Celeron.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install zram-tools -y
sudo nano /etc/default/zramswap
```

In the file, uncomment `ALGO=lz4` and change `PERCENT=20` and save. Then restart the zram swap with
```bash
sudo systemctl restart zramswap.service
lsblk
```
and you should see your swap space.

### Install snapper
Install snapper with:
```bash
sudo apt install snapper
```
By default, snapper will create a subvolume ".snapshots" under our root subvolume which would then get included in the snapshots. To prevent that, we will delete the subvolume created by snapper and replace it with the one we created.
```bash
cd /
sudo umount .snapshots
sudo rm -r .snapshots/

sudo snapper -c root create-config /
sudo btrfs subvolume delete /.snapshots
sudo mkdir /.snapshots
sudo mount -av
```

Now edit snapper settings as follows:
```bash
# Add sudo group to allow our local user
sudo snapper -c root set-config 'ALLOW_GROUPS=sudo'
sudo snapper -c root set-config 'SYNC_ACL=yes'

# Set the max snapshots to 10
sudo snapper -c root set-config "NUMBER_LIMIT=10"
sudo snapper -c root set-config "NUMBER_LIMIT_IMPORTANT=10"
```

Now to configure auto snapshots, edit the `/etc/snapper/configs/root` file as follows.
```text
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="0"
TIMELINE_LIMIT_MONTHLY="0"
TIMELINE_LIMIT_YEARLY="0"
```

Finally, create an initial snapshot with the following command.
```bash
sudo snapper -c root create --description "init"
```