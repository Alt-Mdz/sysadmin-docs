# RHEL 10.1: SELinux Contexts

This SOP covers SELinux context inspection and management on Rocky Linux 10.1 and RHEL 10.1. Run commands as `root` or with `sudo` and validate labels before changing them in production.

> **Caution:** Context changes can break service access and application behavior. Prefer the system-managed label for a file or path whenever the service or policy expects it.

## 1. View current contexts

Inspect process contexts:

```bash
ps -eZ | head
ps -Z -C sshd
ps -Z -C httpd
```

Inspect file contexts:

```bash
ls -Zd /etc /var/www /srv
ls -Z /var/www/html
ls -Z /var/log/httpd
```

Inspect user contexts:

```bash
id -Z
```

## 2. Interpret a SELinux context

A typical file context:

```text
system_u:object_r:httpd_sys_content_t:s0
```

A typical process context:

```text
system_u:system_r:sshd_t:s0-s0:c0.c1023
```

The labels represent:

- `user` — SELinux identity such as `system_u`
- `role` — access role, often `object_r` or `system_r`
- `type` — the policy type such as `httpd_sys_content_t`
- `level` — sensitivity and category range

## 3. Compare expected vs actual contexts

Check the default context for a path using policy data:

```bash
matchpathcon /var/www/html
matchpathcon -V /var/www/html
```

Check the effective label on the filesystem and compare it to the policy default:

```bash
ls -Zd /var/www/html
matchpathcon /var/www/html
```

If content is copied from another host, the labels may not match the policy. Use `restorecon` to restore the expected file context.

## 4. Check ports and network contexts

Inspect port labels:

```bash
semanage port -l | grep -E 'http|https|ssh|mysql'
```

Check whether a service is bound to a permitted port type:

```bash
semanage port -l | grep http_port_t
ss -tulpn | grep :80
```

If a service needs to use a non-standard port, the port type may need to be added to policy.

## 5. Change contexts carefully

If a file must be moved or created with a specific context, verify it first:

```bash
sudo semanage fcontext -l | grep '/var/www'
```

Temporarily assign a label when troubleshooting:

```bash
sudo chcon -t httpd_sys_content_t /var/www/html/index.html
```

This is a useful test, but it is not a permanent fix. Use `semanage fcontext` for persistent labeling.

## 6. Restore the default context

For files that should follow policy defaults, restore the SELinux context:

```bash
sudo restorecon -Rv /var/www/html
sudo restorecon -Rv /srv
sudo restorecon -Rv /path/to/files
```

Use `-v` for more verbose output and `-F` if you need to force a relabeling operation.

## 7. Create persistent custom labeling rules

If a custom directory is in a standard service path, add a file context rule:

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/custom/app(/.*)?'
sudo restorecon -Rv /custom/app
```

To remove a custom rule:

```bash
sudo semanage fcontext -d '/custom/app(/.*)?'
sudo restorecon -Rv /custom/app
```

Verify the rule is recorded:

```bash
sudo semanage fcontext -l | grep '/custom/app'
```

## 8. Common context troubleshooting

### Web content is blocked by SELinux

```bash
ls -Zd /var/www/html
matchpathcon /var/www/html
sudo restorecon -Rv /var/www/html
journalctl -u httpd -b --no-pager
ausearch -m avc -ts recent
```

### Service cannot read a directory

```bash
ls -Zd /path/to/dir
sudo semanage fcontext -l | grep '/path/to'
sudo restorecon -Rv /path/to/dir
```

### Port denied for service

```bash
semanage port -l | grep http_port_t
sudo semanage port -a -t http_port_t -p tcp 8080
sudo semanage port -l | grep 8080
```

Use port changes only when the service is intended to bind to an alternate port and the policy allows it.

## 9. Best practices

- Use `ls -Z` and `matchpathcon` to compare actual and expected labels.
- Prefer `restorecon` and `semanage fcontext` over manual `chcon` changes.
- Validate the service after any labeling change.
- Review logs and `audit2why` before defining a permanent policy exception.

This SOP is intended to provide a practical workflow for SELinux context inspection and corrective action on Rocky Linux 10.1 systems.
