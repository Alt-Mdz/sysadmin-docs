# RHEL-10.1 & ROCKY 10.1 Linux Sysadmin Documentation

Technical documentation, procedures, troubleshooting guides and reference material for Linux system administration, with a focus on **Rocky Linux** and **RHEL-based systems**.

The goal of this repository is provide practical and automated solutions

---

## Contents

* [Storage](#storage)
* [Systemd](#systemd)
* [SELinux](#selinux)
* [Firewalld](#firewalld)
* [Networking](#networking)
* [Users and Permissions](#users-and-permissions)
* [DNF and Package Management](#dnf-and-package-management)
* [Repositories](#repositories)
* [NFS](#nfs)
* [Samba](#samba)
* [Bash](#bash)
* [Troubleshooting](#troubleshooting)

---

## Storage

Procedures and reference material related to local storage management.

* [LVM](docs/storage/lvm.md)
* [Partitions](docs/storage/partitions.md)
* [Filesystems](docs/storage/filesystems.md)
* [Mounts and `/etc/fstab`](docs/storage/mounts.md)
* [XFS](docs/storage/xfs.md)
* [Storage troubleshooting](docs/storage/troubleshooting.md)

---

## Systemd

Service management and system initialization.

* [Systemd basics](docs/systemd/overview.md)
* [Managing services](docs/systemd/services.md)
* [Targets](docs/systemd/targets.md)
* [Journalctl](docs/systemd/journalctl.md)
* [Timers](docs/systemd/timers.md)
* [Troubleshooting services](docs/systemd/troubleshooting.md)

---

## SELinux

Security administration and troubleshooting.

* [SELinux overview](docs/security/selinux/overview.md)
* [Contexts](docs/security/selinux/contexts.md)
* [Booleans](docs/security/selinux/booleans.md)
* [File labeling](docs/security/selinux/file-labeling.md)
* [Troubleshooting](docs/security/selinux/troubleshooting.md)

---

## Firewalld

Host-based firewall configuration and troubleshooting.

* [Zones](docs/security/firewalld/zones.md)
* [Services](docs/security/firewalld/services.md)
* [Ports](docs/security/firewalld/ports.md)
* [Rich rules](docs/security/firewalld/rich-rules.md)
* [Troubleshooting](docs/security/firewalld/troubleshooting.md)

---

## Networking

Network configuration and diagnostics.

* [NetworkManager](docs/networking/networkmanager.md)
* [nmcli](docs/networking/nmcli.md)
* [Static IP configuration](docs/networking/static-ip.md)
* [DNS](docs/networking/dns.md)
* [Routing](docs/networking/routing.md)
* [Network troubleshooting](docs/networking/troubleshooting.md)

---

## Users and Permissions

User, group and access management.

* [User management](docs/users/users.md)
* [Groups](docs/users/groups.md)
* [sudo](docs/users/sudo.md)
* [File permissions](docs/users/permissions.md)
* [ACLs](docs/users/acls.md)

---

## DNF and Package Management

Package management for Rocky Linux and RHEL-based systems.

* [DNF](docs/software/dnf.md)
* [Package queries](docs/software/package-queries.md)
* [Package installation and removal](docs/software/packages.md)
* [Package history](docs/software/history.md)
* [GPG keys](docs/software/gpg.md)

---

## Repositories

Repository configuration and management.

* [Repository configuration](docs/repositories/configuration.md)
* [Creating a local repository](docs/repositories/local-repository.md)
* [Repository troubleshooting](docs/repositories/troubleshooting.md)
* [Offline systems](docs/repositories/offline-systems.md)

---

## NFS

Network File System administration.

* [NFS server](docs/services/nfs/server.md)
* [NFS client](docs/services/nfs/client.md)
* [Exports](docs/services/nfs/exports.md)
* [NFS troubleshooting](docs/services/nfs/troubleshooting.md)

---

## Samba

SMB/CIFS file sharing.

* [Samba server](docs/services/samba/server.md)
* [Shares](docs/services/samba/shares.md)
* [Permissions](docs/services/samba/permissions.md)
* [Samba troubleshooting](docs/services/samba/troubleshooting.md)

---

## Bash

Shell scripting and automation concepts.

* [Bash fundamentals](docs/bash/fundamentals.md)
* [Variables](docs/bash/variables.md)
* [Conditionals](docs/bash/conditionals.md)
* [Loops](docs/bash/loops.md)
* [Functions](docs/bash/functions.md)
* [Exit codes](docs/bash/exit-codes.md)
* [Input validation](docs/bash/input-validation.md)

> Automation scripts are maintained separately in a private repository.

---

## Troubleshooting

Practical troubleshooting procedures organized by symptom or subsystem.

* [Boot problems](docs/troubleshooting/boot.md)
* [Disk full](docs/troubleshooting/disk-full.md)
* [Network connectivity](docs/troubleshooting/network.md)
* [Service failures](docs/troubleshooting/services.md)
* [SELinux denials](docs/troubleshooting/selinux.md)
* [Permission problems](docs/troubleshooting/permissions.md)
* [Package management](docs/troubleshooting/packages.md)

---

## Documentation Principles

The documentation focuses on:

* Practical procedures
* Reproducible commands
* Verification steps
* Troubleshooting methodology
* Common failure scenarios
* Rollback considerations
* Automated system tasks
* Security implications

The goal is not to document every available command, but to document the commands and procedures that are useful in real-world administration.

---

## Environment

Primary environment:

* Rocky Linux 10.1
* RHEL-10.1
* systemd
* NetworkManager
* SELinux
* firewalld
* XFS
* LVM
* Bash

Commands and procedures may vary between major OS versions. Always verify the applicable documentation for the target system.

---

## Repository Structure

```
.
├── README.md
│
└── docs/
    ├── bash/
    ├── networking/
    ├── repositories/
    ├── security/
    │   ├── firewalld/
    │   └── selinux/
    ├── services/
    │   ├── nfs/
    │   └── samba/
    ├── software/
    ├── storage/
    ├── systemd/
    ├── troubleshooting/
    └── users/
```

---

## Related Projects

Automation and administrative scripts are maintained separately in a private repository.

---

## Disclaimer


Always validate procedures in a non-production environment before applying them to production systems.

Commands that modify storage, networking, security policies, boot configuration or system services should be reviewed carefully before execution.

---
