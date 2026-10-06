# RHEL 10.1: LVM Commands and Troubleshooting

This SOP covers common LVM administration on Rocky Linux 10.1. & RHEL 10.1 Run storage-changing commands as `root` or with `sudo`. Replace example device names, volume-group names, logical-volume names, and sizes with values verified on the target host.

> **Caution:** LVM and filesystem commands can destroy data. Confirm the target device with `lsblk`, `pvs`, `vgs`, and `lvs` before making changes. Back up important data and test procedures before using them in production. Never run `pvcreate`, `mkfs`, or a partitioning command on a device that may contain needed data.

## 1. Basic system and storage checks

```bash
cat /etc/rocky-release
uname -r
lsblk -f
sudo pvs
sudo vgs
sudo lvs -a -o +devices
df -hT
```

Install LVM utilities if the commands are unavailable:

```bash
sudo dnf install lvm2
```

Use `lsblk -f` to identify disks, partitions, filesystems, and mount points. LVM commands report physical volumes (PVs), volume groups (VGs), and logical volumes (LVs); they do not replace filesystem checks such as `df -hT`.

## 2. Common inspection commands

```bash
sudo pvs -o pv_name,vg_name,pv_size,pv_free,pv_attr
sudo vgs -o vg_name,vg_size,vg_free,vg_attr
sudo lvs -a -o lv_name,vg_name,lv_size,lv_attr,devices
sudo pvdisplay
sudo vgdisplay
sudo lvdisplay
sudo lvs --segments
```

Check which device backs a mounted filesystem:

```bash
findmnt /mount/point
lsblk -f
sudo lvs -o lv_path,vg_name,lv_name,devices
```

## 3. Create a basic LVM volume

The example uses an **empty** `/dev/sdb` device, creates a VG named `vgdata`, and creates a 20 GiB LV named `lvdata`.

1. Confirm the device is correct and has no data to preserve, you can work on an unused partition on the same disk without unmounting it.:

   ```bash
   lsblk -f
   sudo fdisk /dev/sdb
   ```

2. Initialize the device and create the VG and LV, It is recommended to specify physical extensions:

   ```bash
   sudo pvcreate /dev/sdb
   sudo pvdisplay
   sudo vgcreate -s 4 vgdata /dev/sdb
   sudo vgdisplay
   sudo lvcreate -l  +100%FREE -n lvdata vgdata
   sudo lvdisplay
   ```

3. Create a filesystem and mount it. This `mkfs` command **erases existing filesystem contents** on the LV:

   ```bash
   sudo mkfs.xfs /dev/mapper/vgdata-lvdata
   sudo mkdir -p /data
   sudo mount /dev/mapper/vgdata-lvdata /data
   df -hT /data
   ```

4. For a persistent mount, obtain the filesystem UUID and add an entry to `/etc/fstab`:

   ```bash
   sudo blkid /dev/mapper/vgdata-lvdata
   ```

   Example entry (replace the UUID with the actual value):

   ```fstab
   UUID=<filesystem-uuid>  /data  xfs  defaults,nofail  0  0
   ```

   Verify the edited file before rebooting:

   ```bash
   sudo findmnt --verify --verbose
   sudo mount -a
   findmnt /data
   df -Ht
   ```

## 4. Extend an LV and filesystem

Check free space first:

```bash
sudo vgs
sudo lvs
df -hT /mount/point
```

To add 5 GiB from free space in the VG and grow the filesystem online:

```bash
sudo lvextend -L +5G -r /dev/vgdata/lvdata
```

`-r` asks LVM to resize the filesystem along with the LV. Confirm the result:

```bash
sudo lvs /dev/vgdata/lvdata
df -hT /mount/point
```

If the VG has no free extents, first confirm and add a new, empty disk or partition. For example, after verifying `/dev/sdc` is the intended device:

```bash
sudo pvcreate /dev/sdc
sudo vgextend vgdata /dev/sdc
sudo vgs
sudo lvextend -L +5G -r /dev/vgdata/lvdata
```

