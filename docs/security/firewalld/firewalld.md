# RHEL 10.1: firewalld Operations and Troubleshooting

This SOP provides a practical baseline for managing `firewalld` on Rocky Linux 10.1 and RHEL 10.1 systems. Run commands as `root` or with `sudo`. Validate the active zone, service, and port rules before making changes in production.

> **Caution:** Firewall changes can block legitimate access. Review the current rules, service requirements, and host role before adding or removing ports, services, or rich rules.

## 1. Basic inspection

Check whether `firewalld` is installed, enabled, and active:

```bash
sudo firewall-cmd --state
sudo firewall-cmd --version
sudo systemctl status firewalld
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --get-default-zone
```

List configured zones and current allowed services:

```bash
sudo firewall-cmd --get-zones
sudo firewall-cmd --zone=public --list-all
sudo firewall-cmd --zone=internal --list-all
```

## 2. Understand zones

Common zones include:

- `public` — default for general-purpose hosts
- `internal` — trusted internal networks
- `trusted` — fully trusted interfaces
- `dmz` — limited exposure
- `block` — all incoming traffic blocked
- `drop` — drop packets silently

Check which interface is assigned to a zone:

```bash
sudo firewall-cmd --get-zone-of-interface=eth0
sudo firewall-cmd --zone=public --list-interfaces
```

Change the default zone if required:

```bash
sudo firewall-cmd --set-default-zone=internal
sudo firewall-cmd --get-default-zone
```

## 3. Manage services

List built-in services:

```bash
sudo firewall-cmd --get-services
```

Add a service to the current zone:

```bash
sudo firewall-cmd --zone=public --add-service=http
sudo firewall-cmd --zone=public --add-service=https
```

Add a service permanently:

```bash
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --reload
```

Remove a service:

```bash
sudo firewall-cmd --permanent --zone=public --remove-service=http
sudo firewall-cmd --reload
```

List allowed services:

```bash
sudo firewall-cmd --zone=public --list-services
sudo firewall-cmd --permanent --zone=public --list-services
```

## 4. Manage ports

Open a port in the active runtime:

```bash
sudo firewall-cmd --zone=public --add-port=8080/tcp
```

Open a port permanently:

```bash
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp
sudo firewall-cmd --reload
```

Remove a port:

```bash
sudo firewall-cmd --permanent --zone=public --remove-port=8080/tcp
sudo firewall-cmd --reload
```

Verify port rules:

```bash
sudo firewall-cmd --zone=public --list-ports
sudo firewall-cmd --permanent --zone=public --list-ports
```

## 5. Rich rules and source filtering

Add a rich rule to permit traffic from a specific source:

```bash
sudo firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" service name="ssh" accept'
```

Add a permanent rich rule:

```bash
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port protocol="tcp" port="3306" service name="ssh" accept'
sudo firewall-cmd --reload
```

List rich rules:

```bash
sudo firewall-cmd --zone=public --list-rich-rules
sudo firewall-cmd --permanent --zone=public --list-rich-rules
```

## 6. Runtime vs permanent rules

`firewalld` stores runtime and permanent configuration separately:

```bash
sudo firewall-cmd --runtime-to-permanent
sudo firewall-cmd --permanent --list-all
sudo firewall-cmd --list-all
```

Use this workflow when making changes:

```bash
sudo firewall-cmd --zone=public --add-service=http
sudo firewall-cmd --zone=public --list-all
sudo firewall-cmd --runtime-to-permanent
sudo firewall-cmd --reload
```

> Temporary runtime rules disappear after a reboot unless you copy them to the permanent configuration.

## 7. Check open ports and service exposure

Use `ss` and `firewall-cmd` together to verify what is listening and what is allowed:

```bash
sudo ss -tulpn | grep LISTEN
sudo firewall-cmd --zone=public --list-all
sudo firewall-cmd --permanent --zone=public --list-all
```

Confirm the exact service mapping for a port:

```bash
sudo firewall-cmd --zone=public --list-services
sudo firewall-cmd --zone=public --list-ports
```

## 8. Troubleshooting common issues

### A service is blocked

Check the active zone and permitted services:

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
sudo firewall-cmd --permanent --zone=public --list-all
sudo firewall-cmd --zone=public --query-service=http
```

If the service should be allowed, add it to the correct zone:

```bash
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --reload
```

### A port is not reachable

Verify the port rule and the service listening condition:

```bash
sudo firewall-cmd --zone=public --list-ports
sudo firewall-cmd --permanent --zone=public --list-ports
sudo ss -tulpn | grep 8080
```

If the port is open but unreachable, check the interface assignment, source restrictions, and network path.

### A host is not reachable from a source network

Inspect rich rules and source restrictions:

```bash
sudo firewall-cmd --zone=public --list-rich-rules
sudo firewall-cmd --permanent --zone=public --list-rich-rules
```

Review source-based rules and remove or adjust the restriction if the access is intentional.

### Firewall is active but no policy is applied

Check the default zone and zone assignments:

```bash
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
```

If the interface belongs to a different zone, apply the policy there or reassign the interface.

## 9. Reset or repair a broken firewall policy

If needed, temporarily clear a rule from the current zone:

```bash
sudo firewall-cmd --zone=public --remove-service=http
sudo firewall-cmd --zone=public --remove-port=8080/tcp
```

If the system is locked out by a bad rule, use a local console or SSH to a trusted network and correct the zone configuration.

To remove all runtime rules for a zone while preserving permanent data, use caution:

```bash
sudo firewall-cmd --zone=public --reset
```

For a full reset of the permanent configuration, only do this when you are ready to rebuild the firewall policy from scratch:

```bash
sudo firewall-cmd --permanent --zone=public --set-target=default
```

## 10. Best practices

- Check active zones before changing policy.
- Prefer a service rule instead of a raw port rule when possible.
- Use `--permanent` for configuration changes that should survive reboot.
- Validate the runtime state after `--reload`.
- Test changes from a trusted network before exposing a service broadly.
- Document firewall changes and any special exceptions in a change log.

This SOP provides a repeatable firewall workflow for Rocky Linux 10.1 systems. Always verify the effective policy on the target host before opening or restricting network access.
