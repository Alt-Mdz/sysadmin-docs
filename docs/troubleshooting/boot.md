# Rocky Linux 10.1: Boot Troubleshooting

This SOP provides a safe workflow for diagnosing boot failures on Rocky Linux 10.1 and RHEL 10.1. Use a VM console, hypervisor console, or physical console when remote access is unavailable. Record the last successful boot, recent changes, exact error text, and the stage where boot stops before changing the system.

> **Caution:** Boot and filesystem repairs can make a system or its data unavailable. Confirm backups and storage health first. Do not run filesystem repair tools against a mounted filesystem, and do not reinstall GRUB using commands copied from a different BIOS/UEFI layout.

## 1. Identify the boot stage

Observe where the failure occurs:

- **No boot device or firmware error:** Check firmware boot order, disk visibility, controller mode, and virtual disk attachment.
- **GRUB menu or prompt:** Check GRUB configuration, boot entries, and `/boot` availability.
- **Kernel or initramfs error:** Try a previous kernel and inspect storage, filesystem, and initramfs messages.
- **Emergency or rescue shell:** Inspect the boot journal, failed units, `/etc/fstab`, and root filesystem state.
- **Login prompt or graphical login appears:** The OS booted; troubleshoot the specific service, display manager, or login issue rather than the bootloader.

If this is a virtual machine, confirm the disk and its controller are attached and the VM is configured to boot from the expected device before changing the guest OS.

## 2. Collect evidence when a shell is available

From an emergency shell, rescue environment, or successful boot, collect the current boot's state:

```bash
cat /etc/os-release
uname -r
systemctl --failed --no-pager
journalctl -b -p warning..alert --no-pager
journalctl -b -u systemd-fsck-root.service --no-pager
journalctl -b -u local-fs.target --no-pager
findmnt --verify --verbose
lsblk -f
```

If the current boot failed but the machine was rebooted successfully, inspect the previous boot if its journal was persisted:

```bash
sudo journalctl --list-boots
sudo journalctl -b -1 -p warning..alert --no-pager
```

Check whether the journal is persistent under `/var/log/journal`. If it was not persistent, previous-boot logs may not be available.

## 3. System stops at the GRUB menu

### Try a previous installed kernel

At the GRUB menu, select **Advanced options for Rocky Linux** and choose an older installed kernel. If it boots, collect evidence before changing packages:

```bash
uname -r
sudo grubby --info=ALL
sudo grubby --default-kernel
sudo journalctl -b -p warning..alert --no-pager
sudo df -h /boot /boot/efi 2>/dev/null
```

A successful boot with an older kernel points toward a problem with the newer kernel, its initramfs, or a kernel module. Keep the known-good kernel installed while investigating.

### GRUB menu is missing or drops to a GRUB prompt

From firmware setup, confirm that the expected disk is present and first in the boot order. If the disk is present but GRUB is unavailable, use Rocky/RHEL installation media in rescue mode and identify the actual boot mode and partitions before attempting bootloader repair:

```bash
lsblk -f
sudo efibootmgr -v
```

UEFI and legacy BIOS use different boot layouts and repair procedures. Do not run `grub2-install` or overwrite EFI files until the installed system's boot mode, EFI System Partition, disk, and recovery method are confirmed. For production systems, use the platform-specific vendor recovery procedure or escalate with the collected layout and console output.

## 4. Kernel panic or initramfs failure

At the GRUB menu, try an older kernel first. If an older kernel starts, inspect the failed kernel's files and available space:

```bash
sudo grubby --info=ALL
sudo ls -lh /boot
sudo df -h /boot /boot/efi 2>/dev/null
```

Rebuild an initramfs only after confirming the exact installed kernel version and that `/boot` has sufficient free space. For example, from a working boot, replace `<kernel-version>` with the exact version shown by `rpm -q kernel-core`:

```bash
rpm -q kernel-core
sudo dracut --force --kver <kernel-version>
```

