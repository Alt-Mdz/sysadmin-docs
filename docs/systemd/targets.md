# Rocky Linux 10.1: systemd Targets and Boot Modes

This SOP covers common systemd target management on Rocky Linux 10.1 and RHEL 10.1. Run commands as `root` or with `sudo`. Confirm the host, target state, and service dependencies before switching boot behavior or changing the default startup target.

> **Caution:** Target changes affect what services start at boot and how the system enters multi-user or rescue modes. Test changes on a non-production host before applying them to production systems.

## 1. Basic target checks

Identify the operating system and view current target information:

```bash
cat /etc/os-release
uname -r
systemctl --version
systemctl list-units --type=target --all --no-pager
systemctl get-default
systemctl list-units --type=target --state=active --all --no-pager
```

Check the current runtime target and the default target:

```bash
systemctl list-units --type=target --state=active --no-pager
systemctl get-default
systemctl show -p DefaultDependencies -p Id -p FragmentPath getty.target
```

## 2. Understand common targets

The most commonly used targets are:

- `multi-user.target`: normal CLI system state
- `graphical.target`: normal graphical login state
- `rescue.target`: minimal environment for repair tasks
- `emergency.target`: minimal single-user recovery mode
- `basic.target`: base unit set before multi-user
- `shutdown.target`: stop actions during shutdown
- `network-online.target`: network connectivity is available

Inspect a target and its dependencies:

```bash
systemctl list-dependencies multi-user.target
systemctl list-dependencies graphical.target
systemctl list-dependencies rescue.target
systemctl show multi-user.target --property=AllowIsolate,Description,After,Before,UnitFileState
```

A target is usually not a process; it is a unit that pulls in other units and provides a stage of the boot process.

## 3. Check the active target and default target

View the current runtime state:

```bash
systemctl isolate --dry-run multi-user.target
systemctl list-units --type=target --state=active --no-pager
systemctl get-default
```

To view the target that is currently booted:

```bash
systemctl list-units --type=target --state=active --all --no-pager | grep -E 'multi-user|graphical|rescue|emergency'
```

The default target is controlled by a symlink in `/etc/systemd/system/default.target`:

```bash
ls -l /etc/systemd/system/default.target
readlink -f /etc/systemd/system/default.target
```

## 4. Change the default target

Change the default boot target to multi-user mode:

```bash
sudo systemctl set-default multi-user.target
systemctl get-default
```

Change the default boot target to graphical mode:

```bash
sudo systemctl set-default graphical.target
systemctl get-default
```

Validate the symlink after changing it:

```bash
ls -l /etc/systemd/system/default.target
readlink -f /etc/systemd/system/default.target
```

> If the host is supposed to boot to a graphical desktop, confirm that the display manager is installed and enabled before switching the default target.

## 5. Switch targets immediately without rebooting

Switch to a different target in the running system:

```bash
sudo systemctl isolate multi-user.target
sudo systemctl isolate graphical.target
```

Switch to rescue mode without rebooting:

```bash
sudo systemctl rescue
```

Switch to emergency mode:

```bash
sudo systemctl emergency
```

Return to the default boot target after rescue operations:

```bash
sudo systemctl isolate multi-user.target
```

To stop a target and leave the system in a minimal state, use:

```bash
sudo systemctl isolate rescue.target
```

## 6. Use rescue and emergency mode safely

### Rescue mode

Rescue mode gives a basic shell and stops most normal services. It is useful when a system cannot boot normally, but the root filesystem is intact.

```bash
sudo systemctl rescue
```

Once in rescue mode:

```bash
mount
lsblk -f
journalctl -b -n 100 --no-pager
```

To leave rescue mode and continue booting normally:

```bash
exit
```

### Emergency mode

Emergency mode is even more minimal. It drops to a root shell before the root filesystem is remounted read-write and before most services are started.

```bash
sudo systemctl emergency
```

This mode is useful for fixing serious boot issues, but it is not the normal operational state. Commands should be limited to essential recovery tasks.

## 7. Target dependency and boot troubleshooting

Inspect all targets and their relationships:

```bash
systemctl list-unit-files --type=target --no-pager
systemctl list-dependencies --reverse graphical.target
systemctl list-dependencies --reverse multi-user.target
```

Check if the default target points to the wrong state:

```bash
systemctl get-default
ls -l /etc/systemd/system/default.target
```

If the system boots into the wrong target or a unit does not start, inspect the logs and the boot target chain:

```bash
journalctl -b -u graphical.target --no-pager
journalctl -b -u multi-user.target --no-pager
systemctl status graphical.target
systemctl status multi-user.target
```

### Target is not appearing or is not active

Check whether the unit file exists and whether the target is enabled:

```bash
systemctl list-unit-files --type=target | grep -E 'multi-user|graphical|rescue|emergency'
systemctl cat graphical.target
```

If a target is missing, verify the systemd unit file and reload the manager:

```bash
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

## 8. Boot order and graphical login checks

Check whether the display manager is enabled:

```bash
systemctl list-unit-files --type=service | grep -E 'gdm|lightdm|sddm|xdm'
systemctl status gdm --no-pager
```

If the default target is `graphical.target`, make sure the display manager is properly installed and enabled:

```bash
sudo systemctl enable --now gdm
sudo systemctl status gdm
```

If the server should boot to console mode, keep the default as `multi-user.target`:

```bash
sudo systemctl set-default multi-user.target
```

## 9. Safe target change workflow

Use the following sequence for a controlled change:

```bash
systemctl get-default
systemctl list-units --type=target --state=active --no-pager
sudo systemctl set-default multi-user.target
systemctl get-default
sudo reboot
```

After the reboot, confirm the target state:

```bash
systemctl list-units --type=target --state=active --no-pager
systemctl get-default
```

## 10. Common recovery commands

If the system cannot reach the expected target, use recovery procedures:

```bash
sudo systemctl rescue
sudo systemctl emergency
sudo systemctl reboot
sudo systemctl emergency --force
```

If the system hangs before the login prompt, review the boot log:

```bash
journalctl -b -n 200 --no-pager
```

Check the target chain and services required for boot:

```bash
systemctl list-dependencies graphical.target
systemctl list-dependencies multi-user.target
```

## 11. Best practices

- Prefer `systemctl get-default` before making any boot-target change.
- Use `multi-user.target` for headless server systems.
- Use `graphical.target` only when a compatible graphical display manager is installed.
- Always validate the boot target after a reboot.
- Use rescue and emergency modes only for recovery and repair.
- Review logs and target dependencies before changing boot behavior.

This SOP is intended as a practical reference for systemd multitarget operations on Rocky Linux 10.1. Always verify the current boot target and service dependencies before making changes in production.
