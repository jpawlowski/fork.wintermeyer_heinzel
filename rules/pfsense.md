# pfSense

Overlay for pfSense CE and pfSense Plus firewalls.
pfSense is built on FreeBSD, so `rules/freebsd.md`
supplies the vocabulary (`ifconfig`, `pfctl`,
`/usr/local/etc`), but most of its instructions for
changing the system are **wrong here**: pfSense
generates the system configuration from one XML
file and overwrites manual edits. This file wins
wherever the two disagree.

The host is usually the network's only way out. A
mistake here cuts off everyone behind it, not just
your SSH session.

Source for everything below unless noted: the
Netgate documentation,
<https://docs.netgate.com/pfsense/en/latest/>.

## Detection

- Marker: `which pfSense-upgrade` (CE and Plus);
  `/etc/platform` contains `pfSense`.
- Version: `cat /etc/version` (e.g.
  `2.9.0-RELEASE`), patch level in
  `/etc/version.patch` (`0` means unpatched).
- **CE or Plus:** CE numbers its releases `2.x.y`,
  Plus uses `YY.MM` (e.g. `26.07`). The two differ
  in upgrade path and features, above all Boot
  Environments (Plus only).
- FreeBSD base: `uname -mrs`.
- Record in server memory:
  `Platform: pfSense CE <version>` or
  `pfSense Plus <version>`.

## Access and Shell

- SSH is **off by default**; the user enables it in
  the web UI or on the console. With default rules
  it is reachable from LAN only.
- `root` and `admin` share the same keys. Keys are
  managed in the web UI (User Manager); pfSense
  rewrites `/root/.ssh/authorized_keys` from the
  config. **Never manage keys from the shell.**
- pfSense generates `/etc/ssh/sshd_config` and
  appends `/etc/sshd_extra` to it. The CLAUDE.md
  SSH taboo covers both files and the host keys in
  `/etc/ssh`: read only.
- An interactive root login shows the console
  menu. A command passed over SSH skips the menu
  and runs under `/bin/sh`.
- A non-root user with shell access gets `tcsh`
  (`rules/os-detection.md` step 1 records it as
  `Shell: tcsh`).
- No `sudo` in the base system. It comes from the
  Sudo package, which asks for a password unless
  the entry is set to "No Password"; `sudo -n` then
  fails. Probe as usual
  (`rules/privilege-escalation.md`), and use root
  SSH as the fallback.
- `sshguard` blocks an address after repeated failed
  SSH or web UI logins. `rules/ssh-connections.md`
  applies; if it blocked you, the user clears it
  from another address (`pfctl -T flush -t sshguard`).

## Configuration Model

- **Everything lives in `/conf/config.xml`**
  (`/conf` links to `/cf/conf`). pfSense generates
  `/etc/ssh/sshd_config`, the pf ruleset
  (`/tmp/rules.debug`) and service configs from it.
  Edits to generated files are overwritten.
- `/etc/rc.conf` is not used for pfSense services,
  and editing it is unsupported. Do not edit it,
  and do not use `sysrc`.
- **Change settings through the web UI.** Give the
  user the exact menu path and values. From the
  shell, only use the documented tools below.
- pfSense keeps the last 30 configs in
  `/cf/conf/backup/`. `rules/backups.md` still
  applies, `config.xml` included. If `/var` is a
  RAM disk (see Logs), keep the copies in
  `/root/heinzel-backups/` instead.
- Editing `config.xml` by hand is the last resort:
  `viconfig` (clears the config cache on save),
  then reapply the affected area in the web UI or
  reboot. The documented safe route is to download
  a backup, edit it, and restore it in the web UI
  (the firewall reboots). Ask before either.
- **PHP shell playback scripts** (`pfSsh.php playback
  <script>`) run the same code as the web UI. Run
  them over SSH or on the console only, one script
  per `pfSsh.php` invocation (several invocations
  may share one SSH call). Useful ones: `svc`,
  `gatewaystatus`, `listpkg`, `pfanchordrill`,
  `pftabledrill`.
  `disabledhcpd`, `removepkgconfig` and
  `removeshaper` **delete config sections**
  (`removepkgconfig` also deletes
  `/usr/local/etc/rc.d/*`); never without an
  explicit request. Never run several scripts in
  one PHP shell session.
- Persistent custom commands: prefer the
  `shellcmd`/`earlyshellcmd` entries in the config
  (Shellcmd package), which are in config backups.
  Netgate also documents `/usr/local/etc/rc.d/*.sh`
  scripts; they run at boot and on some network
  events.
- Back from a bad config change without the web
  UI: console menu option 15, "Restore recent
  configuration".

## Updates

- **Never** use `freebsd-update`, and never add the
  FreeBSD or any non-Netgate package repository.
  Netgate: FreeBSD packages "will have unintended
  side effects", third-party repos can leave the
  system "unbootable".
