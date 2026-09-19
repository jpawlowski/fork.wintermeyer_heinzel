# Home Assistant OS

Rules for Home Assistant OS (HAOS). HAOS is Linux,
but it is an appliance: no package manager, a
read-only root filesystem, and everything is managed
through the Supervisor and its `ha` CLI. No base
family file applies. Where CLAUDE.md or a baseline
expects something a Linux server has (firewall,
automatic updates, `sudo`), this file says what to
check instead.

Sources unless noted: the developer docs,
<https://developers.home-assistant.io/docs/operating-system>,
the user docs, <https://www.home-assistant.io/>, and
the `home-assistant/cli`, `home-assistant/addons`
and `home-assistant/operating-system` repositories.

## Where You Land

`ssh root@<host>` almost never reaches the HAOS
host. There are three ways in:

- **Terminal & SSH add-on** (official, slug `ssh`):
  a container (Alpine) with the `ha` CLI, bash, and
  root inside the container only. The add-on docs:
  "Regardless of how you connect … you end up in
  this app's container." The SSH port is whatever
  the user set in the add-on options.
- **Advanced SSH & Web Terminal** (community): also
  a container, zsh, with the host network and the
  host journal (read-only). `docker` works only when
  the user has turned protection mode off.
- **Host SSH on port 22222**: dropbear on the HAOS
  host itself, root, keys only. It exists for
  developers; the docs say it is "not for end
  users". Never set it up on your own.

Record which of the three it is in server memory
(`Platform: Home Assistant OS <version>, via
<add-on name | host port 22222>`).

## Detection

- Markers: `rules/os-detection.md` step 5.
- Versions: `ha os info`, `ha core info`,
  `ha supervisor info`. `ha info` gives an
  overview.
- Health: `ha resolution info` lists issues,
  suggestions, and whether the system is
  unsupported or unhealthy. Read it on every
  connection, in the same call as the activity
  check.

## What Does Not Apply

- **Packages.** The host has no package manager and
  its system partitions are read-only. `apk add` in
  an add-on is lost when the container is recreated.
  Do not install tools there. If one is needed for
  good, the user adds it to the add-on's package
  option. Language runtimes (`rules/mise.md`) do not
  belong here either.
- **Privileges.** The add-on shell is already root
  of its container, and it cannot reach the host
  beyond what the Supervisor allows. There is no
  `sudo` to probe and no root SSH fallback
  (`rules/privilege-escalation.md`).
- **Firewall.** HAOS has no user-managed firewall.
  A missing one is not a finding. Exposure is
  decided by the router and by which add-ons
  publish ports. Core listens on 8123 by default,
  the Supervisor's observer on 4357.
- **Automatic security updates.** There is no
  `unattended-upgrades`. The Supervisor updates
  itself; Core, OS and add-ons wait for the user.
  Pending updates are the finding, not a missing
  package.
- **sshd.** The add-on generates its SSH config from
  its options. Changes go through the add-on
  configuration in the UI, not through files.
- **Containers.** Do not manage Home Assistant's
  containers with `docker`, even where the Advanced
  add-on allows it. Use `ha`.

## Configuration

- The configuration lives in `/homeassistant`
  (`/config` links to it in the official add-on).
  Also mounted: `/share`, `/ssl`, `/media`,
  `/backup`.
- `rules/backups.md` applies, but keep the copies
  in `/homeassistant/.heinzel-backups/`: the add-on
  container's own filesystem is replaced when the
  add-on updates, and `/share` is shared with any
  add-on that maps it, so copies of `secrets.yaml`
  do not belong there.
- Never print `secrets.yaml` (`rules/secrets.md`).
- **Config test:** `ha core check` validates the
  configuration on disk. It must pass before any
  restart.
- `ha core restart` restarts Home Assistant and
  interrupts every automation for the duration.
  `rules/service-reload.md` decides when to ask.
  `ha core rebuild` recreates the container; ask
  first.

## Updates

- List pending updates: `ha available-updates`.
  `ha refresh-updates` reloads the list.
- Apply, one at a time and only after asking:
  - `ha core update --backup`
  - `ha apps update <slug> --backup`
  - `ha os update` **reboots the host.** HAOS
    writes the other A/B slot and falls back to
    the old one after failed boots; manual
    rollback is `ha os boot-slot other`.
  - `ha supervisor update` is rarely needed; the
    Supervisor updates itself unless the user
    turned that off.
- Add-ons were renamed "apps" in 2026.2. `ha apps`
  is the current command; `ha addons` still works
  as an alias.

## Backups

- `ha backups list`; `ha backups new --name <name>`
  creates a full backup in `/backup`.
- Never pass `--password`: it puts the password on
  the command line (`rules/secrets.md`). If the
  backup must be encrypted, the user creates it in
  the UI.
- Take one before any update that is not already
  run with `--backup`, and before larger
  configuration changes.

## Logs

- `ha core logs`, `ha supervisor logs`,
  `ha apps logs <slug>`, `ha host logs` (the host
  journal, persistent). There is no `ha os logs`.
  Never add a follow flag over non-interactive SSH.
- `logger -t heinzel` inside an add-on probably
  does not reach the host journal. After the first
  entry, read it back with
  `ha host logs -t heinzel`. If the line is
  missing, log to the local changelog only
  (`rules/changelog.md`) and note `Journal: none`
  in server memory, so the activity check does not
  read an empty journal as silence.

## Housekeeping and Audits

- The Linux baseline does not apply (see What Does
  Not Apply). Check `ha available-updates`,
  `ha resolution info` and `ha backups list`
  instead. For a security audit, also report the
  SSH add-on's options and which add-ons publish
  ports.
- Fleet audit: not covered yet. Skip the host with
  a "platform not yet supported" note.

## Network

- `ha network info`; changes go through
  `ha network update <interface> …`. A wrong
  address, gateway or a disabled interface cuts
  every way in, including yours. Ask first, and make
  sure the user has console access.

## Never

- `ha host shutdown` — powers the device off
  (CLAUDE.md taboo).
- `ha os datadisk move` or `wipe` — moves or erases
  the data partition.
- `ha backups restore` without an explicit request.
- Editing files under `/homeassistant/.storage/` by
  hand: the UI owns them.

## Supervised and Core Installs

Home Assistant Supervised and Core on a normal
distribution have been unsupported since 2025.12
(<https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/>).
On such a host the distro's family file applies to
the host, and this file to the `ha` CLI. Report the
unsupported install once as INFO; migrating to HAOS
or the Container install is the user's decision.
