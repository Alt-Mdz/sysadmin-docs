# RHEL 10.1: NFS Server and Client Operations

This SOP provides a practical reference for configuring and troubleshooting NFS on Rocky Linux 10.1 and RHEL 10.1 systems. Run commands as `root` or with `sudo`. Validate the export path, client access, and firewall settings before changing an NFS service in production.

> **Caution:** NFS exposes shared filesystems to remote systems. Review export permissions, security options, and client trust before making changes to production exports.

## 1. Install and verify NFS services

Install the NFS packages on the server and client:

```bash
sudo dnf install nfs-utils
```

Check the NFS service state:

```bash
sudo systemctl status nfs-server
sudo systemctl enable --now nfs-server
sudo systemctl status nfs-client.target
```

Verify the service is active:

```bash
sudo systemctl is-active nfs-server
```

## 2. Prepare a shared directory

Create and secure the exported directory:

```bash
sudo mkdir -p /srv/share
sudo chown nobody:nobody /srv/share
sudo chmod 755 /srv/share
```

For application data with specific ownership, set the owner accordingly:

```bash
sudo chown -R appuser:appgroup /srv/share
```

## 3. Configure exports on the server

Edit the exports file:

```bash
sudo vi /etc/exports
```

Example exports entries:

```text
/srv/share 192.168.10.0/24(rw,sync,no_subtree_check,no_root_squash)
/srv/share 10.0.0.10(rw,sync,no_subtree_check)
```

Common options:

- `rw` — read-write access
- `ro` — read-only access
- `sync` — write to disk before replying
- `async` — asynchronous writes
- `no_subtree_check` — reduce metadata checks
- `root_squash` — map root to anonymous user
- `no_root_squash` — allow root on the client to act as root on the server
- `anonuid` / `anongid` — map anonymous users to a UID/GID

Apply the export configuration:

```bash
sudo exportfs -rav
sudo exportfs -s
```

Check the active exports:

```bash
sudo exportfs -v
```

## 4. Start the NFS service and confirm exports

Reload the export table and ensure the service is running:

```bash
sudo systemctl restart nfs-server
sudo systemctl status nfs-server
sudo exportfs -v
```

Test the server from the local host:

```bash
showmount -e localhost
```

## 5. Configure the NFS client

Create a mount point:

```bash
sudo mkdir -p /mnt/share
```

Mount the exported filesystem manually:

```bash
sudo mount -t nfs 192.168.10.10:/srv/share /mnt/share
```

Check the mount:

```bash
df -h /mnt/share
mount | grep nfs
findmnt /mnt/share
```

## 6. Configure persistent mounts in /etc/fstab

Add a persistent NFS mount:

```fstab
192.168.10.10:/srv/share   /mnt/share   nfs   defaults,_netdev   0  0
```

Test the entry without rebooting:

```bash
sudo mount -a
findmnt /mnt/share
```

## 7. Check client access and permissions

Inspect mount options and effective ownership:

```bash
mount | grep nfs
stat /mnt/share
ls -ld /mnt/share
```

If the mounted filesystem is not writable, verify the server export, client mount options, and local user permissions.

## 8. Troubleshooting NFS issues

### The client cannot see the exported share

```bash
showmount -e 192.168.10.10
rpcinfo -p 192.168.10.10
sudo firewall-cmd --state
sudo firewall-cmd --list-all
```

Check whether the required NFS ports are open in the firewall and that the server is running.

### Mount command fails with `access denied`

```bash
showmount -e 192.168.10.10
sudo exportfs -v
sudo cat /etc/exports
```

Review the client address, export path, and options like `ro`, `rw`, and `root_squash`.

### Remote user cannot write to the share

```bash
ls -ld /srv/share
sudo exportfs -v
id clientuser
```

Confirm that the server export permissions and the local directory ownership match the intended access model.

### Permission denied after mount

```bash
mount | grep nfs
ls -ld /mnt/share
getfacl /mnt/share 2>/dev/null
sudo mount -o remount,rw 192.168.10.10:/srv/share /mnt/share
```

Check whether the server export is read-only or whether the client local filesystem permissions are restrictive.

## 9. Firewall and service checks

If NFS traffic is blocked, review the required ports and firewall policy:

```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

Verify the service and port mapping:

```bash
sudo ss -tulpn | grep :111
sudo ss -tulpn | grep :2049
sudo ss -tulpn | grep :20048
```

## 10. NFSv4 and mount behavior

NFSv4 commonly uses fewer separate port dependencies. Check the mount type and server version:

```bash
mount -t nfs4 192.168.10.10:/srv/share /mnt/share
nfsstat -m
```

If the host has a stale mount, unmount and remount it:

```bash
sudo umount /mnt/share
sudo mount -a
```

## 11. Best practices

- Use an explicit client network or host entry in `/etc/exports`.
- Prefer `rw,sync` for standard shared data and limit access with `root_squash` when appropriate.
- Validate the firewall before enabling NFS on a production host.
- Check `showmount` and `exportfs` before diagnosing client mount issues.
- Test the mount after each export change and document the client list for the exported share.

This SOP provides a practical operational workflow for NFS setup and troubleshooting on Rocky Linux 10.1. Always validate the server export, firewall policy, and client mount before making production changes.
