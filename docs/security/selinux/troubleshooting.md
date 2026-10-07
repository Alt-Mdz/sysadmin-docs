# RHEL 10.1: SELinux Troubleshooting

This SOP covers common SELinux troubleshooting workflows on Rocky Linux 10.1 and RHEL 10.1. Most SELinux issues appear as access denials, permission errors, or unexpectedly blocked service behavior.

> **Caution:** SELinux is meant to prevent unsafe access. Do not disable enforcing mode as a first response. Investigate the denial and correct the policy, labels, or boolean before changing the security state.

## 1. Confirm the current SELinux state

Check whether SELinux is enforcing and whether the system is in permissive mode:

```bash
getenforce
sestatus
cat /etc/selinux/config
```

If the system is in `Permissive` mode, the cause of the denial is still logged and can be reviewed.

## 2. Review the kernel and audit logs

Check the system log for SELinux messages:

```bash
journalctl -k -g SELinux --no-pager
journalctl -k -g avc --no-pager
```

Review the audit log for recent denials:

```bash
auditctl -l
ausearch -m avc -ts recent
ausearch -m avc --raw | tail -n 100
```

## 3. Convert denials into understandable explanations

Use `audit2why` and `audit2allow` to interpret the denial:

```bash
audit2why /var/log/audit/audit.log | tail -n 50
audit2allow -a
```

If `sealert` is installed, use it for a guided review:

```bash
sealert -a /var/log/audit/audit.log
```

## 4. Inspect the affected object and service context

Check the service process context:

```bash
ps -eZ | grep httpd
ps -eZ | grep nginx
ps -eZ | grep sshd
```

Check the file or directory label:

```bash
ls -Zd /var/www/html
ls -Zd /srv
ls -Zd /data
```

Compare the expected label with the policy default:

```bash
matchpathcon /var/www/html
matchpathcon /srv
```

## 5. Fix common denial patterns

### File label mismatch

```bash
ls -Zd /path/to/resource
sudo restorecon -Rv /path/to/resource
sudo systemctl restart service-name
```

### Incorrect port type

```bash
semanage port -l | grep -E 'http|https|ssh|mysql'
ss -tulpn | grep :80
sudo semanage port -a -t http_port_t -p tcp 8080
```

### Service needs a policy boolean

```bash
getsebool -a | grep service-related-keyword
sudo setsebool -P service_boolean on
sudo systemctl restart service-name
```

### Custom application path is blocked

```bash
sudo semanage fcontext -a -t httpd_sys_content_t '/custom/app(/.*)?'
sudo restorecon -Rv /custom/app
```

## 6. Temporarily place SELinux into permissive mode for diagnosis

If you need to verify that SELinux is the source of a problem, switch to permissive mode temporarily:

```bash
sudo setenforce 0
getenforce
```

After diagnosis, return to enforcing mode:

```bash
sudo setenforce 1
getenforce
```

Do not leave the system in permissive mode unless the change is explicitly required and documented.

## 7. Validate the fix

Use the following sequence after correcting a denial:

```bash
sudo restorecon -Rv /path/to/resource
sudo systemctl restart service-name
sudo journalctl -u service-name -b --no-pager
ausearch -m avc -ts recent
```

If the service still fails, check both the service logs and the SELinux denial log together.

## 8. Common recovery steps

If a service is completely blocked after a policy-related change:

```bash
sudo setenforce 0
sudo systemctl status service-name
sudo journalctl -u service-name -b --no-pager
ls -Zd /path/to/resource
matchpathcon /path/to/resource
```

This allows you to confirm whether the issue is caused by SELinux while the service remains available for debugging. Restore enforcing mode after the fix is verified.

## 9. Best practices

- Always inspect `audit2why` before modifying policy.
- Prefer `restorecon` and `semanage fcontext` over manual `chcon` assignment.
- Document boolean or port changes when they are required by service design.
- Validate the service after each corrective action.
- Keep SELinux in enforced mode unless a temporary corrective test is required.

This SOP is intended to help administrators identify and repair SELinux denials on Rocky Linux 10.1 with a consistent, repeatable workflow.
