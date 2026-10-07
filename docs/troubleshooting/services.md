# Rocky Linux 10.1: Service Failure Troubleshooting

This SOP provides a repeatable workflow for common systemd-managed service failures on Rocky Linux 10.1 and RHEL 10.1. Replace `service-name` with the unit name, such as `sshd`, `httpd`, or `postgresql`. Run read-only checks first and record the error, time, recent changes, and impact before restarting a production service.

> **Caution:** Restarting or stopping a service can interrupt users or data processing. Confirm service impact, dependencies, and the approved recovery procedure before changing its state.

## 1. Capture the service state and logs

Check unit state, configuration, and recent logs:

```bash
sudo systemctl status service-name --no-pager -l
sudo systemctl show service-name -p LoadState,ActiveState,SubState,Result,ExecMainStatus,FragmentPath
sudo systemctl cat service-name
sudo journalctl -u service-name -b -n 200 --no-pager
sudo systemctl --failed --no-pager
```

If the issue began after a reboot, check whether the unit is enabled and inspect boot-specific logs:

```bash
sudo systemctl list-units --state=running
sudo systemctl is-enabled service-name
sudo journalctl -b -u service-name --no-pager
sudo journalctl -b -p warning..alert --no-pager
```

Use the service's own configuration validator when available before restarting it. Check the service's documentation for the correct validation command.

## 2. Service fails to start

Check the exit status and the first error in the journal:

```bash
sudo systemctl status service-name --no-pager -l
sudo journalctl -u service-name -b --no-pager
sudo systemctl cat service-name
```

Common causes include invalid configuration, missing files or executables, incorrect ownership or permissions, unavailable dependencies, and port conflicts. Correct the cause first, validate the configuration, then start the service during an approved window:

```bash
sudo systemctl start service-name
sudo systemctl status service-name --no-pager -l
```

Do not repeatedly restart a service while its failure cause is unknown; this can obscure logs or cause further disruption.

## 3. Service repeatedly restarts or hits start limits

Inspect the unit's restart policy and failure history:

```bash
sudo systemctl show service-name -p Restart,RestartUSec,StartLimitIntervalUSec,StartLimitBurst
sudo journalctl -u service-name -b --no-pager
sudo systemctl status service-name --no-pager -l
```

Look for a configuration error, missing dependency, invalid credentials, unavailable storage, or an application crash. Fix the underlying issue first. `systemctl reset-failed` only clears systemd's failed/start-limit state; it does not fix the service:

```bash
sudo systemctl reset-failed service-name
sudo systemctl start service-name
```

Use these commands only after correcting the cause and when starting the service is safe.

## 4. Service is enabled but did not start at boot

`enabled` means systemd is configured to start a unit through its install links; it does not guarantee startup succeeded. Check the unit state, boot logs, and dependencies:

```bash
sudo systemctl is-enabled service-name
sudo systemctl status service-name --no-pager -l
sudo journalctl -b -u service-name --no-pager
sudo systemctl list-dependencies service-name
sudo systemctl list-dependencies --reverse service-name
```

Check for failed prerequisites, incorrect ordering, unavailable network or mounts, and service-specific conditions. Do not add ordering dependencies or enable additional units without confirming the service's actual boot requirements.

## 5. Dependency or mount is unavailable

If logs name a required unit, inspect that unit directly:

```bash
sudo systemctl status dependency-name --no-pager -l
sudo journalctl -b -u dependency-name --no-pager
sudo systemctl list-dependencies service-name
```

For a filesystem or mount dependency, verify the mount and its configuration:

```bash
findmnt
sudo findmnt --verify --verbose
sudo systemctl status local-fs.target --no-pager
```

Repair the dependency first, then start the affected service. Avoid masking or disabling dependencies as a workaround unless the service owner confirms they are not required.

## 6. Port is already in use or service is not listening

If the service fails with an address-in-use error, find the listener and confirm whether it is expected:

```bash
sudo ss -lntup
sudo ss -lntup | grep ':443'
```

Do not stop another process until its owner and purpose are known. If the service is active but the expected port is absent, inspect its configuration, bind address, and logs:

```bash
sudo systemctl status service-name --no-pager -l
sudo journalctl -u service-name -b --no-pager
sudo ss -lntup | grep ':PORT'
```

A service bound only to loopback (`127.0.0.1` or `::1`) is not reachable through the host's external address. For remote connection failures, also inspect the firewall and routing rather than assuming the service itself has failed.

## 7. Permission denied or SELinux denial

Check the executable, configuration, data paths, and parent directory permissions:

```bash
sudo namei -l /path/to/file
sudo ls -lZ /path/to/file
sudo journalctl -u service-name -b --no-pager
sudo ausearch -m AVC -ts recent
sudo sealert -a /var/log/audit/audit.log #dnf install troubbleshoot-server
```

Correct ownership and permissions according to the service's security model. For SELinux denials, identify the denied operation and expected file type; use the relevant SELinux procedure to correct labels or policy. Do not disable SELinux or apply broad `chmod 777` permissions to make the service start.

## 8. Service is active but unhealthy or unavailable

An `active` state only means systemd considers the unit running. Check the service's own health endpoint or client, listener, and logs:

```bash
sudo systemctl is-active service-name
sudo systemctl status service-name --no-pager -l
sudo ss -lntup
sudo journalctl -u service-name -b -n 200 --no-pager
```

Test locally using the service's expected protocol and port. If local access works but remote access fails, check the host firewall, network path, DNS, and upstream controls. If the process is running but its health check fails, investigate application dependencies, credentials, storage, and resource limits.

## 9. Service is slow to start or times out

Review logs around the start attempt and inspect the unit's timeout and type:

```bash
sudo journalctl -u service-name -b --no-pager
sudo systemctl show service-name -p Type,TimeoutStartUSec,ExecStart,ExecStartPre,ExecStartPost
sudo systemctl cat service-name
sudo systemctl list-timers | service.timer
```

Look for slow dependencies, blocked mounts, DNS or network waits, and application startup errors. Do not increase the timeout until logs show that the service is making healthy progress and the service owner confirms the longer startup is expected.

## 10. Verify recovery and document the change

After recovery, confirm the unit is active, no new failures appeared, and the application works from the intended client path:

```bash
sudo systemctl is-active service-name
sudo systemctl status service-name --no-pager -l
sudo systemctl --failed --no-pager
sudo journalctl -u service-name --since '10 minutes ago' --no-pager
```

Record the root cause, corrective change, validation result, and any follow-up monitoring. If the cause is not clear or the issue recurs, preserve the journal output and escalate to the service owner rather than repeatedly restarting the unit.
