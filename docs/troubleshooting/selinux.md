# Rocky Linux 10.1: Troubleshooting SELinux Denials

This SOP covers a repeatable workflow for diagnosing SELinux access denials on Rocky Linux 10.1 and RHEL 10.1. Use it when an application reports `Permission denied` or a service fails and ordinary ownership and mode checks do not explain the failure. Keep SELinux enforcing while investigating; an AVC denial is evidence to interpret, not an instruction to grant access automatically.

> **Caution:** Do not disable SELinux or install policy generated from all recent audit events as a first response. Broad or unreviewed rules can grant unintended access. Change one policy setting at a time and verify the application behavior and audit logs.

## 1. Confirm SELinux state and reproduce the problem

Record the failure time, affected service, user, operation, and resource path. Check SELinux and the service state:

```bash
getenforce
sestatus
sudo systemctl status service-name --no-pager -l
```

Keep the system in `Enforcing` mode during normal diagnosis. If the issue is intermittent, reproduce it once and note the exact time so the corresponding audit event can be isolated.

## 2. Find and interpret the AVC denial

Search recent audit records for access vector cache (AVC) denials:

```bash
sudo ausearch -m AVC,USER_AVC -ts recent -i
sudo journalctl -k --since '30 minutes ago' --no-pager | grep -i 'avc:.*denied'
```

If `ausearch` reports that audit data is unavailable, confirm that `auditd` is running and check whether the audit log exists:

```bash
sudo systemctl status auditd --no-pager
sudo ls -l /var/log/audit/audit.log
```

For a guided explanation, if `audit2why` is installed, provide it the matching raw audit events:

```bash
sudo ausearch -m AVC,USER_AVC -ts recent --raw | audit2why
```

If `setroubleshoot-server` is installed, `sealert` can provide additional context:

```bash
sudo sealert -a /var/log/audit/audit.log
```

Focus on the denied operation, source context (`scontext`), target context (`tcontext`), target type, and path or port. Confirm the event coincides with the failing operation. Do not treat an unrelated or old AVC as proof of the current failure.

## 3. Check ordinary access and SELinux contexts

Inspect the service process context, path labels, and ordinary Unix permissions:

```bash
sudo ps -eZ | grep -F service-process
sudo namei -l /path/to/resource
sudo ls -ldZ /path/to/resource
sudo matchpathcon -V /path/to/resource
```

Check the expected default label for the path. If a file or directory has an incorrect label, restore the policy default:

```bash
sudo restorecon -Rv /path/to/resource
```

`restorecon` applies the default label for the path; it does not make arbitrary application paths valid for a service. If the service intentionally uses a custom path, configure a persistent file-context mapping rather than relying on a temporary `chcon` label.

## 4. Fix common denial types

### Service data is stored in a custom path

First identify the SELinux type expected by the service. Use an existing policy-defined path or consult the service's SELinux documentation. Then add a persistent mapping and apply it. Example only, using an appropriate type for the specific service:

```bash
sudo semanage fcontext -a -t service_data_t '/srv/service-data(/.*)?'
sudo restorecon -Rv /srv/service-data
```

If a local mapping already exists, modify it with `semanage fcontext -m` rather than adding a duplicate. Replace `service_data_t` with the verified SELinux type; do not copy this placeholder literally.

### A supported service behavior needs a policy boolean

Inspect relevant booleans and their descriptions:

```bash
sudo semanage boolean -l -C
getsebool -a | grep -i service-keyword
```

Enable a boolean only if its documented purpose matches the required behavior. Persist the setting with `-P` when it is an approved configuration change:

```bash
sudo setsebool -P boolean_name on
```

Record the previous value so it can be restored if needed. Avoid setting unrelated booleans simply because they appear in an online workaround.

### Service listens on a nonstandard port

Check the port's current SELinux type and the service's supported port types:

```bash
sudo semanage port -l -C
sudo semanage port -l | grep -E 'service_port_type|<port>'
sudo ss -lntup
```

If the port should be handled by a specific SELinux type and is not already assigned elsewhere, add the mapping with the verified type and protocol:

```bash
sudo semanage port -a -t service_port_type -p tcp 8443
```

If the port already has a local mapping, modify that mapping only after confirming it is safe to do so. Replace the example type, protocol, and port with values supported by the service policy.

### AVC indicates a missing or unexpected rule

Do not immediately pipe all audit events into `audit2allow` and install the resulting module. First verify the event is caused by the intended service operation, rule out incorrect labels, wrong service domains, and unsupported configuration, then consult the service policy documentation or security owner. Custom policy should be narrow, reviewed, documented, and tested before deployment.

## 5. Validate the correction

Reproduce the original operation and verify the service. Search for new denials after the change:

```bash
sudo systemctl status service-name --no-pager -l
sudo journalctl -u service-name --since '10 minutes ago' --no-pager
sudo ausearch -m AVC,USER_AVC -ts recent -i
getenforce
```

Confirm the service works as intended while SELinux remains enforcing. If the denial persists, compare the new event with the original and re-check labels, process domain, user identity, and the exact resource involved. Avoid accumulating policy changes from multiple unverified attempts.

## 6. Temporary permissive diagnosis

Changing to permissive mode is not the normal diagnostic path. If an approved, time-limited test is necessary to determine whether SELinux is involved, capture the current state, coordinate the test, and restore enforcing mode immediately afterward:

```bash
getenforce
sudo setenforce 0
# Reproduce the issue once and record the time.
sudo setenforce 1
getenforce
sudo ausearch -m AVC,USER_AVC -ts recent -i
```

Permissive mode logs denials but does not enforce them, so it can expose data or services during the test. Do not make it persistent in `/etc/selinux/config`. Confirm `getenforce` reports `Enforcing` before ending the test or returning the host to service.

## 7. Escalation checklist

Escalate with the relevant evidence when a denial remains unexplained or requires custom policy:

- Exact operation, timestamp, service, user, and resource path or port
- Matching AVC event, including source and target contexts and denied permission
- `getenforce`, service status, and relevant service logs
- File labels and `matchpathcon` output, or current port/boolean configuration
- Recent changes to service configuration, file locations, or SELinux policy
- Corrective changes already tested and their results
