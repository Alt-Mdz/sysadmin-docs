# Rocky Linux 10.1: Disk-Full Troubleshooting

This SOP covers filesystems that are full or close to full on Rocky Linux 10.1 and RHEL 10.1. Run commands as `root` or with `sudo`. If a production filesystem is at 100%, first identify the affected mount and the service impact; avoid broad deletion while the cause is unknown.

> **Caution:** Do not delete files under `/var/lib`, databases, container storage, or application data just because they are large. Confirm ownership and retention requirements before removing data. Never remove active log files by hand or run filesystem repair tools on a mounted filesystem.

## 1. Identify the affected filesystem

Check block capacity, inode capacity, mount points, and filesystem type:

```bash
df -hT
df -ih
findmnt
lsblk -f
```

Interpret the results:

- `Use%` near 100% in `df -hT` means the filesystem has little or no free space in blocks.
- `IUse%` near 100% in `df -ih` means the filesystem has run out of inodes, often due to many small files.
- Compare the affected mount point with `findmnt` and `lsblk -f` to identify its source device and filesystem.

If only one user or application receives `No space left on device`, check quotas and the exact target path as well:

```bash
quota -s <user>
sudo repquota -a
```

Quota tools or quota reporting may not be enabled on every system.

## 2. Find what is consuming space

Use `du` on the affected mount only. Replace `/var` with the actual mount point. The `-x` option prevents crossing into other mounted filesystems:

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
sudo du -xah /var 2>/dev/null | sort -h | tail -n 30
```

For a suspected directory, inspect the largest entries and their ownership before taking action:

```bash
sudo find /var/log -xdev -type f -size +100M -printf '%s %u:%g %p\n' 2>/dev/null | sort -n
sudo ls -lh /path/to/suspected/file
sudo stat /path/to/suspected/file
```

`du` reports space used by directory entries. It may not match `df` when files have been deleted but are still open, when filesystem metadata or snapshots consume space, or when a quota or reserved-space limit is involved.

## 3. Diagnose common causes

### Deleted files still use space

Check for unlinked files held open by a process:

```bash
sudo lsof +L1
```

If a large deleted file appears, identify the owning service and process. Restart or reload only that service using its supported procedure, then recheck `df -hT`. Do not kill an unknown process to reclaim space.

### Logs or the systemd journal are large

Check journal size and log directory usage:

```bash
sudo journalctl --disk-usage
sudo du -xhd1 /var/log 2>/dev/null | sort -h
sudo logrotate -d /etc/logrotate.conf
```

Review retention policy before vacuuming journal entries. If policy permits, remove older archived journal data by age or size:

```bash
sudo journalctl --vacuum-time=14d
sudo journalctl --vacuum-size=1G
```

These commands remove archived journal data; they do not fix a process that is rapidly writing logs. Inspect the responsible service and its log rotation configuration. Do not truncate or remove an active log file manually.

### Package cache is large

Inspect DNF cache usage and remove cached package data if it is not needed:

```bash
sudo du -sh /var/cache/dnf
sudo dnf clean packages
```

Cached packages can be downloaded again when needed.

### Many small files exhausted inodes

Find directories containing many files, starting at the affected mount:

```bash
sudo find /var -xdev -type f -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -n | tail -n 30
```

Check application caches, mail queues, temporary files, and spool directories. Identify the owning application and use its supported cleanup or retention controls; do not delete files based on path alone.

### Container or application data grew

Inspect usage with the tool that owns the data, for example:

```bash
sudo podman system df
sudo docker system df
sudo du -xhd1 /var/lib 2>/dev/null | sort -h
```

Remove only resources confirmed as unused and safe to discard. Avoid prune commands that remove images, volumes, or build cache without understanding whether workloads depend on them.

### LVM thin pool or snapshots are full

Check logical-volume and thin-pool usage:

```bash
sudo lvs -a -o lv_name,vg_name,lv_size,data_percent,metadata_percent,lv_attr
sudo vgs
```

A full thin pool can cause write failures even when a filesystem reports free capacity. Resolve the pool capacity or snapshot retention issue using the storage change procedure; do not remove snapshots until their recovery purpose is understood.

## 4. Free space safely

Prefer cleanup through the owning service or package manager. Before deleting anything, confirm the path, owner, last access or retention requirements, and whether the file is active:

```bash
sudo stat /path/to/file
sudo lsof /path/to/file
```

For system temporary files, use the configured systemd cleanup policy rather than deleting all of `/tmp`:

```bash
sudo systemd-tmpfiles --cat-config
sudo systemd-tmpfiles --clean
```

After each cleanup action, recheck the affected filesystem and confirm the service is healthy:

```bash
df -hT /mount/point
df -ih /mount/point
sudo systemctl --failed --no-pager
```

## 5. Expand a filesystem when cleanup is not enough

First verify the backing device, filesystem type, and available capacity. For LVM:

```bash
findmnt /mount/point
lsblk -f
sudo pvs
sudo vgs
sudo lvs -a -o +devices
```

If the volume group has free extents, extend the verified logical volume and filesystem together. Replace the example LV and size with values confirmed for the host:

```bash
sudo lvextend -r -L +5G /dev/vgdata/lvdata
```

If the VG has no free space, adding capacity may require expanding the virtual or physical disk and the partition/PV first. Follow the storage procedure in [LVM commands and troubleshooting](../storage/lvm.md) and the platform's storage runbook. Never run `pvcreate`, `mkfs`, or partitioning commands on a device until you have positively verified it is the intended empty device.

## 6. Validate recovery

After cleanup or expansion, confirm both block and inode availability and verify that affected services recovered:

```bash
df -hT
df -ih
sudo systemctl --failed --no-pager
sudo journalctl -b -p err --no-pager
```

Check the application's own health and confirm that it can write to the affected path. Monitor usage to see whether the filesystem is filling again.

## 7. Escalate when

Stop cleanup and investigate with the storage or application owner when:

- `df` reports full but `du` cannot account for the usage and `lsof +L1` shows no obvious deleted files.
- The device reports I/O errors, filesystem errors, or becomes read-only.
- A database, container volume, thin pool, snapshot, or production data directory is involved.
- There is no approved retention policy or safe data to remove.
- Space continues to disappear after cleanup.
