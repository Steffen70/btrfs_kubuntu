## Install Kununtu on BTRFS with multiple drives

Install Kubuntu via installer and select BTRFS.

### Add the other drives

```bash
# Identify the drives
lsblk

# Wipe the drives
sudo wipefs -a /dev/sdc
sudo wipefs -a /dev/sdd

# Add the drives to the BTRFS pool
sudo btrfs device add /dev/sdc /dev/sdd /

# Check the status
sudo btrfs filesystem show

# Balance the data
sudo btrfs balance start -dconvert=raid0 -mconvert=raid1 /

# Check the status
sudo btrfs filesystem show /
sudo btrfs filesystem df /
```

### Create a workspce subvolume

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
