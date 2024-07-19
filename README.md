## Install Kununtu on BTRFS with multiple drives

### Create the partitions

```bash
# Identify the drives
lsblk
# Your drives will differ from mine but I will use /dev/sdb as the main drive
# /dev/sdd and /dev/sdc are my other SSDs i will use for the BTRFS pool
# /dev/sda is my HDD that I will use for backups

# Install gdisk
sudo apt update
sudo apt install gdisk

# Clear the partition table
sudo wipefs -a /dev/sdb

# Create the EFI partition
sudo gdisk /dev/sdb

# Create a new partition
n

# Set partition number to default (press enter)
# Set the first sector to the default
# Set the patition size to 512MB
+512M

# Set the partition type to EFI
ef00

# Create the swap partition
n

# Set partition number to default
# Set the first sector to the default
# Set the partition size to your ram size
+32G

# Set the partition type to swap
8200

# Create the root partition
n

# Set partition number to default
# Set the first sector to the default
# Set the last sector to the default
# Set the partition type to default (linux filesystem)

# Write the changes
w

# Check the partitions
lsblk
# You should see the partitions /dev/sdb1, /dev/sdb2, /dev/sdb3

# Format the EFI partition
sudo mkfs.fat -F32 /dev/sdb1

# Format the swap partition
sudo mkswap /dev/sdb2

# Create the BTRFS filesystem
sudo mkfs.btrfs /dev/sdb3

# Create the BTRFS subvolumes
sudo mount /dev/sdb3 /mnt

# Create the subvolumes
sudo btrfs subvolume create /mnt/@
sudo btrfs su cr /mnt/@home
sudo btrfs su cr /mnt/@snapshots

# Unmount the drive
sudo umount /mnt

# Mount the subvolumes
sudo mount -t btrfs -o noatime,compress=lzo,space_cache=v2,subvol=@ /dev/sdb3 /mnt

# Create the directories
sudo mkdir -p /mnt/{boot,home,.snapshots}

# Mount the other subvolumes
sudo mount -t btrfs -o noatime,compress=lzo,space_cache=v2,subvol=@home /dev/sdb3 /mnt/home
sudo mount -t btrfs -o noatime,compress=lzo,space_cache=v2,subvol=@snapshots /dev/sdb3 /mnt/.snapshots

# Check if the subvolumes are mounted
lsblk
# You should see the subvolumes mounted on /mnt, /mnt/home, /mnt/.snapshots under /dev/sdb3

# Create a folder for the EFI partition
sudo mkdir -p /mnt/boot/efi

# Mount the EFI partition
sudo mount /dev/sdb1 /mnt/boot/efi
```

### Install the system

```bash
# Install debootstrap
sudo apt install debootstrap

# Get the release name
lsb_release -a

# Install the base system
sudo debootstrap noble /mnt
# sudo debootstrap $ubuntu_codename /mnt

# Copy over the apt sources list
sudo cp /etc/apt/sources.list /mnt/etc/apt/sources.list

# Remove the cdrom source (first line)
sudo nano /mnt/etc/apt/sources.list

# Now mount the proc, sys, and dev directories
for dir in proc sys dev; do sudo mount --rbind /$dir /mnt/$dir && sudo mount --make-rslave /mnt/$dir; done

# Now copy the resolv.conf file
sudo cp /etc/resolv.conf /mnt/etc/resolv.conf

# Chroot into the system
sudo chroot /mnt /bin/bash

# Update apt
apt update

# Install the BTRFS drivers
apt install btrfs-progs

# Set the locale to en_US.UTF-8
dpkg-reconfigure locales

# Set the root password
passwd

# Install all essential packages including the desktop environment (KDE Plasma)
apt install linux-image-generic sudo nano kubuntu-desktop

# Copy the /proc/mounts file to /etc/fstab
cp /proc/mounts /etc/fstab

# List the UUIDs of the drives
blkid

# Replace the /dev/sdb3 and /dev/sdb1 with the UUID of the BTRFS drive (no quotes around the UUID)
# UUID=$uuid / btrfs ...

# And delete all exept the sdb3 and sdb1 entries

# Add the temp directory to the fstab
# tempfs /tmp tempfs rw,nosuid,nodev,inode64 0 0

# Add the swap partition to the fstab
# UUID=$uuid swap swap defaults 0 0
nano /etc/fstab

# Set the hostname
echo "dste-desktop" > /etc/hostname

# Replace localhost with dste-desktop in /etc/hosts
nano /etc/hosts

# Set the timezone
dpkg-reconfigure tzdata

# Set the keyboard layout (generic 105-key PC > Switzerland > German no dead keys and default for the rest)
dpkg-reconfigure keyboard-configuration

# Create the user
useradd -mG sudo dste
passwd dste

# Install the bootloader
apt install grub-efi-amd64
grub-install /dev/sdb
update-grub
```

### Add the other drives

```bash
lsblk

# Wipe the drives
sudo wipefs -a /dev/sdc
sudo wipefs -a /dev/sdd

# Add the drives to the BTRFS pool
sudo btrfs device add /dev/sdc /dev/sdd /

# Check the status
sudo btrfs filesystem show

# Balance the data
sudo btrfs balance start -dconvert=raid0 -mconvert=raid0 / --force

# Check the status
sudo btrfs filesystem show /
sudo btrfs filesystem df /
```

### Create a workspace subvolume

```bash
# Create the workspace subvolume
sudo btrfs subvolume create /@workspace

# Mount the workspace subvolume
sudo mkdir /workspace

# Change the permissions to allow all users to access the workspace
sudo chmod 777 /workspace

# Get the UUID of the workspace drive
sudo blkid | grep btrfs

# Add the workspace subvolume to the fstab
sudo nano /etc/fstab

# Add the following line
# UUID=$device_uuid /workspace     btrfs   subvol=/@workspace 0 0

# Restart the system to apply the changes
```

### Add my HDD as a backup drive for the snapshots

```bash
# Ceate the backup drive
sudo mkfs.btrfs -L backups /dev/sda -f

# Mount the drive
sudo mkdir /mnt/backups
sudo mount /dev/sda /mnt/backups

# Create the snapshots directory
sudo mkdir /mnt/backups/btrfs_snapshots
```

### Save a snapshot of the workspace on the backup drive

```bash
# Create a snapshot of the workspace
sudo btrfs subvolume snapshot -r /@workspace /@workspace_snapshot_$(date +%Y-%m-%d)

# Send the snapshot to the backup drive
sudo btrfs send /@workspace_snapshot_$(date +%Y-%m-%d) | sudo btrfs receive /mnt/backups/btrfs_snapshots

# List the snapshots
sudo btrfs subvolume list /mnt/backups/btrfs_snapshots

# Delete the snapshot
sudo btrfs subvolume delete /@workspace_snapshot_$(date +%Y-%m-%d)
```
