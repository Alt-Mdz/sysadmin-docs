# RHEL 10.1: Networking Operations and Troubleshooting

This SOP provides a practical baseline for common networking tasks on Rocky Linux 10.1 and RHEL 10.1 systems. Run commands as `root` or with `sudo`. Always confirm the interface name, IP plan, and current state before changing network configuration.

> **Caution:** Network changes can interrupt remote access or break service connectivity. Test changes in a controlled environment before applying them to production systems.

## 1. Identify the host and interfaces

Check the OS and hardware interfaces:

```bash
cat /etc/os-release
uname -r
ip addr show
nmcli general status
nmcli device status
nmcli connection show
```

Inspect routing and DNS defaults:

```bash
ip route show
ip route show table main
cat /etc/resolv.conf
```

Use the network interface name to target a specific device, for example `eth0`, `ens192`, or `eno1`.

## 2. Check connectivity and link status

Verify link state and transceiver status:

```bash
ip link show up
ip link show dev eth0
ethtool eth0
```

Check if the host can reach a local gateway or remote system:

```bash
ping -c 3 192.168.1.1
ping -c 3 8.8.8.8
```

Check port connectivity and service listeners:

```bash
ss -tulpn | head
ss -tulpn | grep :22
ss -tulpn | grep :443
```

## 3. Inspect IP configuration

Show current IPv4 and IPv6 addresses:

```bash
ip addr show
ip -4 addr show
ip -6 addr show
```

Check whether an interface holds a valid address:

```bash
ip addr show dev eth0
```

If using NetworkManager, inspect the profile for the interface:

```bash
nmcli connection show
nmcli connection show "System eth0"
nmcli device show eth0
```

## 4. Configure a static IP with NetworkManager

Example: assign `192.168.10.25/24` and set the gateway to `192.168.10.1`.

Create or modify a connection profile:

```bash
sudo nmcli connection add type ethernet ifname eth0 con-name "eth0-static" ipv4.method manual ipv4.addresses 192.168.10.25/24 ipv4.gateway 192.168.10.1 ipv4.dns "8.8.8.8 1.1.1.1" connection.autoconnect yes
```

Activate the connection:

```bash
sudo nmcli connection up "eth0-static"
```

Verify the result:

```bash
ip addr show dev eth0
ip route show
cat /etc/resolv.conf
```

To edit an existing profile:

```bash
sudo nmcli connection modify "eth0-static" ipv4.addresses 192.168.10.25/24 ipv4.gateway 192.168.10.1 ipv4.dns "8.8.8.8 1.1.1.1"
sudo nmcli connection up "eth0-static"
```

## 5. Configure a DHCP interface

Ensure the interface is managed and uses DHCP:

```bash
sudo nmcli connection modify "System eth0" ipv4.method auto
sudo nmcli connection up "System eth0"
```

Confirm the lease was received:

```bash
ip addr show dev eth0
ip route show default
```

## 6. Manage hostname and DNS

Set a host name:

```bash
sudo hostnamectl hostname server01.example.com
```

Check the local hostname and DNS resolution:

```bash
hostnamectl
hostname
getent hosts server01.example.com
```

Test DNS name resolution:

```bash
nslookup example.com
getent ahostsv4 example.com
```

## 7. Review routing

Show the current route table:

```bash
ip route show
ip route get 8.8.8.8
```

Add a static route if required:

```bash
sudo ip route add 10.10.0.0/24 via 192.168.10.1 dev eth0
```

Persist the route in NetworkManager or the OS routing configuration depending on your environment, then verify it after reboot.

## 8. Troubleshooting connectivity problems

### Interface is down

```bash
ip link show eth0
sudo ip link set eth0 up
ip addr show dev eth0
```

### No IP address assigned

```bash
nmcli device status
nmcli connection show
sudo nmcli connection up "eth0-static"
journalctl -u NetworkManager -b --no-pager
```

### No default gateway

```bash
ip route show
sudo ip route add default via 192.168.10.1 dev eth0
```

### DNS is not working

```bash
cat /etc/resolv.conf
nslookup example.com
getent hosts example.com
sudo systemctl restart NetworkManager
```

### Remote host is unreachable

```bash
ping -c 3 192.168.10.1
ping -c 3 8.8.8.8
traceroute 8.8.8.8
ip route show
ss -tulpn | grep LISTEN
```

## 9. Check common service ports

Verify whether a service is listening on the expected port:

```bash
ss -tulpn | grep :22
ss -tulpn | grep :80
ss -tulpn | grep :443
```

If the port is not listening, check the service status:

```bash
sudo systemctl status sshd
sudo systemctl status httpd
sudo journalctl -u sshd -b --no-pager
```

## 10. Packet-level troubleshooting

Use `ping`, `tracepath`, and `ss` for basic validation:

```bash
ping -c 3 10.0.0.10
tracepath 8.8.8.8
ss -s
```

Check whether the firewall is blocking traffic:

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
```

## 11. Safe networking change workflow

Use this sequence for a controlled network change:

```bash
ip addr show
ip route show
nmcli connection show
sudo nmcli connection modify "eth0-static" ipv4.addresses 192.168.10.25/24 ipv4.gateway 192.168.10.1 ipv4.dns "8.8.8.8 1.1.1.1"
sudo nmcli connection up "eth0-static"
ip addr show dev eth0
ip route show
```

Then verify:

```bash
ping -c 3 192.168.10.1
ping -c 3 8.8.8.8
```

## 12. Best practices

- Verify the interface name before editing a profile.
- Prefer `nmcli` for NetworkManager-managed systems.
- Keep a record of the intended IP, gateway, and DNS values.
- Check service listeners and firewall rules after network changes.
- Test reachability before declaring the change successful.

This SOP is intended as a practical operating reference for networking tasks on Rocky Linux 10.1. Always validate the host’s network state before and after a change, especially on production systems.
