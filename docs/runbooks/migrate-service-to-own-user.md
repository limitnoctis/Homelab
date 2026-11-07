# Runbook: Move a service to its own user

Goal: A rootless Podman service moves from its current user to a dedicated user without reassigning permissions.

**Before you start:** take a snapshot of the VM.

## Why it's not just a copy

Every user has its own range of subuids on the host. A file that belongs to postgres (UID 999) inside a container belongs to a UID from the old user's range on the host. Copied over with `cp` or `chown`, the UIDs would no longer match inside the new user's containers.

The solution is to pack and unpack everything from the container's point of view. Then UID 999 stays UID 999 inside the container, and the host UID follows automatically.

## 1. Create the user

```bash
sudo useradd -m -s /bin/bash <service>
sudo loginctl enable-linger <service>
```

`useradd` assigns the user its own subuid and subgid range. Linger keeps the user's services running without a login.

Copy the old user's `~/.config/containers/containers.conf` into the same place in the new user's home. It holds the proxy settings for image pulls.

## 2. Stop the service as the old user

```bash
podman-compose down
```

Run it in the service's directory. The data must not change while it's being packed.

## 3. Pack the data

**Bind mounts** (directories next to the compose file that the container writes to):

```bash
podman unshare tar -C ~/<service> -cpf /var/tmp/<service>.tar .
```

`podman unshare` runs `tar` inside the user's UID range, so ownership is stored the way the container sees it.

**Named volumes** (managed by Podman):

```bash
podman volume export <project>_<volume> -o /var/tmp/<project>_<volume>.tar
```

Volumes that only hold a cache can be left behind.

**Podman secrets** belong to a single user and have to be recreated. Write each secret's value to a file that only the old user can read, and delete it once the secret is recreated in step 5.

## 4. Hand the archives over

Move the archives into the new user's home and give them to the new user with mode 600.

**The archives contain databases and keys.** Don't leave copies in `/var/tmp`.

## 5. Unpack as the new user

Switch to the new user. `sudo -u` is not enough for rootless Podman, it needs a full user session:

```bash
sudo machinectl shell <service>@
```

Unpack the bind mounts into the home directory:

```bash
podman unshare tar --no-overwrite-dir -C ~ -xpf ~/<service>.tar
```

`--no-overwrite-dir` keeps the home directory's own permissions (700).

Create and import each named volume:

```bash
podman volume create <project>_<volume>
podman volume import <project>_<volume> ~/<project>_<volume>.tar
```

The volume name must match what the compose file expects. `podman-compose` prefixes volume names with the directory name, which is now the new user's home.

Recreate each secret from its file with `podman secret create`, then delete the archives and secret files.

## 6. Start

```bash
systemctl --user enable --now podman-restart.service
podman-compose up -d
```

`podman-restart.service` starts the containers after a reboot, but only those with `restart: always` in the compose file.

## 7. Test and clean up

Test the service through the reverse proxy and reboot the VM once to make sure it comes back.

Only then remove the old user's volumes, networks, images and secrets. Its bind mount directories belong to subuids, so delete them with `podman unshare rm -rf` as the old user.