- Never upgrade packages on their own before the
  system; Netgate says "Do not upgrade packages
  before upgrading pfSense software".
- Check for an update:
  ```
  pfSense-upgrade -d -c
  ```
  Exit code 2 means an update is available, 0 means
  current, 1 that another instance is running. Read
  the output too: it also exits 0 after "Aborted
  due to block_external_services flag". `-c` checks
  the current release branch only; `-C` looks for a
  major upgrade. It refreshes the repo setup first,
  so it is not strictly read-only, but it installs
  nothing. The web UI caches its last check in
  `/var/run/pfSense_version`.
- Apply: `pfSense-upgrade` (console menu option 13).
  **It reboots the firewall when done.** Always ask.
  Over SSH run it inside `screen`, which is not in
  the base system (`pkg install screen`, ask
  first). Log: `/conf/upgrade_log.latest.txt`.
- **Plus on ZFS (24.03 and later)** upgrades inside
  a new Boot Environment and rolls back on its own
  if the new one fails to boot. **CE has no Boot
  Environments**: recovering from a failed upgrade
  needs console access. Say which case applies
  before asking.
- There is no automatic update. Pending updates are
  a finding for housekeeping, not a missing
  package.
- CE stays CE and Plus stays Plus; switching
  editions is a separate migration.

## Packages

- pfSense packages are named `pfSense-pkg-<name>`.
  Install and remove them through the web UI
  (System > Packages) or its CLI equivalent:
  `pfSsh.php playback installpkg <name>`,
  `uninstallpkg <name>`, `listpkg`.
- CLAUDE.md "Service Class Conflict Check" still
  applies: pfSense already brings a web server, DNS
  resolver and DHCP server.

## Firewall

- **Expected:** pf, managed by pfSense. The WAN
  default is to block all inbound traffic; LAN has
  default allow rules.
- Read-only: `pfctl -sr` (rules), `pfctl -sn`
  (NAT), `pfctl -ss` (states),
  `pfSsh.php playback pfanchordrill`. The generated
  ruleset is in `/tmp/rules.debug`.
- **Rules are changed in the web UI**, never with
  `pfctl -f` on a file you wrote: the next filter
  reload (almost every save) replaces it. The
  `freebsd.md` pf workflow (edit `pf.conf`, load,
  `at` safety net) does not apply.
- The only documented CLI rule tool is `easyrule`:
  ```
  easyrule showblock wan
  ```
  `easyrule pass <if> <proto> <src> <dst> [port]` and
  `easyrule block <if> <src>` add real rules to the
  config, `easyrule unblock <if> <src>` removes a
  block. Ask first.
- **Anti-lockout rule:** keeps the web UI and SSH
  reachable on LAN, ahead of user rules. It can be
  disabled under System > Advanced > Admin Access.
  Check that it is on before any rule change. Do
  not turn it off unless the user explicitly asks.
- `pfctl -d` switches off the firewall **and NAT**
  until the next reload: everyone behind it loses
  internet access. It is an emergency tool for the
  user on the console, not a safety net for heinzel.
- `pfSsh.php playback enableallowallwan` opens WAN
  completely. Never.

## Services

- Service control:
  `pfSsh.php playback svc <action> <service>`
  with `start`, `stop`, `restart` or `status`, and
  the name as shown under Status > Services, e.g.
  `pfSsh.php playback svc restart unbound`.
- Netgate documents only `svc` for this. Do not
  fall back to `service <name> restart`.
- Web UI stuck: `/etc/rc.restart_webgui` (menu
  option 11), `/etc/rc.php-fpm_restart` (option 16).
- `rules/service-reload.md` still decides when to
  ask.

## Logs

- Plain text in `/var/log/*.log` since CE 2.5.0 /
  Plus 21.02; older versions use binary clog files
  (`clog /var/log/filter.log`).
- Rotated logs are bzip2-compressed by default
  (`bzcat`, `bzgrep`); new installs with `/var/log`
  on compressed ZFS default to no compression.
  Check the log settings before assuming either.
- Readable firewall log:
  `tail -n 50 /var/log/filter.log | filterparser.php`.
- `/var` may be a RAM disk: logs from before an
  unclean restart may be gone. Not a finding on its
  own (`rules/verify-before-reporting.md`).
- No `journalctl`. `logger -t heinzel` lands in
  `/var/log/system.log`; read the heinzel changelog
  back from there.

## Housekeeping and Audits

- The FreeBSD checks for `freebsd-update`,
  `pf.conf` and enabled rc services do not apply.
  Check `pfSense-upgrade -d -c` (never without
  `-c`: it reboots), the anti-lockout rule,
  `pfctl -sr` and Status > Services
  (`pfSsh.php playback svc status <service>`)
  instead.
- Report settings pfSense generates as web UI
  changes, not file edits.
- Fleet audit: not covered yet. Skip the host with
  a "platform not yet supported" note.
