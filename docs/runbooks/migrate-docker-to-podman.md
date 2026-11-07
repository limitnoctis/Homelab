# Runbook: Move a service from rootful Docker to rootless Podman

Goal: A service moves from a Docker VM to an AlmaLinux host under its own rootless user.

## 1. Stop the service cleanly

```bash
sudo docker compose down
```

Run it in the service's directory. Without `sudo` (or membership in the docker group) the command fails and the service keeps running. An archive of a running database is not reliable, so make sure the containers are really gone before continuing.

SQLite databases may leave `-wal` and `-shm` files behind. That's fine as long as they're copied together with the database.

## 2. Pack the data

With rootful Docker, files in volumes belong to real UIDs on the host. Pack each volume's `_data` directory under `/var/lib/docker/volumes/` with `tar --numeric-owner`, so the UIDs are stored as numbers and not as user names.

Pack the service's directory with the compose file and its config files separately.

Restrict all archives to mode 600. They contain databases and keys.

## 3. Transfer

If the old and new host are in networks that can't reach each other, copy the archives through your own machine:

```bash
scp -3 admin@<old-host>:/tmp/<archive>.tar admin@<new-host>:/var/tmp/
```

## 4. Import as the new user

Create the user and import the data as described in [Move a service to its own user](migrate-service-to-own-user.md), steps 1, 4 and 5. Files from the compose directory belonged to your own user on the old host and can be unpacked normally. Volumes are imported with `podman volume import`, which maps the stored UIDs into the new user's range.

## 5. Fix the volume's root directory

The import sets the ownership of the contents, but the volume's top-level directory still belongs to root inside the container. A service that creates new files there fails, for example SQLite with "attempt to write a readonly database".

Set the owner of the volume directory to the UID the container runs as:

```bash
podman unshare chown <container-uid>:<container-gid> <volume-directory>
```

## 6. Adjust the compose file

| Docker | Podman |
|---|---|
| Short image names like `grafana/grafana` | Full names including the registry, like `docker.io/grafana/grafana`. Podman doesn't guess. |
| `restart: unless-stopped` | `restart: always`, otherwise the container doesn't come back after a reboot |
| Ports published on all interfaces | Ports bound to the host's own IP |
| Bind mounts without options | Add `:Z` to every bind mount so SELinux allows the container to read it |
| External or named networks | The default network with the driver option `isolate: "true"` |

Config files that the container reads while running under its own UID need read permission for "others". The home directory with mode 700 still keeps other users out.

## 7. Start and switch over

Start the service as in step 6 of [Move a service to its own user](migrate-service-to-own-user.md), then [expose it](expose-service.md) and point everything that used the old address to the new one: proxy hosts, log shippers, dashboards.

Keep the old VM shut down for a few days before deleting it.
