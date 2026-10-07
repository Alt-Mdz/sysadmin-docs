# RHEL 10.1: Users and Permissions Operations

This SOP provides a practical baseline for user account management and filesystem permission control on Rocky Linux 10.1 and RHEL 10.1 systems. Run commands as `root` or with `sudo`. Confirm the account, group, and target path before making changes that affect access control.

> **Caution:** Changes to users, groups, and permissions can grant or remove access to critical services or data. Validate the intended access model and test the result before applying changes in production.

## 1. Review the current system identity

Check the current user and effective permissions:

```bash
id
whoami
id username
getent passwd | head
getent group | head
```

Inspect the current account state:

```bash
sudo passwd -S username
sudo chage -l username
```

## 2. Create and manage local users

Create a new user with a home directory and default shell:

```bash
sudo useradd -m -s /bin/bash username
sudo passwd username
```

Create a user without a home directory when required:

```bash
sudo useradd -M -s /bin/bash username
```

Modify an existing user:

```bash
sudo usermod -aG wheel username
sudo usermod -s /bin/bash username
sudo usermod -d /home/newhome username
```

Delete a user when it is no longer needed:

```bash
sudo userdel -r username
```

## 3. Manage groups

Create a system or admin group:

```bash
sudo groupadd appadmins
sudo groupadd developers
```

Add a user to a group:

```bash
sudo usermod -aG appadmins username
```

Remove a user from a group:

```bash
sudo gpasswd -d username appadmins
```

List group membership:

```bash
id username
getent group appadmins
```

## 4. Understand Linux permissions

Inspect file and directory permissions:

```bash
ls -l /etc/passwd
ls -ld /srv/app
ls -l /var/www/html/index.html
```

Permission bits are represented as:

```text
-rwxr-xr--
```

Breakdown:

- `-` or `d` = file or directory
- `rwx` = owner permissions
- `r-x` = group permissions
- `r--` = others permissions

Common commands:

```bash
stat /path/to/file
namei -l /path/to/file
```

## 5. Change ownership

Set the owner and group on a file or directory:

```bash
sudo chown user:group /path/to/file
sudo chown -R user:group /path/to/directory
```

Set only the owner:

```bash
sudo chown username /path/to/file
```

Set only the group:

```bash
sudo chgrp appadmins /path/to/file
```

## 6. Change mode bits

Set read, write, and execute permissions for the owner:

```bash
chmod 700 /path/to/dir
```

Set a common shared directory permission for owner and group:

```bash
chmod 750 /path/to/shared-dir
```

Set read-only access for all users:

```bash
chmod 644 /path/to/file
```

Set executable permission on a script:

```bash
chmod 755 /path/to/script.sh
```

Use symbolic mode when it is easier to read:

```bash
chmod u+rwx,go-rwx /path/to/dir
chmod g+rw /path/to/file
chmod o-r /path/to/file
```

## 7. Set default permissions with umask

Check the current umask:

```bash
umask
```

Typical secure umask values:

```bash
umask 022
umask 027
```

Apply a default policy for new files and directories:

```bash
echo 'umask 027' >> ~/.bashrc
source ~/.bashrc
```

## 8. Use ACLs when standard groups are not enough

Check whether ACLs are enabled:

```bash
getfacl /path/to/file
setfacl -m u:username:rwx /path/to/file
setfacl -m g:appadmins:r-x /path/to/file
setfacl -m o::--- /path/to/file
```

Remove an ACL entry:

```bash
setfacl -x u:username /path/to/file
```

View effective access:

```bash
getfacl -p /path/to/file
```

ACLs are useful for shared data when a single group is not enough to model the access policy.

## 9. Sudo access and privilege escalation

Check whether a user can run root commands:

```bash
sudo -l -U username
```

Grant sudo rights through the wheel group:

```bash
sudo usermod -aG wheel username
```

Review sudoers configuration:

```bash
sudo visudo
sudo grep -n "wheel\|username" /etc/sudoers /etc/sudoers.d/* 2>/dev/null
```

## 10. Secure file ownership for service data

For application directories, set ownership to the relevant service user and group:

```bash
sudo chown -R apache:apache /var/www/html
sudo chmod -R 750 /var/www/html
```

For user-specific data:

```bash
sudo chown -R username:username /home/username
sudo chmod 700 /home/username
```

## 11. Troubleshooting permission problems

### User cannot read a file

```bash
ls -l /path/to/file
id username
getfacl /path/to/file
sudo setfacl -m u:username:r-- /path/to/file
```

### Shared directory is not writable

```bash
ls -ld /path/to/dir
getfacl /path/to/dir
sudo chown -R user:group /path/to/dir
sudo chmod 2775 /path/to/dir
```

### Service cannot write to its application directory

```bash
ls -ld /srv/app
sudo chown -R appuser:appgroup /srv/app
sudo chmod -R 750 /srv/app
sudo systemctl restart service-name
```

### User has the wrong supplemental groups

```bash
id username
sudo usermod -aG appadmins username
id username
```

## 12. Best practices

- Use the least privilege necessary for the user or service.
- Prefer group-based access over individual permissions when practical.
- Use ACLs only when the access model is more complex than standard owner/group/other permissions.
- Validate access after changing ownership or mode bits.
- Keep sudo privileges tightly controlled and auditable.

This SOP is intended as a practical reference for user and permission administration on Rocky Linux 10.1. Always confirm the target account, group membership, and filesystem path before changing access controls.
