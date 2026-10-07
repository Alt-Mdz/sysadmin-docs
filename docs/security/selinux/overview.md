# RHEL 10.1: SELinux Overview and Operational Baseline

This SOP provides a practical baseline for SELinux administration on Rocky Linux 10.1 and RHEL 10.1 systems. Run commands as `root` or with `sudo`. Always validate the effect of policy changes before applying them to production systems.

> **Caution:** SELinux is a mandatory access control system. Incorrect file labels, booleans, or policy changes can block services and applications. Do not disable SELinux as a routine fix unless you are performing a controlled, documented recovery action.

## 1. Confirm SELinux state

Check whether SELinux is enabled and in which mode it is operating:

```bash
getenforce
sestatus
cat /etc/selinux/config
```

Example output: `Enforcing`, `Permissive`, or `Disabled`.

Inspect the current policy:

```bash
seinfo -t
semanage boolean -l | head
```

## 2. Understand object labels

SELinux labels are attached to files, processes, ports, and other objects. They determine whether a subject can access an object.

Inspect process labels:

```bash
ps -eZ | head
ps -Z -C sshd
```

Inspect file labels:

```bash
ls -Zd /etc /var/www /srv
ls -Z /var/log/httpd
```

Typical labels look like this:

```text
system_u:object_r:httpd_sys_content_t:s0
```

The fields are:

- `user`
- `role`
- `type`
- `level` (MLS / MCS sensitivity)

## 3. Common SELinux commands

Use these commands for routine inspection:

```bash
id -Z
ls -Zd /var/www/html
ls -Zd /home/user
semanage fcontext -l | head
semanage user -l
getsebool -a | grep httpd
```

## 4. Inspect logs and denials

Review SELinux denial messages from the kernel and audit log:

```bash
journalctl -k -g SELinux
ausearch -m avc -ts recent
ausearch -m avc --raw | tail -n 50
```

Common tools for understanding denials:

```bash
audit2why /var/log/audit/audit.log | tail -n 50
audit2allow -a
sealert -a /var/log/audit/audit.log
```

## 5. Minimal safe workflow

When troubleshooting a SELinux problem, use the following sequence:

```bash
ls -Zd /path/to/resource
ps -eZ | grep service-name
ausearch -m avc -ts recent
audit2why /var/log/audit/audit.log | tail -n 50
restorecon -Rv /path/to/resource
```

This workflow confirms the labels, checks the process context, reviews the denial, and applies the minimal valid corrective action.

## 6. Best practices

- Preserve SELinux labels when moving or restoring files.
- Prefer `restorecon` over broad `chcon` changes when the file context is expected to be standard.
- Use `setsebool -P` only when you understand the policy impact.
- Validate service health and logs after each change.
- Use `audit2why` to understand why a denial occurred before modifying policy.

## 7. Recovery guidance

If the system is in `Permissive` mode and a policy issue is under investigation:

```bash
sudo setenforce 0
sudo getenforce
```

Do not leave SELinux disabled permanently. Return to enforcing mode after diagnosis:

```bash
sudo setenforce 1
sudo getenforce
```

This SOP is intended to provide a practical starting point for SELinux operations on Rocky Linux 10.1. Always confirm the current policy state, object labels, and audit denials before changing configuration or access control.
