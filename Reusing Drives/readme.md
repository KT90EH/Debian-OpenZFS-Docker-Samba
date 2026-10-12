Backup your data first...

TrueNAS Scale has GUI wrappers for most of this, plus shell access if you need it. Here’s the flow:

Via the Web UI (recommended — safest)

1. Export/destroy the old pool (if it still exists as a pool)

Storage → find the pool → click the three-dot menu → Export/Disconnect
Check “Destroy data on this pool” and confirm the disks it’s using (TrueNAS lists serials, so you can visually verify)
This handles zpool destroy cleanly and removes it from TrueNAS’s pool config/database

2. Wipe the disk

Go to Storage → Disks (or System → Disks depending on version)
Select the target disk (again, verify by serial number shown in the list)
Click Wipe
Choose wipe mode:
Quick – clears partition table + ZFS labels (fast, usually sufficient)
Full with zeros – overwrites entire disk with zeros
Full with random data – slower, for sensitive data disposal
For NVMe reuse in a new pool, Quick is normally enough since you’re not disposing of the drive


Before wiping, hover/check the disk’s serial in the UI — TrueNAS shows it directly in Storage → Disks, so you don’t have to hunt through /dev/disk/by-id/ yourself like on bare Linux. Cross-reference against the pool’s old config (Storage → pool → Status shows serials per vdev member) to be sure.
