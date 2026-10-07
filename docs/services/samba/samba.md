# RHEL 10.1: Samba Server Operations

This SOP covers a standalone Samba file server on Rocky Linux 10.1 and RHEL 10.1. Run commands as `root` or with `sudo`. Replace example users, groups, paths, hostnames, and network ranges with values approved for your environment. Restrict SMB access to trusted networks and confirm a backup exists before changing production shares.

> **Caution:** Samba access requires both filesystem permissions and Samba authentication. A permissive setting in either layer can expose data; validate access from a client account after each change.

## 1. Install Samba

Install the server and administration tools:

```bash
sudo dnf install samba samba-client samba-common-tools
```

Check the installed version and available service:

```bash
smbd --version
systemctl status smb
```

## 2. Create a group and shared directory

Create a group for authorized share users, add an existing Linux account, and prepare the directory:

```bash
sudo groupadd smbshare
sudo usermod -aG smbshare alice
sudo mkdir -p /srv/samba/team
sudo chown root:smbshare /srv/samba/team
sudo chmod 2770 /srv/samba/team
```

The setgid bit (`2`) causes new files and directories to inherit the `smbshare` group. Users may need to sign out and back in before their new Linux group membership takes effect.

## 3. Add Samba credentials

Samba authenticates users separately from Linux login credentials. The Samba account must correspond to an existing Linux account:

```bash
sudo smbpasswd -a alice
sudo smbpasswd -e alice
sudo pdbedit -L
```

Use a unique Samba password. Do not put passwords in scripts or command-line arguments.

## 4. Configure the share

Back up the current configuration, then edit `/etc/samba/smb.conf`:

```bash
sudo cp -a /etc/samba/smb.conf /etc/samba/smb.conf.$(date +%Y%m%d%H%M%S).bak
sudo vi /etc/samba/smb.conf
```

Use or adapt this standalone server configuration. Preserve any required existing settings and share definitions:

```ini
[global]
    workgroup = WORKGROUP
    server role = standalone server
    security = user
    map to guest = never

[team]
    path = /srv/samba/team
    browseable = yes
    read only = no
    valid users = @smbshare
    force group = smbshare
    create mask = 0660
    directory mask = 2770
```

Validate syntax before applying the configuration:

```bash
sudo testparm -s
```

Resolve any reported errors before restarting the service.

## 5. Configure SELinux

Keep SELinux enforcing and label the share for Samba. Install the management utility if `semanage` is not present:

```bash
sudo dnf install policycoreutils-python-utils
sudo semanage fcontext -a -t samba_share_t '/srv/samba/team(/.*)?'
sudo restorecon -Rv /srv/samba/team
ls -Zd /srv/samba/team
```

If the file-context rule already exists, modify it instead of adding a duplicate:

```bash
sudo semanage fcontext -m -t samba_share_t '/srv/samba/team(/.*)?'
```

Do not disable SELinux to make a share work. Check denials with `sudo ausearch -m AVC -ts recent` if access still fails.

## 6. Open the firewall and start Samba

Inspect active firewall zones and the zone assigned to the server's client-facing interface:

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --get-default-zone
```

Allow the predefined Samba service only in the appropriate zone. Replace `public` if the interface uses another zone:

```bash
sudo firewall-cmd --permanent --zone=public --add-service=samba
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --query-service=samba
```

Start the SMB service and enable it at boot:

```bash
sudo systemctl enable --now smb
sudo systemctl status smb
sudo systemctl is-active smb
```

The `nmb` service is not required for normal SMB access by DNS name or IP. Enable it only when legacy NetBIOS name service is explicitly needed.

## 7. Verify from the server and a client

List the configured shares locally and check the listener:

```bash
smbclient -L localhost -U alice
sudo ss -lntp | grep -E ':(445|139)\b'
```

From a Linux client with `samba-client` installed, list and access the share:

```bash
smbclient //server.example.com/team -U alice
```

At the `smb: \\>` prompt, use `ls`, then test writing with `put` using a harmless test file. On Windows, open `\\server.example.com\team` and authenticate as `alice` with the Samba password.

Confirm that the created file has the expected owner, group, and mode on the server:

```bash
sudo ls -la /srv/samba/team
sudo stat /srv/samba/team
```

## 8. Manage users and permissions

Add another authorized user by creating or selecting a Linux account, adding it to the share group, and adding a Samba password:

```bash
sudo usermod -aG smbshare bob
sudo smbpasswd -a bob
```

Remove Samba access without deleting the Linux account:

```bash
sudo smbpasswd -d bob
```

Remove a Samba account entirely when appropriate:

```bash
sudo smbpasswd -x bob
```

After changing `smb.conf`, validate and reload the service:

```bash
sudo testparm -s
sudo systemctl reload smb
```

## 9. Troubleshoot common failures

### Share is not listed or the client cannot connect

```bash
sudo systemctl status smb
sudo testparm -s
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-services
sudo ss -lntp | grep -E ':(445|139)\b'
getent hosts server.example.com
```

Check name resolution, client routing, firewall zone, and TCP port 445 reachability. Use the server IP in a test to distinguish DNS problems from SMB problems.

### Authentication fails

```bash
sudo pdbedit -L
sudo smbpasswd -a alice
sudo smbpasswd -e alice
```

Confirm that the Linux account exists, the Samba account is enabled, and the client is using the intended username and Samba password.

### User can connect but cannot read or write

```bash
sudo testparm -s
sudo namei -l /srv/samba/team
sudo ls -ldZ /srv/samba/team
id alice
sudo ausearch -m AVC -ts recent
```

Check `valid users`, Linux group membership, directory ownership and mode, and the SELinux label. Confirm that the account has both Samba authorization and filesystem access.

### Review service logs

```bash
sudo journalctl -u smb --since '30 minutes ago'
sudo journalctl -t smbd --since '30 minutes ago'
```

## 10. Roll back a configuration change

Restore the backup made before editing, validate it, and restart Samba:

```bash
sudo cp -a /etc/samba/smb.conf.YYYYMMDDHHMMSS.bak /etc/samba/smb.conf
sudo testparm -s
sudo systemctl restart smb
```

Remove the firewall allowance only if no other Samba shares on the host need it:

```bash
sudo firewall-cmd --permanent --zone=public --remove-service=samba
sudo firewall-cmd --reload
```

Review the directory and SELinux changes separately before removing them; do not delete shared data as part of a service rollback.

## 11. Operational checklist

- Limit SMB access to trusted client networks and authorized users.
- Use both Samba account controls and restrictive Linux permissions.
- Keep SELinux enforcing and label shared paths with `samba_share_t`.
- Validate configuration with `testparm` before reload or restart.
- Verify access from an authorized client and confirm unauthorized accounts are denied.
- Back up `smb.conf` and shared data according to the host's recovery policy.
