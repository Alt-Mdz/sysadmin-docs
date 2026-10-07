# RHEL 10.1: Service Management with systemd

This SOP covers common operational tasks for managing services on Rocky Linux 10.1 and RHEL 10.1 systems. Run commands as `root` or with `sudo`. Confirm the target host, service unit name, and runtime state before restarting or disabling services.

> **Caution:** Service management changes affect running applications and boot behavior. Validate the service name, review unit configuration, and test in a non-production environment before applying changes to production systems.

## 1. Basic service checks

Use these commands to identify the host OS and inspect service state:

```bash
cat /etc/os-release
uname -r
systemctl --version
systemctl list-units --type=service --state=running
systemctl list-units --type=service --all | head
```

List active, failed, and enabled services:

```bash
systemctl list-units --type=service --all --no-pager
systemctl list-units --type=service --state=failed --all
systemctl list-unit-files --type=service --state=enabled,disabled
```

## 2. Service status and unit discovery

Check the current state of a service:

```bash
sudo systemctl status sshd
sudo systemctl is-active sshd
sudo systemctl is-enabled sshd
sudo systemctl show sshd --property=ActiveState,SubState,LoadState,UnitFileState,FragmentPath
```

Inspect service unit configuration and dependencies:

```bash
sudo systemctl cat sshd
sudo systemctl list-dependencies sshd
sudo systemctl list-dependencies --reverse sshd
sudo systemctl show -p WantedBy -p Before -p After sshd
```

When a service name is unknown or the unit is not loaded, search by name and description:

```bash
systemctl list-unit-files | grep -i httpd
systemctl list-units --state=running
systemctl list-units --type=service --all | grep -i nginx
systemctl --type=service --all --no-pager | grep -i ssh
```

## 3. Start, stop, restart, and reload services

Start a service:

```bash
sudo systemctl start sshd
```

Stop a service:

```bash
sudo systemctl stop sshd
```

Restart a service:

```bash
sudo systemctl restart sshd
```

Reload without interrupting the service, when supported:

```bash
sudo systemctl reload sshd
```

Check whether the service is healthy after a change:

```bash
sudo systemctl status sshd
sudo systemctl is-active sshd
sudo systemctl show -p ActiveState sshd
```

## 4. Enable and disable startup behavior

Enable a service to start at boot:

```bash
sudo systemctl enable --now sshd
```

Disable autostart:

```bash
sudo systemctl disable sshd
```

Enable but do not start immediately:

```bash
sudo systemctl enable --now sshd
```

Disable and stop the service now:

```bash
sudo systemctl disable --now sshd
```

Verify the unit file state:

```bash
sudo systemctl is-enabled sshd
sudo systemctl list-unit-files --type=service | grep sshd
```

## 5. View logs and identify failures

Check the journal for a specific service:

```bash
sudo journalctl -u sshd -b --no-pager
sudo journalctl -u sshd -n 100 --no-pager
```

Follow logs live:

```bash
sudo journalctl -fu sshd
```

Look for unit failures or dependency issues:

```bash
sudo journalctl -xe --no-pager
sudo systemctl --failed --no-pager
```

When a service fails to start, inspect the unit and logs together:

```bash
sudo systemctl cat sshd
sudo systemctl status sshd
sudo journalctl -u sshd -b --no-pager
```

## 6. Common service troubleshooting

### Service is not starting

Run the following checks:

```bash
sudo systemctl status service-name
sudo systemctl cat service-name
sudo journalctl -u service-name -b --no-pager
sudo systemctl list-dependencies service-name
```

Look for missing configuration files, permissions problems, invalid options, or dependencies that are not active. If the service depends on another unit, start that unit first and inspect its status:

```bash
sudo systemctl status dependency-unit
sudo systemctl start dependency-unit
```

### Service is enabled but failed to start at boot

Check the boot journal and the unit file state:

```bash
sudo systemctl is-enabled service-name
sudo systemctl status service-name
sudo journalctl -b -u service-name --no-pager
```

If the service should run on boot but is not starting, confirm the unit file is enabled and that the service has no conflicting dependency or failed precondition.

### Service is active but not listening on the expected port

Check the socket or port binding and review the daemon configuration:

```bash
sudo ss -tulpn | grep :80
sudo ss -tulpn | grep :443
sudo systemctl status service-name
sudo journalctl -u service-name -n 100 --no-pager
```

If the application binds to a different interface or port, update the service configuration and reload or restart the unit.

### Service is stuck during shutdown or restart

Use the unit status to learn whether it is blocked on lock files, filesystem state, or network resources:

```bash
sudo systemctl status service-name
sudo systemctl kill -s SIGTERM service-name
sudo journalctl -u service-name -b --no-pager
```

Only use `systemctl kill` when the normal stop sequence is not working and you have confirmed that the action is safe for the service.

## 7. Create and validate a custom service unit

Create a unit file in `/etc/systemd/system/` for a custom service. Example:

```ini
[Unit]
Description=Example Worker Service
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/example
ExecStart=/usr/local/bin/example-worker
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
## 7.1 Create a systemd.timer
[Unit]
Description=Trigger the backup script daily

[Timer]
# Option A: Run on a calendar schedule (e.g., every day at 2:30 AM)
OnCalendar=*-*-* 02:30:00

# Option B: Or run relative to events (e.g., 5 min after boot, and every 1 hour after)
# OnBootSec=5min
# OnUnitActiveSec=1h
# Persistent=true

[Install]
WantedBy=timers.target


Load the new service and verify it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now example-worker.timer
sudo systemctl list-timers
sudo systemctl status example-worker.timer
sudo systemctl cat example-worker.service
```

If the service does not start, check the unit file, log output, and any permissions or paths referenced by `ExecStart`. Custom scripts should be in /usr/local/sbin

## 8. Safe service change workflow

Use the following sequence whenever changing service behavior:

```bash
sudo systemctl status service-name
sudo systemctl cat service-name
sudo systemctl daemon-reload
sudo systemctl restart service-name
sudo systemctl status service-name
sudo journalctl -u service-name -n 50 --no-pager
```

This workflow confirms the current state, checks configuration, applies the change, validates the result, and reviews logs.

## 9. Recovery and rollback

If a service change causes unexpected behavior:

```bash
sudo systemctl status service-name
sudo systemctl stop service-name
sudo systemctl disable service-name
sudo journalctl -u service-name -b --no-pager
```

If you modified a unit file, restore the previous version from a known-good backup or source control before restarting the service. Revert service changes systematically and verify each step before repeating the procedure.

## 10. Best practices

- Use the service unit name exactly as reported by `systemctl list-units`.
- Prefer `status`, `restart`, and `journalctl` before making changes.
- Reload configuration when a service supports it instead of restarting unnecessarily.
- Review `ExecStart`, `Restart`, and `WantedBy` in the unit file.
- Validate service health and logs after each change.
- Keep changes small and document why the service was altered.

This SOP is meant to provide a repeatable working baseline for service management on Rocky Linux 10.1. and RHEL 10.1 Always confirm the target service, dependencies, and logs before making operational changes.
