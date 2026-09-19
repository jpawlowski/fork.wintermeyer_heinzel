# OPNsense

Overlay for OPNsense firewalls. OPNsense is built
on FreeBSD, so `rules/freebsd.md` supplies the
vocabulary (`ifconfig`, `pfctl`, ZFS), but most of
its instructions for changing the system are
**wrong here**: OPNsense generates the system
configuration from one XML file and overwrites
manual edits. This file wins wherever the two
disagree.

The host is usually the network's only way out. A
mistake here cuts off everyone behind it, not just
your SSH session.

Source for everything below unless noted: the
OPNsense documentation, <https://docs.opnsense.org/>,
and the `opnsense/core` source where the docs are
silent.

## Detection

- Marker: `which opnsense-version`.
- `opnsense-version` prints e.g.
  `OPNsense 26.7.4 (amd64)`; `opnsense-version -v`
  only the version, `-V` the series (`26.7`).
- Two major releases a year (`YY.1`, `YY.7`), minor
  updates in between.
- Record in server memory:
  `Platform: OPNsense <version>`.

## Access and Shell

- **root runs remote commands in csh.** root's
  login shell is `opnsense-shell`, which hands a
  command passed over SSH to `/bin/csh -c`
  (`rules/os-detection.md` step 1 records it as
  `Shell: csh`). An admin user with its own login
  shell gets that shell instead.
- An interactive root login shows the console menu.
- SSH is **off by default** on an installed system;
  the user enables it under System > Settings >
  Administration. Root login is a separate option
  there, and only members of `wheel` may log in at
  all.
- Non-root admins get a shell only when one is set
  for them under System > Access > Users. Keys are
  managed there too, not in `authorized_keys` by
  hand.