It is recommended to use empty disks.

## 5. Remove a volume safely

Before removing an LV, verify that it is the intended volume, determine whether it is mounted or used by an application, and make a backup if needed:

```bash
sudo lvs -a -o +devices
findmnt
```

Stop dependent applications, unmount the filesystem, remove its `/etc/fstab` entry, then remove the LV only after confirming that its contents are no longer needed:

```bash
sudo umount /mount/point
sudo lvremove /dev/mapper/vgdata-lvdata
```

Do not remove a VG or PV until all dependent LVs and data have been accounted for. `vgremove` and `pvremove` are destructive operations.

## 6. Troubleshooting

### LVM commands are missing

Install the package and retry:

```bash
sudo dnf install lvm2
```

If package installation fails, check repository and network access:

```bash
sudo dnf repolist
sudo dnf makecache
```

### A PV, VG, or LV is not visible

Check the device and scan for LVM metadata:

```bash
lsblk -f
sudo pvs -a
sudo vgs -a
sudo lvs -a
sudo pvscan
sudo vgscan
```

Confirm that the expected disk is attached and visible to the operating system. Do not initialize it with `pvcreate` as a troubleshooting step; that can overwrite metadata.

### A VG or LV is inactive

Inspect the reported state, then activate the specific VG if it is expected to be in use:

```bash
sudo vgdisplay vgdata
sudo lvscan
sudo vgchange -ay vgdata
sudo lvs -a -o +devices
```

If activation reports missing devices or inconsistent metadata, stop and investigate the underlying storage and logs rather than forcing activation.

### The filesystem is full but the LV has free space

Compare filesystem capacity with LV capacity:

```bash
df -hT /mount/point
sudo lvs
```

If the LV is larger than the filesystem, grow the filesystem. Prefer the combined command where supported:

```bash
sudo lvextend -r -L +5G /dev/vgdata/lvdata
```

For XFS, the mounted filesystem can be grown with:

```bash
sudo xfs_growfs /mount/point
```

For ext4, grow the filesystem with:

```bash
sudo resize2fs /dev/vgdata/lvdata
```

Check the filesystem type with `df -T` or `lsblk -f` before choosing a filesystem-specific command.

### The VG has no free space

Check current allocation and attached PVs:

```bash
sudo vgs
sudo pvs
sudo lvs -a -o +devices
```

If approved, add a verified empty disk as a PV and extend the VG (see section 4). Do not remove or repurpose a PV just to create space without planning data evacuation.

### Mount fails after reboot

Check the mount definition and filesystem identity:

```bash
sudo findmnt --verify --verbose
sudo blkid
sudo lvs
sudo journalctl -b -p warning
```

Prefer a filesystem UUID in `/etc/fstab`; correct any stale UUID, device path, filesystem type, or mount options, then test with `sudo mount -a` before rebooting.

### A disk or PV is missing

Check kernel and system logs and verify the device is presented to the host:

```bash
lsblk
sudo pvs -a
sudo journalctl -k -b
```

If a VG depends on the missing PV, some LVs may be unavailable or degraded. Restore the underlying device or storage path and follow an approved recovery plan. Avoid `vgreduce --removemissing`, `pvremove`, or force options until the missing data and metadata situation is understood; these can make recovery harder or cause data loss.

### LVM-related logs

```bash
sudo journalctl -b | grep -iE 'lvm|device-mapper'
sudo dmesg -T | grep -iE 'lvm|device-mapper|I/O error'
```

## 7. XFS and LV reduction warning

XFS filesystems cannot be shrunk. Do **not** run `lvreduce` on an LV containing XFS data to try to reduce filesystem size; it can truncate the filesystem and destroy data. If less capacity is needed, plan a backup-and-restore or migration to a newly created, correctly sized LV.

For other filesystems, reduction still requires filesystem-specific checks and an offline procedure. Do not reduce an LV until the filesystem has been safely reduced and the exact target size has been verified.
