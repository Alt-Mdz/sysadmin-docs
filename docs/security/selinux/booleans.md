# RHEL 10.1: SELinux Booleans

This SOP covers SELinux boolean management on Rocky Linux 10.1 and RHEL 10.1. Booleans are policy switches that permit or deny a set of behaviors without changing the full policy.

> **Caution:** SELinux booleans change access control behavior. Review the boolean purpose and service dependency before enabling or disabling it in production.

## 1. Inspect booleans

List booleans and filter the relevant ones:

```bash
getsebool -a | head
getsebool -a | grep httpd
getsebool -a | grep samba
getsebool -a | grep ftp
```

Check a specific boolean state:

```bash
getsebool httpd_enable_homedirs
getsebool samba_enable_home_dirs
```

## 2. Interpret boolean states

A boolean usually reports one of the following values:

```text
httpd_enable_homedirs --> off
samba_enable_home_dirs --> on
```

The `on` or `off` state indicates whether the policy allows the feature.

## 3. Change a boolean temporarily

Use `setsebool` to change a boolean in the current runtime:

```bash
sudo setsebool httpd_enable_homedirs on
getsebool httpd_enable_homedirs
```

This change is not persistent unless you use `-P`.

## 4. Change a boolean persistently

Make a boolean change persist across reboots:

```bash
sudo setsebool -P httpd_enable_homedirs on
getsebool httpd_enable_homedirs
```

Verify the service works with the new policy state and review the logs if the service still fails.

## 5. Common service booleans

Examples of relevant booleans include:

```bash
getsebool -a | grep httpd
getsebool -a | grep samba
getsebool -a | grep ftp
getsebool -a | grep nfs
```

Common use cases:

- allow a web server to access home directories
- permit Samba to share user home directories
- allow an FTP server to read user content
- enable a service to connect to network shares or custom paths

## 6. Check which booleans affect a service

Use `semanage boolean -l` for a more structured listing:

```bash
sudo semanage boolean -l | grep -i httpd
sudo semanage boolean -l | grep -i samba
```

This helps identify which booleans govern a service or use case.

## 7. Troubleshooting with booleans

If a service is denied access to content or a custom path, inspect the policy and service logs:

```bash
ausearch -m avc -ts recent
audit2why /var/log/audit/audit.log | tail -n 50
getsebool -a | grep service-name
```

If the denial is due to a known policy toggle, set the boolean and test again:

```bash
sudo setsebool -P httpd_enable_homedirs on
sudo systemctl restart httpd
sudo journalctl -u httpd -b -n 50 --no-pager
```

## 8. Safe boolean rollback

If a boolean creates unexpected behavior, revert it:

```bash
sudo setsebool -P httpd_enable_homedirs off
getsebool httpd_enable_homedirs
```

Check service health and confirm the denial is resolved before leaving the system in the reverted state.

## 9. Best practices

- Read the purpose of a boolean before changing it.
- Prefer `-P` only when the behavior must remain after reboot.
- Verify the service after changing the boolean.
- Use `audit2why` and `ausearch` to confirm the root cause.
- Document the boolean change in your change record or runbook.

This SOP is intended for administrators performing targeted SELinux policy adjustments on Rocky Linux 10.1 systems. Always validate the effect on the application or service before leaving the change in place.
