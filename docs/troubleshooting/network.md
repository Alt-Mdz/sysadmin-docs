# Rocky Linux 10.1: Network Connectivity Troubleshooting

This SOP provides a layered workflow for diagnosing connectivity failures on Rocky Linux 10.1 and RHEL 10.1. Run read-only checks first and record the host, affected interface, source and destination, protocol/port, time of failure, and recent changes. Replace example interface names, addresses, hostnames, and ports with values for your environment.

> **Caution:** Network profile changes, interface cycling, or restarting NetworkManager can disconnect remote sessions. Use console or out-of-band access for risky changes, and confirm the rollback path before applying them.

## 1. Identify scope and current state

Determine whether the issue affects one application, one host, one network segment, or all destinations. Confirm whether IPv4, IPv6, wired, wireless, VPN, or only name-based access is affected.

Collect the current host and NetworkManager state:

```bash
hostnamectl
ip -br link
ip -br address
ip route show
ip -6 route show
nmcli general status
nmcli device status
nmcli connection show --active
```

Check the target and source address that the kernel would use for a destination:

```bash
ip route get 192.0.2.20
```

Use the actual destination IP. The selected interface and source address should match the expected network path.

## 2. Check physical or virtual link

Identify the interface and inspect its link state:

```bash
ip link show dev ens192
nmcli device show ens192
sudo ethtool ens192
```

A `DOWN` state, missing carrier, or `NO-CARRIER` points to a local link, switch port, cable, virtual NIC, or upstream issue. If the interface should be managed by NetworkManager, inspect the associated connection profile before activating it:

```bash
nmcli connection show
nmcli connection show "System ens192"
```

Do not repeatedly cycle the interface on a remote host. If link is down, verify the switch port or hypervisor NIC and cabling with the network owner.

## 3. Check address and DHCP configuration

Check for the expected address, prefix, and DHCP lease:

```bash
ip -4 address show dev ens192
ip -6 address show dev ens192
nmcli device show ens192
nmcli connection show "System ens192"
```

If an address is missing or incorrect, inspect NetworkManager logs and the active profile before changing it:

```bash
sudo journalctl -u NetworkManager -b --no-pager
nmcli -f GENERAL,IP4,IP6,DHCP4,DHCP6 device show ens192
```

Confirm the profile uses the intended method (`auto` for DHCP or `manual` for static addressing), addresses, gateway, and DNS servers. Do not create a second profile or assign an address until you know whether the interface is expected to use DHCP or a static configuration.

## 4. Test connectivity in layers

Test the configured default gateway first, then a known reachable IP in the intended network or service path:

```bash
ip route show default
ping -c 3 192.168.10.1
ping -c 3 192.0.2.20
```

ICMP may be filtered, so a failed ping alone does not prove the host or service is unreachable. Test the actual application port when possible:

```bash
nc -vz 192.0.2.20 443
curl -v --connect-timeout 5 https://service.example.com/
```

If `nc` is unavailable, use an approved network diagnostic package or test with the application client. Compare results from another host on the same network to determine whether the problem is local or upstream.

## 5. Diagnose routing failures

Inspect routes and neighbor resolution:

```bash
ip route show
ip -6 route show
ip route get 192.0.2.20
ip neigh show
```

Check for a missing or unexpected default gateway, incorrect prefix, overlapping routes, or a route through the wrong interface. For an unreachable subnet, compare the host route with the network design and inspect intermediate hops where permitted:

```bash
tracepath 192.0.2.20
```

A host may not receive replies to traceroute probes because intermediate devices filter them. Confirm routing and ACLs with the network team before adding a static route or changing a gateway.

## 6. Diagnose DNS failures

First test the destination by IP, then test name resolution:

```bash
getent ahosts service.example.com
cat /etc/resolv.conf
nmcli device show ens192 | grep -E 'IP4.DNS|IP6.DNS|IP4.DOMAIN|IP6.DOMAIN'
```

If IP connectivity works but name resolution fails, verify configured DNS servers, search domains, and resolver reachability. Query a specific approved DNS server if `dig` is installed:

```bash
dig @192.168.10.53 service.example.com
```

Do not edit `/etc/resolv.conf` directly when it is managed by NetworkManager. Correct the active NetworkManager profile or the DHCP-provided settings, then verify with `getent` and the affected application.

## 7. Check firewalld and listening services

For inbound connection failures, confirm the service is running and listening on the expected address and port:

```bash
sudo systemctl status service-name --no-pager
sudo ss -lntup
sudo journalctl -u service-name -b --no-pager
```

A listener bound only to `127.0.0.1` or `::1` is not reachable through the host's external interface. Check the active firewalld zone and allowed services or ports:

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
sudo firewall-cmd --get-zone-of-interface=ens192
```

Compare the rule with the approved security policy. Do not disable firewalld as a diagnostic shortcut. If an authorized rule change is required, scope it to the correct zone and service/port, then test from a remote client.

For outbound failures, check local firewall policy and any required proxy, VPN, or upstream egress controls. `ss -ntup` can show current connections and their state:

```bash
sudo ss -ntup
```

## 8. Review logs and packet-level evidence

Inspect recent kernel and NetworkManager messages around the failure:

```bash
sudo journalctl -b -u NetworkManager --since '30 minutes ago' --no-pager
sudo journalctl -b -k --since '30 minutes ago' --no-pager
```

If available and approved, capture a short, narrowly filtered packet trace while reproducing the issue:

```bash
sudo tcpdump -ni ens192 host 192.0.2.20 and port 443
```

A packet capture may contain sensitive information. Store it securely and stop it as soon as enough evidence is collected. If requests leave the host but no replies return, investigate the remote host, routing, ACLs, or upstream firewall. If no request leaves, inspect local route selection, address configuration, and firewall rules.

## 9. Apply and verify a controlled correction

Before changing a profile, record its current values and confirm console or rollback access. Example inspection:

```bash
nmcli connection show "System ens192"
```

Make only the change supported by the diagnosis, then reactivate the connection if necessary. Reactivating can interrupt remote access:

```bash
sudo nmcli connection up "System ens192"
```

Avoid restarting NetworkManager on a remote host unless there is a maintenance window and an independent access path. After any change, verify:

```bash
ip -br address
ip route show
getent ahosts service.example.com
nc -vz service.example.com 443
```

Test the actual application workflow and confirm that required connectivity remains available over both IPv4 and IPv6 where applicable.

## 10. Escalation checklist

Escalate with the following evidence when the issue is outside the host or remains unresolved:

- Source host, interface, source IP, destination IP or hostname, and port/protocol
- Exact start time and whether the failure is intermittent
- Link state, address configuration, route selection, and DNS results
- Gateway and destination test results, including whether tests used ICMP or TCP
- Relevant NetworkManager/kernel logs and a short packet trace if approved
- Recent host, switch, firewall, VPN, DNS, or hypervisor changes
