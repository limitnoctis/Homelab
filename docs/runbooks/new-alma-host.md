# Runbook: Set up a new AlmaLinux container host

Goal: An AlmaLinux 10 VM that gets packages and images exclusively through the egress proxy and provides rootless Podman.

## Prerequisites

- VM with a static IP in the target network
- OPNsense rule from the VM to the egress proxy
- On the proxy's allowlist: `repo.almalinux.org`, `dl.fedoraproject.org` and the container registries the services need

## 1. Route dnf through the proxy

Add a `proxy=` line with the egress proxy's address to `/etc/dnf/dnf.conf`.

## 2. Replace mirror lists with fixed sources

AlmaLinux and EPEL use mirror lists by default, which point to changing servers that are not on the allowlist.

In every file under `/etc/yum.repos.d/` that starts with `almalinux`:
- comment out the `mirrorlist=` line
- uncomment the `baseurl=` line below it

The debug and source repositories are disabled anyway and can stay as they are.

## 3. Add EPEL

```bash
sudo dnf install -y epel-release
```

In `/etc/yum.repos.d/epel.repo`, section `[epel]`:
- comment out the `metalink=` line
- uncomment the `baseurl=` line and replace the placeholder host with `dl.fedoraproject.org`

## 4. Install Podman

```bash
sudo dnf install -y podman podman-compose systemd-container
```

`podman-compose` comes from EPEL. `dnf` aborts the whole transaction if a single package can't be found, so EPEL has to be set up first. `systemd-container` provides `machinectl`, which is needed to work as a service user.

SELinux stays in enforcing mode.

Next: [Move a service to its own user](migrate-service-to-own-user.md) or [Expose a service](expose-service.md).
