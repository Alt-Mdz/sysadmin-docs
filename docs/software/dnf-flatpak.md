# RHEL 10.1: DNF and Flatpak Operations

This SOP provides a practical workflow for package management with DNF and for installing desktop applications with Flatpak on Rocky Linux 10.1 and RHEL 10.1 systems. Run commands as `root` or with `sudo`. Always confirm the package name, repository state, and target host before changing installed software.

> **Caution:** Package and application changes affect availability, dependencies, and system stability. Validate the source repository and package dependencies before installing or removing software in production.

## 1. Check the system and repositories

Identify the operating system and available repositories:

```bash
cat /etc/rocky-release
uname -r
sudo dnf repolist
sudo dnf repolist --all
sudo dnf makecache
```

Check DNF version and enabled modules:

```bash
dnf --version
sudo dnf module list
```

## 2. Query packages

Search for packages:

```bash
sudo dnf search nginx
sudo dnf search --all nginx
```

Show package information:

```bash
sudo dnf info nginx
sudo dnf info "@development-tools"
```

Check what is already installed:

```bash
sudo dnf list installed | head
sudo dnf list installed | grep -i httpd
```

## 3. Install packages

Install a specific package:

```bash
sudo dnf install nginx
```

Install a package group:

```bash
sudo dnf group install "Development Tools"
```

Install a package and all required dependencies:

```bash
sudo dnf install --assumeyes nginx
```

Install from a repository with a specific architecture when needed:

```bash
sudo dnf install --downloadonly --downloaddir=/tmp/packages nginx
```

## 4. Remove packages

Remove a package and its unneeded dependencies:

```bash
sudo dnf remove nginx
```

Remove a package group:

```bash
sudo dnf group remove "Development Tools"
```

Check the change before confirming the operation:

```bash
sudo dnf repoquery --whatrequires nginx
```

## 5. Update the system

Check for available updates:

```bash
sudo dnf check-update
```

Update installed packages:

```bash
sudo dnf upgrade
```

Update a single package:

```bash
sudo dnf update nginx
```

Apply system updates and restart services if needed:

```bash
sudo dnf upgrade --refresh
```

## 6. Review package history

List recent package transactions:

```bash
sudo dnf history
sudo dnf history info 10
```

Undo a recent transaction when necessary:

```bash
sudo dnf history undo 10
```

Use `dnf history` to identify the last change and confirm the package set before rollback.

## 7. Manage repositories

Add a repository from a file in `/etc/yum.repos.d/`:

```bash
sudo vi /etc/yum.repos.d/custom.repo
```

Example repository file:

```ini
[custom]
name=Custom Repository
baseurl=https://example.com/repo/rhel-10-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://example.com/repo/RPM-GPG-KEY
```

Refresh the metadata:

```bash
sudo dnf clean all
sudo dnf makecache
```

## 8. Troubleshooting DNF issues

### Repo metadata errors

```bash
sudo dnf clean all
sudo dnf makecache
sudo dnf repolist
```

If the repository is unreachable or the metadata is stale, verify URL, GPG key, and connectivity:

```bash
curl -I https://example.com/repo/rhel-10-x86_64
sudo dnf repolist --verbose
```

### Package not found

```bash
sudo dnf search package-name
sudo dnf list available | grep -i package-name
sudo dnf repolist
```

### Dependency resolution failure

```bash
sudo dnf install package-name --best --allowerasing
sudo dnf check
```

If the package conflicts with another installed package, review the dependency tree before proceeding.

### Package install is blocked by GPG

```bash
sudo rpm --import /path/to/RPM-GPG-KEY
sudo dnf install package-name
```

## 9. Install and manage Flatpak

Check whether Flatpak is installed:

```bash
flatpak --version
```

Install Flatpak if needed:

```bash
sudo dnf install flatpak
```

Add a remote repository:

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

Update remotes and installed applications:

```bash
flatpak update
flatpak remote-list
```

Install an application from Flathub:

```bash
flatpak install flathub org.mozilla.firefox
```

List installed Flatpak applications:

```bash
flatpak list
flatpak list --app
```

Run a Flatpak app:

```bash
flatpak run org.mozilla.firefox
```

Remove an installed Flatpak app:

```bash
flatpak uninstall org.mozilla.firefox
```

## 10. Manage Flatpak permissions and runtimes

View installed runtimes and extensions:

```bash
flatpak list --runtime
flatpak list --app
```

Refresh the runtime metadata:

```bash
flatpak update --appstream
```

Remove an unused runtime:

```bash
flatpak uninstall --unused
```

## 11. Troubleshooting Flatpak issues

### Remote cannot be added or updated

```bash
flatpak remote-list
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak update
```

### App fails to launch

```bash
flatpak run --verbose org.mozilla.firefox
flatpak list --app
flatpak info org.mozilla.firefox
```

### Runtime or dependency is missing

```bash
flatpak update
flatpak repair
flatpak uninstall --unused
```

## 12. Best practices

- Prefer official repositories for system packages and use DNF for RPM-based software.
- Use Flatpak only when the app is better delivered as a sandboxed desktop application.
- Check available updates and repository health before installing or removing software.
- Review package history and avoid destructive changes without verifying the target package.
- Keep Flatpak remotes limited to trusted sources.

This SOP is intended to provide a repeatable workflow for installing, updating, and troubleshooting software on Rocky Linux 10.1. Always validate the package source, repository health, and application dependency chain before making changes to production systems.
