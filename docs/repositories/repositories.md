# RHEL 10.1: Repositories and Package Source Management

This SOP covers repository configuration and troubleshooting on Rocky Linux 10.1 and RHEL 10.1 systems. Run commands as `root` or with `sudo`. Confirm the repository URL, GPG key, and package architecture before making changes to package sources.

> **Caution:** Repository changes affect package availability, dependency resolution, and security posture. Validate the source before enabling or changing a repository in production.

## 1. Inspect the active repositories

List enabled repositories:

```bash
sudo dnf repolist
sudo dnf repolist --all
```

List repository configuration files:

```bash
ls /etc/yum.repos.d/
ls /etc/dnf.repos.d/
```

Read a repository definition:

```bash
sudo grep -R "\[.*\]" /etc/yum.repos.d /etc/dnf.repos.d
sudo cat /etc/yum.repos.d/rocky.repo
```

## 2. Understand repository files

A typical repository file looks like this:

```ini
[baseos]
name=Rocky Linux $releasever - BaseOS
mirrorlist=https://mirrors.rockylinux.org/mirrorlist?arch=$basearch&repo=BaseOS-$releasever
# baseurl=http://dl.rockylinux.org/pub/rocky/$releasever/BaseOS/$basearch/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-rocky
```

Common repository options:

- `name` — human-friendly description
- `baseurl` or `mirrorlist` — repository source
- `enabled` — whether the repo is active
- `gpgcheck` — verify package signatures
- `gpgkey` — signing key path or URL
- `metadata_expire` — cache refresh interval
- `repo_gpgcheck` — verify metadata signatures

## 3. Add a custom repository

Create a repository file in `/etc/yum.repos.d/`:

```bash
sudo dnf config-manager --add-repo <repository_URL or file >
sudo vi /etc/yum.repos.d/custom.repo
```

Example repository entry:

```ini
[custom]
name=Custom Repository
baseurl=https://example.com/repo/rhel-10-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://example.com/repo/RPM-GPG-KEY-custom
```

Refresh repository metadata:

```bash
sudo dnf clean all
sudo dnf makecache
sudo dnf repolist
```

## 4. Disable or enable a repository

Temporarily disable a repository:

```bash
sudo dnf config-manager --set-disabled appstream
```

Re-enable it:

```bash
sudo dnf config-manager --set-enabled appstream
```

Disable a repository by editing the file directly:

```ini
enabled=0
```

## 5. Check repository metadata and sync state

Look for metadata and leftover caches:

```bash
sudo dnf makecache
sudo ls -l /var/cache/dnf/
```

Force metadata refresh:

```bash
sudo dnf clean all
sudo dnf makecache --refresh
```

Check whether a repo is reachable:

```bash
curl -I https://example.com/repo/rhel-10-x86_64/
```

## 6. Validate packages before install

Check which repository provides a package:

```bash
sudo dnf repoquery --whatprovides nginx
sudo dnf repoquery --available nginx
```

Inspect package metadata:

```bash
sudo dnf info nginx
sudo dnf list available | grep -i nginx
```

This confirms the package source, version, and repository before installation.

## 7. Troubleshooting repository issues

### Repository metadata is stale or missing

```bash
sudo dnf clean all
sudo dnf makecache --refresh
sudo dnf repolist
```

### Repository is unreachable

```bash
curl -I https://example.com/repo/
ping -c 3 example.com
sudo dnf repolist --verbose
```

If the server is reachable but the repository is still failing, check the URL, TLS certificate, proxy settings, and firewall rules.

### GPG verification fails

```bash
sudo rpm --import /path/to/RPM-GPG-KEY
sudo dnf install package-name
```

Check the key file and verify the repository is using the correct public key.

### Package not available from enabled repos

```bash
sudo dnf repolist
sudo dnf search package-name
sudo dnf list available | grep -i package-name
```

If the package is not in the enabled repositories, either enable the correct repository or add the intended source.

## 8. Secure repository practices

- Prefer the vendor-signed repository and official GPG keys.
- Use HTTPS URLs when possible.
- Verify the `baseurl` or `mirrorlist` before enabling a repository.
- Keep mirror lists and metadata current.
- Restrict repository changes to approved sources and documented packages.

## 9. Local repository example

Create a local RPM repository on a shared filesystem:

```bash
sudo mkdir -p /srv/repo/rhel-10-x86_64
sudo createrepo_c /srv/repo/rhel-10-x86_64
```

Add the repository file:

```ini
[localrepo]
name=Local Repository
baseurl=file:///srv/repo/rhel-10-x86_64
enabled=1
gpgcheck=0
```

Refresh and test the repository:

```bash
sudo dnf clean all
sudo dnf makecache
sudo dnf list available | grep -i package-name
```

## 10. Safe repository change workflow

Use this sequence when adding or changing a repo:

```bash
sudo dnf repolist
sudo cat /etc/yum.repos.d/custom.repo
sudo dnf clean all
sudo dnf makecache --refresh
sudo dnf repolist
sudo dnf info package-name
```

This confirms the repo file, reloads metadata, and validates the package source before installing software.

## 11. Best practices

- Keep repository definitions simple and readable.
- Prefer official or approved vendor repositories.
- Validate repository metadata before installing or upgrading packages.
- Record any custom repository addition in your change log or runbook.
- Remove stale or unused repo files when they are no longer needed.

This SOP provides a repeatable process for repository management on Rocky Linux 10.1. Always verify the repository source and package availability before enabling new or custom package sources.
