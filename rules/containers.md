# Services in Containers

How heinzel inspects a service that runs in a
container: Docker, Podman (rootful and rootless),
containerd with `nerdctl`.

Kubernetes is out of scope. Containers in the
containerd namespace `k8s.io`, or a host running
`kubelet`/k3s, are managed by the cluster: report
them, change nothing by hand. System containers
(LXC, Incus, LXD, Proxmox):
`rules/system-containers.md`.

## Privileges

- **Docker, rootful Podman, nerdctl:** root. The
  `docker` group is root-equivalent
  (`rules/privilege-escalation.md`).
- **Rootless Podman or rootless Docker:** each
  account sees only its own containers. Root sees
  none of them with `podman ps`; run the command as
  the owner (next section).

## Detect the Runtime

Run these probes in one SSH call
(`rules/ssh-connections.md`).

```bash
for rt in docker podman nerdctl; do
  command -v "$rt" >/dev/null 2>&1 \
    && "$rt" --version 2>&1
done
systemctl is-active docker podman.socket \
  containerd 2>/dev/null
ps -C conmon,rootlesskit -o user= | sort -u
ls -d /home/*/.local/share/containers \
  /home/*/.local/share/docker 2>/dev/null
```

`docker --version` answering `podman version …` is
the `podman-docker` shim: treat the host as Podman.

The `ps` line names accounts with running
containers (`conmon` for Podman, `rootlesskit` for
rootless Docker); `root` there is rootful Podman.
The `ls` line also finds accounts whose containers
have all stopped. Query one owner as root:

```bash
uid=$(id -u alice)
sudo -u alice env XDG_RUNTIME_DIR=/run/user/$uid \
  podman ps -a
```

Rootless Docker answers on
`unix:///run/user/<uid>/docker.sock`: pass it as
`DOCKER_HOST` the same way.

containerd keeps containers in namespaces; `nerdctl`
shows only `default` unless told otherwise:

```bash
nerdctl namespace ls
nerdctl --namespace <ns> ps -a
```

Record the runtime in server memory as
`rules/service-class-check.md` shows, and keep its
list of rootless owners current.

## List and Inspect

`docker`, `podman` and `nerdctl` take the same
commands below; the examples use `docker`.
`docker inspect` takes several names: pass
`$(docker ps -aq)` for all containers at once.

```bash
docker ps -a --format \
  '{{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker inspect --format \
  '{{.Name}} {{.HostConfig.RestartPolicy}} {{.HostConfig.LogConfig}}' \
  $(docker ps -aq)
docker inspect --format \
  '{{range .Mounts}}{{println .Type .Source .Destination .RW}}{{end}}' \
  <name>
```

Config files and data behind a `bind` mount live on
the host at `Source`: read them there with ordinary
tools. Named volumes sit under the runtime's data
directory (`/var/lib/docker/volumes/<name>/_data`,
`/var/lib/containers/storage/volumes/`, or
`~/.local/share/…` when rootless).

Environment variables: names only
(`rules/secrets.md`).

```bash
docker inspect --format \
  '{{range .Config.Env}}{{println .}}{{end}}' <name> \
  | cut -d= -f1
```

## Find What Defines the Container

Before any change, find the file that recreates the
container, or the change is lost at the next
recreate.

- **Compose:** `docker compose ls -a` lists every
  project with its config files. For one container:

  ```bash
  docker inspect --format \
    '{{index .Config.Labels "com.docker.compose.project.config_files"}}' \
    <name>
  ```
- **Quadlet (Podman):** the label
  `PODMAN_SYSTEMD_UNIT` names the unit;
  `systemctl show -p SourcePath <unit>` names the
  `.container` file. Or list them all:
  `podman quadlet list`. Both as the owner when
  rootless (`systemctl --user`): root's
  `systemctl show` answers an empty `SourcePath`.
  Search paths: `/etc/containers/systemd/` and
  `~/.config/containers/systemd/`, see
  `podman-systemd.unit(5)`.
- **Own systemd unit** calling `docker run` or
  `podman run`:
  `grep -rlE '(docker|podman) run' /etc/systemd/system`.
- **None of these:** a hand-started container. Say
  so; its options exist only in `docker inspect`.

## Logs

```bash
docker logs --since 1h --tail 200 <name> 2>&1
```

Works for the `json-file`, `local` and `journald`
drivers, and for Podman's default (`journald`, or
`k8s-file` without a usable journal). Always pass
`--since` or `--tail`: a container's log can be
gigabytes. With the `journald` driver the entries
outlive the container:

```bash
journalctl -b CONTAINER_NAME=<name> -n 200 --no-pager
```

## Commands Inside a Container

`docker exec` / `podman exec` / `nerdctl exec` only
for what exists only inside, and read-only: `cat`,
`ls`, a version flag, a config test (`nginx -t`), a
status query. Prefer mounts, logs and `inspect`.

- **No state changes:** no shell (`sh`, `bash`,
  `-it`), no package manager, no writes, no restart
  of the process inside. A missing tool is reported,
  never installed.
- **It runs with the container's privileges:** as
  the image's user, often root inside, with the
  container's capabilities and its mounts. A write
  through a `rw` bind mount changes the host. Never
  add `--privileged` or `--user 0`.
- **No secrets:** no `env`, no `cat` of
  `/run/secrets/*` or key files
  (`rules/secrets.md`).

## Changes

Every change to a container follows
`rules/service-reload.md`:

- `restart`, `stop`/`start`, recreate
  (`docker compose up -d`, a quadlet unit restart)
  and an image pull that is followed by a recreate
  are **restarts**: ask.
- A reload signal (`docker kill -s HUP`) counts
  as a **reload** only when the service's config test
  inside passed.
- `rm`, `prune`, `volume rm`, `down -v` are
  destructive: name every container and volume that
  goes.
- A compose file, quadlet file or unit is a config
  file: back it up before editing
  (`rules/backups.md`).
