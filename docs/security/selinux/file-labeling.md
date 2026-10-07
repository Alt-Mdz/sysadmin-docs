# RHEL 10.1: SELinux File Labeling

This SOP covers regular file labeling tasks on Rocky Linux 10.1 and RHEL 10.1. File labels determine access control for files and directories. Correct labeling is a common fix when a service cannot read or write data even though the permissions look correct.

> **Caution:** Incorrect file contexts can break service access. Prefer the default policy label and restore it with `restorecon`, rather than manually assigning labels without confirming the expected type.

## 1. Inspect current labels

Check the SELinux label on files and directories:

```bash
ls -Zd /var/www /var/www/html /srv /opt /data
ls -Z /var/www/html
ls -Z /srv
```

Compare the current label to the policy default:

```bash
matchpathcon /var/www/html
matchpathcon /srv
```

## 2. Restore default labels

The primary tool for resetting labels to the expected policy values is `restorecon`:

```bash
sudo restorecon -Rv /var/www/html
sudo restorecon -Rv /srv
sudo restorecon -Rv /path/to/directory
```

Use `-v` for detailed output when troubleshooting:

```bash
sudo restorecon -Rv -v /var/www/html
```

## 3. Assign a label for testing

If a service needs a custom temporary label during troubleshooting, use `chcon` to test the hypothesis:

```bash
sudo chcon -t httpd_sys_content_t /var/www/html/index.html
ls -Z /var/www/html/index.html
```

This can help confirm whether a file type mismatch is causing the denial. It is not a permanent fix.

## 4. Make a label persistent

Persistent custom labels should be set using SELinux file context management:

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/custom/site(/.*)?'
sudo restorecon -Rv /custom/site
ls -Zd /custom/site
```

To remove a custom rule:

```bash
sudo semanage fcontext -d '/custom/site(/.*)?'
sudo restorecon -Rv /custom/site
```

## 5. Label shared content and network storage

When mounting or sharing data, confirm the files have the expected labels:

```bash
ls -Zd /data
ls -Zd /mnt/share
sudo restorecon -Rv /data
sudo restorecon -Rv /mnt/share
```

For NFS, CIFS, or other remote storage, the server may enforce its own labels. If the storage is not local to the host, verify the mount context and the service policy before changing labels.

## 6. Common labeling issues

### Web content type is wrong

```bash
ls -Zd /var/www/html
matchpathcon /var/www/html
sudo restorecon -Rv /var/www/html
```

### Application data cannot be read or written

```bash
ls -Zd /opt/app/data
sudo restorecon -Rv /opt/app/data
journalctl -u app.service -b --no-pager
ausearch -m avc -ts recent
```

### Files copied from another system retain incorrect labels

```bash
ls -Z /path/to/copied/files
sudo restorecon -Rv /path/to/copied/files
```

## 7. Verify the fix

After a label change, validate the service:

```bash
ls -Z /path/to/resource
sudo systemctl status service-name
sudo journalctl -u service-name -b -n 50 --no-pager
```

If the service still fails, inspect the AVC denials:

```bash
audit2why /var/log/audit/audit.log | tail -n 50
ausearch -m avc -ts recent
```

## 8. Best practices

- Prefer `restorecon` over manual labeling when the file is expected to use the default policy context.
- Use `semanage fcontext` for custom paths that are meant to keep a specific type.
- Document custom labels and file context rules in your change notes.
- Validate service behavior after any relabeling operation.

This SOP provides a standard workflow for labeling files on Rocky Linux 10.1 systems. Always confirm the expected file type before permanently adjusting labels.
