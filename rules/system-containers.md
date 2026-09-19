# System Containers (LXC, Incus, LXD, Proxmox)

A system container shares the host's kernel but
boots its own init, package manager and journal. For
heinzel it is a server, not a service: application
containers (Docker, Podman) are `rules/containers.md`.

## Reaching It

**SSH first, always.** Via-host mode
(`rules/first-connection.md` → Via-host mode) is
the fallback, for two cases only:

- **SSH gives no answer:** only as
  `rules/ssh-unreachable.md` → A guest on a known
  host allows, for this session. Server memory
  keeps its SSH access.
- **No sshd in the guest:** ask the user once:
  install one (then SSH), or record via-host mode
  in its memory (`rules/server-memory.md`).

Through the host's manager, as root inside:

- **Incus:** `incus list --all-projects`,
  `incus exec <ct> -- <cmd>`
- **LXD:** `lxc list --all-projects`,
  `lxc exec <ct> -- <cmd>`
- **LXC:** `lxc-ls -f`, `lxc-attach -n <ct> -- <cmd>`
- **Proxmox:** `pct list`, `pct exec <vmid> -- <cmd>`
- **Proxmox VM:** `qm guest exec <vmid> -- <cmd>`
  after `qm guest cmd <vmid> ping` in the same call

LXD's client is named `lxc`; the classic LXC tools
are `lxc-*`. A libvirt VM, or one without QEMU
guest agent, has only its console: interactive,
the user's tool, not heinzel's.

## Privileges

Managing containers needs root on the host
(`rules/privilege-escalation.md`). A privileged
container's root is host UID 0: record
`- Container: privileged` in its memory. Find them
with `grep -L '^unprivileged: 1' /etc/pve/lxc/*.conf`
(Proxmox) or `security.privileged: "true"` in
`incus config show <ct>`.

## Changes

Creating and changing guests is ordinary work on
the host, and every change is asked first:

- Restarting a container or VM is a reboot
  (`rules/service-reload.md`).
- Before a config change (`incus config set`,
  `pct set`): a snapshot (below), else a copy of
  `/etc/pve/lxc/<vmid>.conf` or of
  `incus config show <ct>` (`rules/backups.md`).
- **Stopping or deleting one** powers off or
  destroys a server: only on the user's explicit
  request. First show, from the live host and in
  one call, what it hits: ID, name, host, state,
  disks, and the newest backup (a snapshot goes
  with the guest).
- Deleting a snapshot, or rolling back to one
  (which discards everything since): name what
  goes.

## Snapshots

With access to the host, a snapshot is the better
safety net before a risky change to a guest, to its
config or inside it (an upgrade, a larger config
rework): it covers the whole guest and rolls back
in one command.

List what exists first, and take one only when
nothing fits:

- **Proxmox:** `pct listsnapshot <vmid>`,
  `pct snapshot <vmid> <name>` (`qm` for a VM)
- **Incus / LXD:** `incus info <ct>` lists them,
  `incus config show <ct>` shows `snapshots.schedule`
  and `snapshots.expiry`;
  `incus snapshot create <ct> <name>`
  (LXD: `lxc info`, `lxc snapshot <ct> <name>`)
- **libvirt:** `virsh snapshot-list <dom>`,
  `virsh snapshot-create-as <dom> <name>`
- **ZFS host**, for automatic ones (sanoid,
  zfs-auto-snapshot), newest five of the guest's
  dataset:

  ```bash
  zfs list -t snapshot -H -o name,creation \
    -s creation -d 1 <dataset> | tail -n 5
  ```

A fresh automatic snapshot (hourly, say) is enough,
and whatever makes it also removes it: name it to
the user and ask whether it will do. A new one is
named for heinzel and the change
(`heinzel-pre-upgrade-20260919`). The storage must
support it, else the command fails and the file
backup applies. A snapshot lives on the guest's own
storage: it is no backup, and it goes with the
guest.

A snapshot heinzel took is heinzel's to remove, as
it grows while it exists. Offer to delete it once
the change has proved itself; if the user wants to
wait, write `- [ ] delete snapshot <name> on <host>`
into the guest's `todo.md` (CLAUDE.md → Session
To-Do List).

Docker in a system container (common with Proxmox
`nesting=1`): `rules/containers.md`.