Do not use `uname -r` for a different kernel than the one being repaired. Review `dracut` output, then reboot and select the repaired kernel. If storage drivers, encrypted volumes, or third-party modules are involved, verify their configuration before rebuilding.

## 5. System enters emergency mode

Emergency mode often indicates a required mount or filesystem could not be activated. Start with the exact failure message and boot journal:

```bash
journalctl -xb --no-pager
systemctl --failed --no-pager
findmnt --verify --verbose
lsblk -f
```

### Check `/etc/fstab` mount failures

Compare each entry in `/etc/fstab` with the actual UUIDs and filesystems:

```bash
cat /etc/fstab
blkid
lsblk -f
findmnt --verify --verbose
```

For a confirmed stale or unavailable non-root mount, back up `/etc/fstab` and temporarily comment out only the failing entry. Do not remove or change the root (`/`) or boot (`/boot`, `/boot/efi`) entries without a verified recovery plan.

```bash
cp -a /etc/fstab /etc/fstab.$(date +%Y%m%d%H%M%S).bak
vi /etc/fstab
mount -a
```

If the mount is optional and should not block boot when unavailable, consider the `nofail` option only after confirming that dependent applications tolerate the mount being absent. Correct the underlying device, UUID, network mount, or mount options, then verify with `findmnt --verify --verbose` and `mount -a` before rebooting.

### Check a failed service or target dependency

If the journal identifies a specific unit, inspect that unit rather than disabling unrelated services:

```bash
systemctl status <unit-name> --no-pager
journalctl -b -u <unit-name> --no-pager
systemctl cat <unit-name>
```

Correct the reported configuration, dependency, or permissions issue. Use `systemctl reset-failed <unit-name>` only after addressing the cause.

## 6. Root filesystem or storage problems

Inspect block devices, filesystem types, mount state, and available capacity:

```bash
lsblk -f
findmnt /
df -hT
sudo journalctl -b -k --no-pager | grep -Ei 'I/O error|filesystem|xfs|ext4|nvme|scsi|timeout'
```

If the root filesystem is mounted read-only, do not force it read-write until logs and storage health have been reviewed. A remount may be a temporary diagnostic step only when there is no evidence of filesystem or device errors:

```bash
sudo mount -o remount,rw /
```

Do not run `fsck` or `xfs_repair` on a mounted filesystem. XFS is not repaired with `fsck`; use the filesystem-specific recovery procedure from rescue media with the filesystem unmounted. If the device reports I/O errors, stop write-heavy repair attempts and investigate the underlying disk, storage controller, or virtual storage first.

## 7. Boot completes but login or graphical startup fails

Check the default target and failed units:

```bash
systemctl get-default
systemctl --failed --no-pager
journalctl -b -p warning..alert --no-pager
```

For a graphical login failure, inspect the display manager and its logs (commonly `gdm`):

```bash
systemctl status gdm --no-pager
journalctl -b -u gdm --no-pager
```

If console access is available, switch temporarily to the text-mode target to isolate a graphical startup problem:

```bash
sudo systemctl isolate multi-user.target
```

Do not change the default target unless the intended operating mode is known. If the system is expected to provide a graphical login, check that the display manager is installed, enabled, and healthy.

## 8. Verify recovery

Before returning the host to service:

```bash
sudo systemctl --failed --no-pager
sudo journalctl -b -p warning..alert --no-pager
findmnt --verify --verbose
systemctl get-default
```

Reboot during an approved maintenance window and verify that the system reaches its intended target, all required filesystems mount, critical services start, and application data is accessible. Keep the previous working kernel or rescue path available until the new boot is confirmed.

## 9. Escalation checklist

When the issue persists, capture:

- Exact console error and the stage where boot stops
- Whether the system boots with an older kernel
- Recent kernel, storage, firmware, or `/etc/fstab` changes
- `lsblk -f`, boot mode, and partition layout
- `journalctl -b` output or previous-boot journal, if available
- Any storage-controller or disk I/O errors
- Recovery actions already attempted