- `sudo` is installed but grants nothing until the
  "Sudo" option under System > Settings >
  Administration allows it for `wheel` ("Ask
  password" or "No password"). With "Ask password",
  `sudo -n` fails. Probe as usual
  (`rules/privilege-escalation.md`).
- The SSH config is generated into
  `/usr/local/etc/ssh/sshd_config`, host keys live
  in `/conf/sshd/`. `sshd_config` includes
  `/usr/local/etc/ssh/sshd_config.d/*.conf`. The
  CLAUDE.md SSH taboo covers all of these and the
  whole `/conf/sshd/` directory: read only.

## Configuration Model

- **Everything lives in `/conf/config.xml`.**
  `configd` renders the system configuration from
  it through templates and overwrites, among
  others: `sshd_config`, `/boot/loader.conf`, root's
  crontab, `/etc/rc.conf.d/*`, the pf ruleset
  (`/tmp/rules.debug`) and sudoers.
- **Change settings through the web UI** (or the
  API, if the user has set up a key). Give the user
  the exact menu path and values. Do not use
  `sysrc` or edit `rc.conf`.
- Every save keeps a copy in
  `/conf/backup/config-<epoch>.<fraction>.xml`.
  `rules/backups.md` still applies, `config.xml`
  included. If `/var` is a RAM disk (see Logs),
  keep the copies in `/root/heinzel-backups/`
  instead.
- **Apply from the shell with `configctl`**, the
  front end to `configd`:
  - `configctl configd actions` lists all actions.
  - `configctl filter reload` regenerates and
    loads the firewall rules.
  - `configctl service reload all` reapplies
    everything (console menu option 11).
- Documented places for persistent custom
  additions: `/usr/local/etc/rc.syshook.d/<event>/`
  (boot and event hooks), `/usr/local/etc/cron.d/`
  (own cron jobs), System > Settings > Tunables
  (loader and sysctl values). Template overrides in
  `+TARGETS.D` may break on upgrades; avoid them.
- `configctl system halt`,
  `system reset_factory_defaults` and
  `system flush config_history` exist. Never.
- configd action names with a dot are called with
  a space: `configctl filter rule stats`, not
  `filter rule.stats`.

## Updates

- **Never** use `freebsd-update`: base and kernel
  come as signed sets through `opnsense-update`.
  Never add the FreeBSD or any other repository
  under `/usr/local/etc/pkg/repos/`; OPNsense calls
  that "not supported".
- Check without installing:
  ```
  configctl firmware probe
  cat /tmp/pkg_upgrade.json
  ```
  `needs_reboot` in the JSON says whether the
  update would reboot; `upgrade_needs_reboot` only
  says that a major upgrade is available.
- Apply a minor update: `configctl firmware update`.
  **It reboots the firewall on its own** when base
  or kernel changed, and after any package change
  when the firmware "reboot" setting is on. Always
  ask first, and say whether the probe expects a
  reboot. It returns at once and runs in the
  background; follow it with
  `configctl firmware status`.
- On ZFS (24.7.3 and later), take a snapshot before
  updating: `configctl zfs snapshot list`, then
  `configctl zfs snapshot create <name>`. It is the
  rollback path (boot menu option 8). On UFS there
  is none; say so before asking.
- **Major upgrades** (e.g. 26.1 → 26.7) run offline
  and take the firewall down for the duration. The
  docs want console access. Hand them to the user
  (console menu option 12); never run
  `configctl firmware upgrade` yourself. Read the
  release notes first; they list plugins and repos
  that block the upgrade.
- Automatic updates exist as an opt-in cron job
  ("Automatic firmware update", minor updates only).
  Its absence is not a finding; pending updates are.
- Keep the firmware release type on
  "Production".
- Update log: `opnsense-update -g`; last major
  upgrade: `opnsense-update -G`.

## Plugins

- Plugins are `os-*` packages. Install with
  `configctl firmware install os-<name>`, remove
  with `configctl firmware remove os-<name>`. Plain
  `pkg install` skips the registration in the
  config, so the plugin is lost on a config
  restore.
- Community plugins are "Tier 3": supported by the
  community, not the core team. Third-party repos
  are not OPNsense's at all. Ask before installing
  either.
- CLAUDE.md "Service Class Conflict Check" still
  applies: OPNsense already brings a web server, DNS
  resolver and DHCP server.

## Firewall

- **Expected:** pf, managed by OPNsense. A default
  deny rule blocks everything no other rule
  matches; WAN also blocks private and bogon
  networks by default.
- Read-only: `pfctl -sr` (rules), `pfctl -s nat`,
  `pfctl -si`, `configctl filter rule stats`. The
  generated ruleset is in `/tmp/rules.debug`.
- **Rules are changed in the web UI or the API**,
  never with `pfctl -f` on a file you wrote: the
  next `configctl filter reload` replaces it. The
  `freebsd.md` pf workflow (edit `pf.conf`, load,
  `at` safety net) does not apply.
- Since 26.7 the rules API applies changes at once;
  the automatic rollback of older versions is gone.
  Do not count on a timed revert.
- **Anti-lockout rule:** keeps the web UI and SSH
  reachable on LAN (or the first interface that
  exists), ahead of user rules. It can be disabled
  under Firewall > Settings > Advanced (26.7; the
  menu moves in later versions). Check that it is
  on before any rule change. Do not turn it off
  unless the user explicitly asks.
- `pfctl -d` switches off the firewall **and NAT**
  until the next reload: everyone behind it loses
  internet access. It is not a safety net for
  heinzel.
- `configctl filter flush states` and the
  `filter kill …` actions drop live connections,
  including your own SSH session. Ask first.

## Services

- `pluginctl -s` lists services;
  `pluginctl -s <name> restart|start|stop|status`
  controls one. `configctl service list` (JSON)
  and `configctl service restart <name>` do the
  same through `configd`.
- Some services have their own namespace, e.g.
  `configctl webgui restart`,
  `configctl openssh restart`.
- Avoid `service <name> restart`: the generated
  config is written by `configd`, not by the rc.d
  script. The documented exception is
  `service configd restart`.
- `rules/service-reload.md` still decides when to
  ask. Restarting `openssh` over SSH needs the same
  care as on any host.

## Logs

- One file per day:
  `/var/log/<area>/<area>_<YYYYMMDD>.log`, written
  by syslog-ng.
- `opnsense-log -l` lists the areas;
  `opnsense-log <area>` prints the current log,
  `opnsense-log -n <area>` its path
  (`/var/log/<area>/latest.log`). Never use
  `-f` over non-interactive SSH: it does not exit.
- `/var` may be a RAM disk (an option in System >
  Settings): logs from before a reboot may be
  gone. Not a finding on its own
  (`rules/verify-before-reporting.md`).
- No `journalctl`. `logger -t heinzel` lands in
  the `system` area (`opnsense-log system`), but
  only at level notice or higher: keep `logger`'s
  default priority, never `-p user.info`. It also
  needs local logging to be on.

## Housekeeping and Audits

- The FreeBSD checks for `freebsd-update`,
  `pf.conf` and enabled rc services do not apply.
  Check `configctl firmware probe` (pending updates
  are the finding), the anti-lockout rule,
  `pfctl -sr` and `pluginctl -s` instead.
- Report settings OPNsense generates as web UI
  changes, not file edits.
- Fleet audit: not covered yet. Skip the host with
  a "platform not yet supported" note.
